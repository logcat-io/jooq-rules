---
name: jooq-rules
description: jOOQ 로 쿼리를 작성·리뷰할 때 적용하는 규약. 코드생성 메타모델, 프로젝션 우선(selectFrom 은 전부 필요할 때만), Record→도메인 매핑 경계, cursor 페이지네이션, 동적 조건 조립, N+1 대응(batch fetch / MULTISET), 트랜잭션 경계와 Outbox. jOOQ 를 ORM 이 아니라 타입세이프 SQL DSL 로 쓰기 위한 룰.
---

# jOOQ 사용 규약

> jOOQ 는 **타입 세이프 SQL DSL** 이다. ORM 이 아니다.
> SQL 을 숨기는 도구가 아니라, **SQL 을 컴파일 타임에 검증 가능하게 만드는** 도구다.

이 구분이 모든 규칙의 뿌리다. "SQL 이 안 보이게" 만드는 추상화는 jOOQ 를 쓰는 이유를 지운다.

예제 도메인은 커머스(`ORDERS` / `ORDER_ITEMS` / `PRODUCTS` / `CUSTOMERS`)를 쓴다. 실제 프로젝트의 테이블·정책은 해당 리포에서 가져온다.

---

## 1. 왜 jOOQ 인가

| 비교 | JPA / Hibernate | jOOQ |
|---|---|---|
| 접근 방식 | 객체 중심. SQL 을 숨긴다 | SQL 중심. SQL 이 코드에 보인다 |
| 쿼리 예측 | N+1 이 언제 나는지 모른다 | 실행될 쿼리가 코드에 명시적이다 |
| 복잡한 조회 | JPQL 또는 Native Query 로 탈출 | DSL 로 자연스럽게 표현 |
| 스키마 변경 | 런타임에 터진다 | 코드 생성 → **컴파일 에러**로 먼저 안다 |
| 실수의 발견 시점 | 배포 후 그 쿼리가 실행될 때 | 빌드 시점 |

**한 줄:** JPA 는 단순 CRUD 에 좋다. 목록 조회 / cursor 페이지네이션 / N+1 회피 / 통계처럼 **쿼리 모양을 통제해야 하는 영역**에서는 SQL 이 보이는 jOOQ 가 압도적으로 다루기 쉽다.

**jOOQ 를 쓰면서 하지 말아야 할 것:** jOOQ 위에 다시 ORM 을 얹는 것. 아래 §8 의 "금지되는 추상화" 참조.

---

## 2. 코드 생성 메타모델

모든 프로덕션 쿼리는 코드 생성기가 만든 테이블/필드 클래스를 쓴다.

```kotlin
// ✗ 피한다 — 문자열로 스키마를 참조하면 오타를 런타임에야 안다.
DSL.table("orders")
DSL.field("total_amount")

// ✓ 기본 — 생성 클래스. 오타는 컴파일 에러.
ORDERS
ORDERS.TOTAL_AMOUNT
```

**예외 — codegen 이 표현하지 못하는 것.** DB 고유 함수, JSON/배열 연산자, 전문검색(`ts_rank` 등) 일부는 plain SQL 이 유일한 방법이다. 이건 룰 위반이 아니라 **정당한 탈출구**다. 다만:

```kotlin
// 한 곳에 격리하고, 왜 plain SQL 인지 남긴다
private fun tsRank(query: String) =
    DSL.field("ts_rank(search_vector, plainto_tsquery({0}))", Double::class.java, DSL.`val`(query))
```

- 값은 반드시 **바인드 변수**로 넘긴다 (`{0}` + `DSL.val`). 문자열 접합은 SQL 인젝션이다
- plain SQL 조각은 Adapter 안 private 헬퍼로 모은다. 쿼리 본문에 흩뿌리지 않는다
- **스키마 식별자(테이블·컬럼)를 문자열로 쓰는 것**과 **표현식을 plain SQL 로 쓰는 것**은 다르다. 전자는 피하고, 후자는 필요하면 쓴다

> **왜?** 컬럼을 문자열로 쓰면 `total_amount` 를 `totalAmount` 로 오타 내도 컴파일이 통과한다.
> 운영에 배포된 뒤 그 쿼리가 실행되는 순간에야 터진다.
> 생성 클래스를 쓰면 스키마가 바뀔 때 codegen → **컴파일 에러로 영향 범위가 즉시 드러난다.** 이게 jOOQ 를 쓰는 가장 큰 이유다.

**스키마 변경 절차 (순서 고정):**

```bash
# 1. 마이그레이션 SQL 추가 (Flyway 등)
# 2. 마이그레이션 실행
./gradlew flywayMigrate
# 3. 코드 재생성
./gradlew jooqCodegen
# 4. 컴파일 — 영향받는 쿼리가 에러로 올라온다
./gradlew build
```

4번을 건너뛰면 codegen 의 의미가 없다. **컴파일이 곧 영향도 분석이다.**

생성 패키지는 정적 import 로 DSL 가독성을 높인다.

---

## 3. 프로젝션 — 쿼리가 무엇을 읽는지 명시한다

**프로젝션은 성능 최적화이기 이전에 계약이다.** 어떤 컬럼을 읽는지가 코드에 적혀 있으면, 그 쿼리가 무엇에 의존하는지 읽는 것만으로 안다.
그래서 규칙은 "`selectFrom` 을 쓰지 마라"가 아니라 **"이 쿼리가 무엇을 읽는지 의도가 드러나게 하라"** 다.

### 3.1 기본값은 명시적 프로젝션

**`selectFrom` 은 금지가 아니다.** 다만 대부분의 조회에서 프로젝션이 더 낫기 때문에 **기본값을 프로젝션으로 둔다.**

```kotlin
// 기본 — 이 쿼리가 쓰는 컬럼만 명시
dsl.select(ORDERS.ID, ORDERS.STATUS, ORDERS.TOTAL_AMOUNT, ORDERS.CREATED_AT)
    .from(ORDERS)
    .where(ORDERS.CUSTOMER_ID.eq(customerId))

// 정당 — 레코드 전체가 실제로 필요한 경우
dsl.selectFrom(ORDERS)
    .where(ORDERS.ID.eq(id))
    .forUpdate()
```

**판단 기준 한 줄: "이 쿼리가 읽고도 안 쓰는 컬럼이 있는가?"**
있으면 프로젝션. 없으면 `selectFrom` 을 써도 된다 — 컬럼을 전부 나열하는 건 같은 말을 길게 하는 것뿐이다.

`selectFrom` 이 자연스러운 경우:
- 잠금 후 갱신 (`forUpdate` → 도메인 재구성 → UPDATE)
- 단건 상세 조회에서 정말 모든 필드를 쓰는 경우
- 컬럼이 적고 안정적인 코드 테이블·매핑 테이블

> **왜 그래도 기본값이 프로젝션인가?** 테이블 컬럼은 시간이 지나면 **늘어난다.**
> 목록 조회가 `SELECT *` 이면, 누군가 `orders.raw_payload JSONB` 를 추가한 순간
> **아무도 그 쿼리를 건드리지 않았는데 응답 시간이 늘어난다.** 느려진 쿼리의 코드에는 변경 이력이 없어서 원인 추적도 어렵다.
>
> 남는 트레이드오프: 지금은 모든 컬럼을 쓰더라도 **나중에 추가되는 컬럼은 자동으로 딸려온다.**
> 그래서 **넓어질 테이블과 목록 경로**는 "지금 다 필요해 보여도" 프로젝션이 안전하다.
> 반대로 좁고 안정적인 테이블의 단건 조회라면 `selectFrom` 이 더 읽기 좋다.

### 3.2 유스케이스마다 프로젝션이 다르다

같은 테이블이라도 목록과 상세가 읽는 컬럼은 다르다. **하나의 "만능 조회"를 만들지 않는다.**

```kotlin
// 목록 — 카드에 필요한 것만
private val ORDER_SUMMARY = listOf(
    ORDERS.ID, ORDERS.STATUS, ORDERS.TOTAL_AMOUNT, ORDERS.CREATED_AT,
)

// 상세 — 배송지·메모까지
private val ORDER_DETAIL = ORDER_SUMMARY + listOf(
    ORDERS.SHIPPING_ADDRESS, ORDERS.MEMO, ORDERS.UPDATED_AT,
)
```

> 프로젝션을 상수로 뽑으면 **"이 화면이 어떤 데이터를 쓰는지"가 한 곳에 모인다.** 컬럼 추가 시 어디를 손봐야 하는지도 명확해진다.
> 단, 상수 재사용이 **서로 다른 유스케이스를 억지로 묶는 수단이 되면 안 된다.** 목록과 상세가 같은 상수를 쓰기 시작하면 곧 목록이 상세만큼 무거워진다.

### 3.3 프로젝션 결과가 엔티티가 아닐 때는 전용 타입으로

집계·조인 결과를 도메인 엔티티에 억지로 담지 않는다.

```kotlin
// ✗ 도메인 엔티티에 통계 필드를 끼워넣는다 — 대부분의 호출처에서 null
data class Order(val id: UUID, /* ... */, val itemCount: Int?)

// ✓ 읽기 전용 프로젝션 타입을 따로 둔다
data class OrderSummary(
    val id: UUID,
    val status: OrderStatus,
    val totalAmount: BigDecimal,
    val itemCount: Int,
)

dsl.select(
        ORDERS.ID,
        ORDERS.STATUS,
        ORDERS.TOTAL_AMOUNT,
        DSL.count(ORDER_ITEMS.ID).`as`("item_count"),
    )
    .from(ORDERS)
    .leftJoin(ORDER_ITEMS).on(ORDER_ITEMS.ORDER_ID.eq(ORDERS.ID))
    .where(ORDERS.CUSTOMER_ID.eq(customerId))
    .groupBy(ORDERS.ID, ORDERS.STATUS, ORDERS.TOTAL_AMOUNT)
    .fetch { it.toOrderSummary() }
```

> **왜?** 엔티티에 nullable 통계 필드를 붙이기 시작하면, 그 필드가 채워진 경로와 안 채워진 경로를 호출처가 알 수 없다.
> "이 Order 의 itemCount 는 믿어도 되나?"를 매번 확인해야 한다면 그건 타입이 거짓말을 하는 것이다.

---

## 4. 쿼리 스타일

### 4.1 SQL 순서 그대로 쓴다

```kotlin
dsl.select(ORDERS.ID, ORDERS.STATUS, ORDERS.TOTAL_AMOUNT, ORDERS.CREATED_AT)
    .from(ORDERS)
    .where(
        ORDERS.CUSTOMER_ID.eq(customerId)
            .and(ORDERS.DELETED_AT.isNull)
    )
    .orderBy(ORDERS.CREATED_AT.desc(), ORDERS.ID.desc())
    .limit(20)
    .fetch { it.toOrder() }
```

select → from → where → orderBy → limit. **읽으면 SQL 이 보여야 한다.**

### 4.2 복잡한 조건은 의미 있는 변수로 분리

```kotlin
// 조건이 길어지면 무엇을 거르는지 안 보인다
dsl.select(...)
    .from(PAYMENTS)
    .where(
        PAYMENTS.STATUS.eq("PENDING")
            .and(PAYMENTS.SCHEDULED_AT.le(now))
            .and(PAYMENTS.RETRY_COUNT.lt(MAX_RETRY))
    )

// 이름을 붙이면 의도가 드러난다
val pending   = PAYMENTS.STATUS.eq("PENDING")
val due       = PAYMENTS.SCHEDULED_AT.le(now)
val retryable = PAYMENTS.RETRY_COUNT.lt(MAX_RETRY)

dsl.select(...)
    .from(PAYMENTS)
    .where(pending.and(due).and(retryable))
```

변수명이 곧 도메인 규칙의 문서다.

---

## 5. 타입 안전성과 널 처리

```
1. 컬럼 타입은 생성된 타입 그대로 쓴다 — field("col", String::class.java) 같은 캐스팅 지양
2. nullable 컬럼의 디폴트 값은 도메인 모델이 정한다 — Adapter 에서 ?: "" / ?: 0 금지
```

> **왜 Adapter 에서 디폴트를 넣으면 안 되는가?**
>
> 비동기로 채워지는 컬럼을 생각해보자. 외부 결제사 응답을 나중에 받아 채우는 `payments.approval_code` 는
> **응답 전에도 NULL 이고, 응답이 실패해도 NULL 이다.** 이 둘은 완전히 다른 상태다.

```kotlin
// ✗ Adapter 에서 임의 디폴트 — 두 상태가 하나로 뭉개진다
approvalCode = this[PAYMENTS.APPROVAL_CODE] ?: ""

// 결과: 도메인이 "아직 안 옴"과 "승인 실패"를 구분할 수 없다.
// UI 는 전자면 "처리 중", 후자면 "실패, 재시도"를 보여야 하는데 분기 불가.

// ✓ 도메인 모델이 디폴트 정책을 소유한다
data class Payment(
    val id: UUID,
    val approvalCode: String?,       // NULL 그대로 보존
    val status: PaymentStatus,       // 상태는 상태 컬럼이 말한다
) {
    val displayLabel: String get() = when {
        status == PaymentStatus.PENDING -> "처리 중"
        approvalCode.isNullOrBlank()    -> "승인 실패"
        else                            -> "승인 완료 ($approvalCode)"
    }
}
```

**원칙: Adapter 는 DB 가 말한 것을 그대로 옮긴다. 해석은 도메인이 한다.**

---

## 6. 매핑 경계

```
1. jOOQ 는 infrastructure(adapter) 레이어에서만 쓴다
2. 도메인 서비스·UseCase 에 jOOQ 타입이 노출되지 않는다
3. Record → 도메인 모델 변환은 Adapter 내부 private 함수
```

```kotlin
@Repository
class OrderJooqAdapter(
    private val dsl: DSLContext,
) : OrderCommandPort, OrderQueryPort {

    override fun findById(id: UUID): Order? =
        dsl.select(
                ORDERS.ID, ORDERS.CUSTOMER_ID, ORDERS.STATUS,
                ORDERS.TOTAL_AMOUNT, ORDERS.SHIPPING_ADDRESS,
                ORDERS.MEMO, ORDERS.CREATED_AT,
            )
            .from(ORDERS)
            .where(
                ORDERS.ID.eq(id)
                    .and(ORDERS.DELETED_AT.isNull)
            )
            .fetchOne { it.toOrder() }

    // 매퍼는 private — 외부에서 Record 를 직접 다루지 못하게 한다
    private fun Record.toOrder(): Order = Order(
        id              = this[ORDERS.ID]!!,
        customerId      = this[ORDERS.CUSTOMER_ID]!!,
        status          = OrderStatus.valueOf(this[ORDERS.STATUS]!!),
        totalAmount     = this[ORDERS.TOTAL_AMOUNT]!!,
        shippingAddress = this[ORDERS.SHIPPING_ADDRESS],  // nullable 그대로
        memo            = this[ORDERS.MEMO],              // nullable 그대로
        createdAt       = this[ORDERS.CREATED_AT]!!,
    )
}
```

> **왜 매퍼가 private 인가?**
> public 이면 다른 Adapter 나 UseCase 가 Record 를 직접 변환하기 시작한다.
> 그러면 필드 하나 추가할 때 **"변환 로직이 어디 어디 있더라"** 를 찾아다녀야 한다.
> private 이면 변환은 항상 그 Adapter 안에서만 일어난다 — 수정 범위가 파일 하나로 고정된다.

Port 의 반환 타입은 언제나 도메인 모델이다. `Record` 나 `Result<*>` 가 Port 시그니처에 나타나면 경계가 이미 깨진 것이다.

---

## 7. 페이지네이션

### 7.1 기본값은 cursor, OFFSET 은 유계일 때만

**사용자에게 계속 쌓이는 목록의 기본값은 cursor 다.**

```kotlin
// 쌓이는 목록에서 위험 — page 가 커질수록 스캔량이 선형으로 는다
    .orderBy(ORDERS.CREATED_AT.desc())
    .limit(20)
    .offset(page * 20)   // page=1000 이면 20,000 행을 읽고 20개만 반환
```

> **왜?** OFFSET 은 "N 행을 건너뛰라"가 아니라 **"N 행을 읽고 버려라"** 다.
> 인덱스가 있어도 OFFSET 자체를 없애주지는 못한다.
> 데이터가 쌓이는 테이블에서는 **어느 날 갑자기 뒷페이지만 느려진다** — 아무도 그 쿼리를 건드리지 않았는데.

**OFFSET 이 맞는 경우도 있다:**

| 상황 | 이유 |
|---|---|
| 총량이 작고 고정된 집합 (코드 테이블, 설정 목록) | 스캔량 자체가 작다 |
| 상한이 명확한 조회 ("최근 200건까지만") | 뒷페이지가 존재하지 않는다 |
| **임의 페이지 점프가 요구사항** (관리자 화면의 "7페이지로") | cursor 로는 불가능하다 |
| 총 페이지 수를 보여줘야 하는 UI | cursor 에는 그 개념이 없다 |

> **cursor 의 대가:** 앞뒤 순차 이동만 된다. **"N 페이지로 점프"와 "전체 몇 페이지"를 포기하는 것**이 cursor 선택의 트레이드오프다.
> 무한 스크롤·더보기 UI 면 애초에 그 기능이 필요 없으니 cursor 가 맞고, 페이지 번호가 박힌 관리자 테이블이면 OFFSET 이 맞다.
> **UI 요구사항을 먼저 확인하고 고른다.**

### 7.2 Cursor 기반 (기준값 방식)

```kotlin
fun findOrders(
    customerId: UUID,
    status: OrderStatus,
    cursor: OrderCursor?,
    size: Int,
): List<Order> {
    val base = ORDERS.CUSTOMER_ID.eq(customerId)
        .and(ORDERS.STATUS.eq(status.name))
        .and(ORDERS.DELETED_AT.isNull)

    val condition = if (cursor == null) base else base.and(
        ORDERS.CREATED_AT.lt(cursor.createdAt)
            .or(
                ORDERS.CREATED_AT.eq(cursor.createdAt)
                    .and(ORDERS.ID.lt(cursor.id))
            )
    )

    return dsl.select(ORDERS.ID, ORDERS.STATUS, ORDERS.TOTAL_AMOUNT, ORDERS.CREATED_AT)
        .from(ORDERS)
        .where(condition)
        .orderBy(ORDERS.CREATED_AT.desc(), ORDERS.ID.desc())  // ← cursor 와 정렬 기준이 같아야 한다
        .limit(size)
        .fetch { it.toOrderSummary() }
}
```

> cursor 에는 **정렬 기준 컬럼 값을 전부** 담는다. `created_at` 만으로는 같은 시각 데이터에서 중복/누락이 난다.
> 그래서 단조 증가하는 `id` 를 tie-breaker 로 함께 쓴다.
> 정렬이 `(priority DESC, created_at DESC, id DESC)` 3중 키라면 **cursor 도 3개 값을 전부 봉인해야 한다.** 하나라도 빠지면 페이지 경계에서 행이 새거나 겹친다.

### 7.3 Cursor 객체

```kotlin
sealed class OrderCursor {
    abstract val createdAt: Instant
    abstract val id: UUID

    data class ByCreatedAt(
        override val createdAt: Instant,
        override val id: UUID,
    ) : OrderCursor()

    data class ByPriority(
        val priority: Int,
        override val createdAt: Instant,
        override val id: UUID,
    ) : OrderCursor()
}

data class OrderPage(
    val items: List<OrderSummary>,
    val nextCursor: OrderCursor?,   // null 이면 마지막 페이지
) {
    companion object {
        fun of(
            items: List<OrderSummary>,
            requestedSize: Int,
            cursorOf: (OrderSummary) -> OrderCursor,
        ): OrderPage {
            val hasNext = items.size == requestedSize
            return OrderPage(items, if (hasNext) cursorOf(items.last()) else null)
        }
    }
}
```

**외부에 노출하는 cursor 는 불투명 문자열로 인코딩하고 서명한다.** 정렬 키 값을 날것으로 내보내면 클라이언트가 조작해 다른 사용자의 범위를 긁을 수 있다. 최소한 **호출자 식별자와 필터 조건을 cursor 안에 봉인**하고, 복호화 시 요청자와 대조한다.

### 7.4 COUNT 는 분리한다

```kotlin
val items = dsl.select(ORDER_SUMMARY)
    .from(ORDERS).where(condition)
    .orderBy(ORDERS.CREATED_AT.desc(), ORDERS.ID.desc())
    .limit(size)
    .fetch { it.toOrderSummary() }

val total = dsl.selectCount()
    .from(ORDERS).where(condition)
    .fetchOne(0, Long::class.java) ?: 0L
```

COUNT 를 JOIN 안에 섞으면 행이 중복되고, 큰 테이블에서는 비용이 본 쿼리보다 커진다.

> COUNT 가 비싼 규모라면 **정확한 총계를 포기하는 것도 설계다.** 추정치(통계 기반)를 쓰거나, 아예 총계를 없애고 "다음 페이지 있음/없음"만 반환한다. 사용자가 실제로 총 건수를 필요로 하는지 먼저 묻는다.

### 7.5 체크리스트

```
□ 쌓이는 목록이면 cursor — OFFSET 을 썼다면 유계이거나 페이지 점프가 요구사항인가
□ ORDER BY 명시 — 정렬이 없으면 cursor 는 무의미하다
□ cursor 컬럼 조합에 인덱스 존재 (예: (customer_id, status, created_at DESC, id DESC))
□ tie-breaker 로 단조 증가 id 사용
□ 다중 정렬키면 cursor 에 모든 키 값 봉인
□ 외부 노출 cursor 는 서명 + 호출자/필터 봉인
□ nextCursor 가 null 이면 마지막 페이지임을 응답에 명시
□ COUNT 는 별도 쿼리 (또는 제거)
□ soft delete 테이블은 deleted_at IS NULL 누락 없음
```

---

## 8. 동적 쿼리와 재사용

```kotlin
private fun buildSearchCondition(filter: OrderSearchFilter): Condition {
    var condition = ORDERS.CUSTOMER_ID.eq(filter.customerId)
        .and(ORDERS.DELETED_AT.isNull)

    filter.status?.let  { condition = condition.and(ORDERS.STATUS.eq(it.name)) }
    filter.minAmount?.let { condition = condition.and(ORDERS.TOTAL_AMOUNT.ge(it)) }
    filter.from?.let    { condition = condition.and(ORDERS.CREATED_AT.ge(it)) }

    return condition
}
```

**허용되는 추상화:** 복잡한 SQL 을 읽기 쉽게 만드는 함수 — 조건 조립, 프로젝션 상수, 매퍼
**금지되는 추상화:** ORM 처럼 쿼리 모양 자체를 감추는 것

```kotlin
// ✗ 금지 — 어떤 SQL 이 나가는지 호출처에서 알 수 없다
fun findAll(spec: OrderSpec): List<Order> = queryBuilder.withSpec(spec).execute()

// ✓ 허용 — SQL 구조는 보이고 조건만 분리됐다
fun search(filter: OrderSearchFilter): List<OrderSummary> {
    val condition = buildSearchCondition(filter)
    return dsl.select(ORDER_SUMMARY)
        .from(ORDERS)
        .where(condition)
        .orderBy(ORDERS.CREATED_AT.desc(), ORDERS.ID.desc())
        .limit(filter.size)
        .fetch { it.toOrderSummary() }
}
```

판별 기준 한 줄: **"이 함수를 읽고 실행될 SQL 을 그릴 수 있는가?"** 못 그리면 과한 추상화다.

---

## 9. N+1 방지

목록 응답이 연관 데이터를 함께 반환하면(주문 20건 × 항목 + 상품 + 고객) 순진하게 짜서 수십 쿼리가 나간다.

### 9.1 batch fetch — 기본값

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

### 9.2 MULTISET — 중첩 구조가 필요할 때

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

## 10. 트랜잭션 경계와 Outbox

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

## 11. 새 Adapter 체크리스트

```
□ 스키마 식별자는 생성 메타모델 — plain SQL 은 표현식에만, 바인드 변수로, 한 곳에 격리
□ 프로젝션 명시가 기본 — selectFrom 을 썼다면 모든 컬럼이 실제로 필요한 경우인가
□ 유스케이스별 프로젝션 분리 (목록 ≠ 상세)
□ 집계·조인 결과는 전용 프로젝션 타입으로 (엔티티에 nullable 통계 필드 금지)
□ Record → 도메인 매퍼가 private
□ Port 반환 타입이 도메인 모델 — Record 노출 없음
□ nullable 디폴트를 Adapter 에서 넣지 않음
□ SQL 순서(select→from→where→orderBy→limit)가 자연스러움
□ 복잡한 조건은 의미 있는 변수로 분리
□ 페이지네이션 방식이 UI 요구와 맞는가 (쌓이는 목록→cursor / 페이지 점프 필요→OFFSET) — §7.5
□ N+1 위험 시 batch fetch 기본, MULTISET 은 측정 후
□ 외부 호출은 트랜잭션 밖 — 안에서는 outbox INSERT 만
□ soft delete 테이블은 deleted_at IS NULL 필터 확인
```
