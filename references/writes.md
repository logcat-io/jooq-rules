# 읽기 다중화와 쓰기 — N+1 · 벌크 · 트랜잭션 경계

> `SKILL.md` 의 근거와 예제. 연관 데이터를 함께 읽거나, 대량으로 쓰거나, 트랜잭션 경계를 정할 때 읽는다.

---

## 1. N+1 방지

목록 응답이 연관 데이터를 함께 반환하면(주문 20건 × 항목 + 상품 + 고객) 순진하게 짜서 수십 쿼리가 나간다.

### 1.1 batch fetch — 기본값

```kotlin
fun findItemsByOrderIds(orderIds: List<UUID>): Map<UUID, List<OrderItem>> =
    dsl.select(ORDER_ITEMS.ORDER_ID, ORDER_ITEMS.ID, PRODUCTS.NAME, ORDER_ITEMS.QUANTITY)
        .from(ORDER_ITEMS)
        .join(PRODUCTS).on(PRODUCTS.ID.eq(ORDER_ITEMS.PRODUCT_ID))
        .where(ORDER_ITEMS.ORDER_ID.`in`(orderIds))
        .fetch()
        .groupBy(
            { it[ORDER_ITEMS.ORDER_ID]!! },
            {
                OrderItem(
                    id          = it[ORDER_ITEMS.ID]!!,
                    productName = it[PRODUCTS.NAME]!!,
                    quantity    = it[ORDER_ITEMS.QUANTITY]!!,
                )
            },
        )

// 호출부에서 합친다 — 쿼리 2번으로 고정
val orders = findOrders(...)
val itemsByOrderId = findItemsByOrderIds(orders.map { it.id })
```

쿼리 수가 **데이터 건수와 무관하게 상수**로 고정된다. 대부분의 경우 이걸로 충분하다.

### 1.2 MULTISET — 중첩 구조가 필요할 때

jOOQ 3.15+ 의 `multiset` 은 연관 컬렉션을 **타입 세이프하게 한 쿼리로** 가져온다. 문자열 SQL 조각을 쓰는 `JSON_AGG` 수작업과 달리 컴파일 타임 검증이 유지된다.

```kotlin
val items = multiset(
    select(PRODUCTS.NAME, ORDER_ITEMS.QUANTITY)
        .from(ORDER_ITEMS)
        .join(PRODUCTS).on(PRODUCTS.ID.eq(ORDER_ITEMS.PRODUCT_ID))
        .where(ORDER_ITEMS.ORDER_ID.eq(ORDERS.ID))
).convertFrom { r -> r.map { OrderItem(it.value1()!!, it.value2()!!) } }

dsl.select(ORDERS.ID, ORDERS.STATUS, items)
    .from(ORDERS)
    .where(condition)
    .fetch { OrderWithItems(it[ORDERS.ID]!!, OrderStatus.valueOf(it[ORDERS.STATUS]!!), it[items]) }
```

> **트레이드오프:** 쿼리 1회로 줄고 타입 안전성도 지킨다. 대신 생성 SQL 이 복잡해져 실행 계획을 읽기 어렵고, DB 별 지원 편차가 있다.
> **기본은 9.1 batch fetch 로 간다.** 중첩이 2단 이상이거나 왕복 비용이 실제로 측정된 뒤에 9.2 로 전환한다.
> "한 방 쿼리가 멋있어서" 쓰는 건 도입 근거가 아니다.

---

---

---

## 2. 대량 쓰기

한 건씩 `execute()` 를 반복하면 **행 수만큼 DB 왕복**이 생긴다. 1만 건이면 1만 번이다.
읽기 쪽 N+1 을 잡아놓고 쓰기 쪽을 방치하면 같은 문제를 반대편에서 만난다.

### 2.1 세 가지 방식과 고르는 기준

| 방식 | jOOQ | 언제 |
|---|---|---|
| **다중 행 INSERT** | `insertInto(...).values(...).values(...)` | 기본값. 수백~수천 건 |
| **JDBC batch** | `dsl.batch(query).bind(...)...` | 같은 문장을 다른 값으로 아주 많이. 드라이버가 묶어 보낸다 |
| **Loader** | `dsl.loadInto(T)...` | 만 건 이상, 외부 파일/스트림에서 적재. 커밋·에러 정책을 선언적으로 |

작은 규모에서 굳이 batch 로 가지 않는다. **다중 행 INSERT 한 문장이 읽기도 쉽고 실행 계획도 단순하다.**

```kotlin
fun saveAll(items: List<OrderItem>) {
    dsl.insertInto(ORDER_ITEMS, ORDER_ITEMS.ORDER_ID, ORDER_ITEMS.PRODUCT_ID, ORDER_ITEMS.QUANTITY)
        .apply { items.forEach { values(it.orderId, it.productId, it.quantity) } }
        .execute()
}
```

### 2.2 chunk 크기에는 근거가 있어야 한다

"적당히 1000개씩"이 아니라 **상한에서 역산한다.**

> **PostgreSQL 확장 쿼리 프로토콜의 바인드 파라미터 상한은 문장당 65535 개다.**
> 다중 행 INSERT 의 파라미터 수 = `행 수 × 컬럼 수`. 넘으면 드라이버가 던진다.

```kotlin
private const val PG_MAX_BIND_PARAMS = 65535

// 컬럼 수로 나누고 여유를 둔다 — WHERE 절 등 다른 바인드가 섞일 수 있다
private fun chunkSize(columns: Int) = (PG_MAX_BIND_PARAMS / columns / 2).coerceAtLeast(1)

items.chunked(chunkSize(columns = 3)).forEach { chunk -> insertChunk(chunk) }
```

상한이 아니라 **응답 시간과 락 유지 시간**이 먼저 문제가 되는 경우가 더 많다. 그때는 상한과 무관하게 더 작게 자른다. 어느 쪽이든 **숫자에 이유가 붙어 있어야 한다.**

### 2.3 트랜잭션 경계 — 전부를 한 트랜잭션에 넣지 않는다

10만 건을 한 트랜잭션으로 처리하면:

- 락이 작업 내내 유지되고
- 롤백 비용이 작업 시간에 비례해 커지며
- 중간에 실패하면 **전부** 다시 해야 한다

```kotlin
// chunk 단위로 커밋. 실패해도 여기까지는 남는다
items.chunked(size).forEach { chunk ->
    dsl.transaction { cfg -> cfg.dsl().insertChunk(chunk) }
}
```

**대신 부분 성공 상태가 생긴다.** 그래서 chunk 커밋은 **재실행이 안전할 때만** 쓴다 — 즉 2.4 의 멱등성이 전제다.
"전부 아니면 전무"가 도메인 요구라면 단일 트랜잭션이 맞고, 그때는 건수 상한을 두는 게 설계다.

### 2.4 재실행 가능하게 — upsert

배치는 **반드시 다시 돌게 된다.** 재시도, 부분 실패 복구, 운영자의 수동 재실행.
그래서 대량 쓰기는 기본적으로 멱등해야 한다.

```kotlin
dsl.insertInto(PRODUCT_STOCKS, PRODUCT_STOCKS.PRODUCT_ID, PRODUCT_STOCKS.QUANTITY, PRODUCT_STOCKS.UPDATED_AT)
    .apply { rows.forEach { values(it.productId, it.quantity, it.updatedAt) } }
    .onConflict(PRODUCT_STOCKS.PRODUCT_ID)
    .doUpdate()
    .set(PRODUCT_STOCKS.QUANTITY, DSL.excluded(PRODUCT_STOCKS.QUANTITY))
    .set(PRODUCT_STOCKS.UPDATED_AT, DSL.excluded(PRODUCT_STOCKS.UPDATED_AT))
    // 오래된 이벤트가 최신 값을 덮지 않게 — 순서 보장이 없는 파이프라인에서 필수
    .where(PRODUCT_STOCKS.UPDATED_AT.lt(DSL.excluded(PRODUCT_STOCKS.UPDATED_AT)))
    .execute()
```

`DSL.excluded(...)` 가 "지금 넣으려던 값"이다. `where` 절의 조건이 **늦게 도착한 오래된 이벤트를 버리는** 부분이고, 이게 없으면 재실행이 데이터를 되돌린다.

### 2.5 생성 키가 필요하면 한 번에 받는다

```kotlin
val ids: List<UUID> = dsl.insertInto(ORDERS, ORDERS.CUSTOMER_ID, ORDERS.STATUS)
    .apply { rows.forEach { values(it.customerId, it.status.name) } }
    .returningResult(ORDERS.ID)
    .fetch(ORDERS.ID)
```

삽입 후 별도 SELECT 로 다시 찾지 않는다 — 그 조회는 경합에서 틀린 행을 집을 수 있다.

### 2.6 하지 않는 것

| 하지 않음 | 이유 |
|---|---|
| 루프 안에서 `execute()` 한 건씩 | 행 수만큼 왕복. 쓰기 쪽 N+1 |
| 무한정 `chunked` 없이 전체 적재 | 파라미터 상한 초과 또는 OOM |
| 값을 문자열로 접합해 한 문장 만들기 | SQL 인젝션. 바인드 변수를 쓴다 |
| 대량 쓰기를 외부 호출과 같은 트랜잭션에 | 커넥션 풀 고갈 — §3 |
| 멱등성 없이 chunk 커밋 | 재실행이 데이터를 망가뜨린다 |

## 3. 트랜잭션 경계와 Outbox

**외부 호출은 트랜잭션 안에 들어가면 안 된다.** 외부 API 가 느려지면 DB 커넥션이 그 시간만큼 잡혀 있고, 결국 커넥션 풀이 마른다. 트랜잭션 안에서는 **의도만 기록**한다.

```kotlin
@Transactional   // UseCase 레이어에만
class PlaceOrderUseCase(
    private val orderPort: OrderCommandPort,
    private val outboxPort: OutboxPort,
) {
    fun execute(cmd: PlaceOrderCommand): Order {
        val order = Order.create(cmd)
        orderPort.save(order)

        // 외부 호출을 직접 하지 않는다. 이벤트만 INSERT.
        outboxPort.publish(
            OrderPlacedEvent(orderId = order.id, customerId = order.customerId)
        )

        return order
        // commit 이후 outbox worker 가 결제사/알림/분석 도구로 발행한다
    }
}
```

`@Transactional` 은 **UseCase 레이어에만** 붙인다. Adapter 에 붙이면 경계가 흐려지고 외부 호출이 트랜잭션 안으로 새어 들어간다.

---
