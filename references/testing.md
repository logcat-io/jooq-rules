# 테스트 전략 — 컴파일이 못 잡는 것

> 이 규약은 "SQL 을 컴파일 타임에 검증 가능하게 만든다"를 근거로 삼는다.
> 그렇다면 **컴파일이 잡지 못하는 것**이 정확히 테스트의 범위다. 그 경계를 먼저 긋는다.

---

## 1. 경계

| 컴파일이 잡는다 | 컴파일이 못 잡는다 |
|---|---|
| 컬럼명 오타 | **조인 카디널리티** — 행이 몇 배로 불어나는지 |
| 타입 불일치 (`String` ↔ `Int`) | **조건 논리** — `and` 를 `or` 로 쓴 것 |
| 없어진 컬럼 참조 (codegen 후) | **soft delete 필터 누락** — 삭제된 행이 섞이는 것 |
| 존재하지 않는 테이블 | **cursor 경계** — 페이지 사이 중복·누락 |
| 함수 시그니처 | **멱등성** — 두 번 돌렸을 때의 결과 |
| | **쿼리 개수** — N+1 회귀 |
| | 인덱스를 타는지 |

왼쪽 열은 codegen 이 이미 해결했다. **테스트는 오른쪽 열만 겨냥한다.** 왼쪽을 테스트로 다시 확인하는 건 중복이다.

---

## 2. 실제 DB 로 돌린다

**인메모리 DB(H2 등)로 대체하지 않는다.** 이 규약이 쓰는 것들 — partial unique index, `jsonb_agg`, `MULTISET`, `FOR UPDATE SKIP LOCKED`, `ON CONFLICT ... WHERE` — 는 방언이 다르거나 아예 없다. 다른 DB 에서 통과한 테스트는 아무것도 보증하지 않는다.

```kotlin
@Testcontainers
abstract class RepositoryTest {
    companion object {
        @Container
        @JvmStatic
        val postgres = PostgreSQLContainer("postgres:16-alpine")

        @DynamicPropertySource
        @JvmStatic
        fun props(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
        }
    }
}
```

**마이그레이션을 테스트에서도 그대로 돌린다.** 테스트용 스키마 DDL 을 따로 두면 진실이 둘이 되고, 그 순간 "테스트는 통과하는데 운영에서 깨지는" 상태가 만들어진다. Flyway 를 붙여두면 **codegen 이 읽은 스키마와 테스트가 쓰는 스키마가 같다는 것**이 매 실행마다 확인된다.

### DSLContext 를 mock 하지 않는다

```kotlin
// ✗ 이 테스트가 증명하는 것: "빌더를 호출했다"
val dsl = mockk<DSLContext>()
every { dsl.select(any()) } returns ...
```

jOOQ 를 mock 하면 **SQL 이 맞는지는 하나도 검증되지 않는다.** 검증되는 건 우리가 우리 코드를 호출했다는 사실뿐이다. 이 문서 전체가 "SQL 이 맞는지"를 다루는데 그걸 빼고 테스트하면 남는 게 없다.

Adapter 는 실제 DB 로 테스트한다. **Port 를 mock 하는 것은 다르다** — UseCase 테스트에서 Fake Port 를 쓰는 건 옳다. 경계가 거기 있기 때문이다.

---

## 3. 반드시 덮는 것 다섯

### 3.1 조인 카디널리티

1:N 조인은 부모 행을 자식 수만큼 복제한다. 컴파일은 통과하고, 합계만 조용히 틀린다.

```kotlin
@Test
fun `주문 목록에 항목이 여러 개여도 주문은 한 번만 나온다`() {
    val orderId = givenOrder(itemCount = 3)

    val orders = repository.findOrders(customerId)

    assertThat(orders).hasSize(1)
    assertThat(orders.first().id).isEqualTo(orderId)
}
```

**금액·건수를 집계하는 쿼리라면 반드시 자식 2개 이상으로 테스트한다.** 자식 1개짜리 픽스처로는 이 버그가 절대 안 잡힌다.

### 3.2 cursor 경계

정렬 키가 같은 행이 페이지 경계에 걸릴 때가 유일하게 위험한 지점이다. **`created_at` 을 똑같이 준 픽스처**로 만든다.

```kotlin
@Test
fun `동일 시각 데이터가 페이지 경계에 걸려도 중복도 누락도 없다`() {
    val now = Instant.parse("2026-08-18T00:00:00Z")
    val ids = (1..10).map { insertOrder(createdAt = now) }   // 전부 같은 시각

    val collected = mutableListOf<UUID>()
    var cursor: OrderCursor? = null
    do {
        val page = repository.findOrders(customerId, cursor, size = 3)
        collected += page.items.map { it.id }
        cursor = page.nextCursor
    } while (cursor != null)

    assertThat(collected).containsExactlyInAnyOrderElementsOf(ids)
    assertThat(collected).doesNotHaveDuplicates()
}
```

`size` 가 전체 건수를 나누어떨어지지 **않게** 잡는다(10건에 3씩). 나누어떨어지면 마지막 페이지 처리 버그가 숨는다.

### 3.3 soft delete 필터

`deleted_at IS NULL` 은 빠뜨리기 가장 쉽고, 빠져도 개발 중에는 티가 안 난다 — 지운 데이터가 없으니까.

```kotlin
@Test
fun `삭제된 주문은 조회되지 않는다`() {
    val alive = insertOrder()
    val deleted = insertOrder(deletedAt = Instant.now())

    assertThat(repository.findOrders(customerId).map { it.id })
        .containsExactly(alive)
        .doesNotContain(deleted)
}
```

조회 메서드마다 이 테스트를 둔다. 하나에만 두면 나중에 추가된 메서드가 새는데, 그건 운영에서 발견된다.

### 3.4 upsert 멱등성

**"두 번 돌려도 같은 결과"를 결과값이 아니라 물리적 증거로 확인한다.**

```kotlin
@Test
fun `같은 이벤트를 두 번 적용해도 행이 한 번만 쓰인다`() {
    val event = stockEvent(productId, quantity = 10, updatedAt = t1)

    repository.applyStock(listOf(event))
    val afterFirst = fetchXmin(productId)      // PostgreSQL 시스템 컬럼

    repository.applyStock(listOf(event))       // 같은 이벤트 재적용
    val afterSecond = fetchXmin(productId)

    assertThat(afterSecond).isEqualTo(afterFirst)   // 물리 쓰기가 없었다
    assertThat(fetchQuantity(productId)).isEqualTo(10)
}

@Test
fun `늦게 도착한 오래된 이벤트는 최신 값을 덮지 않는다`() {
    repository.applyStock(listOf(stockEvent(productId, 10, updatedAt = t2)))
    repository.applyStock(listOf(stockEvent(productId, 99, updatedAt = t1)))   // t1 < t2

    assertThat(fetchQuantity(productId)).isEqualTo(10)
}
```

행이 바뀌지 않았다는 것은 반환값으로는 증명되지 않는다. `xmin` 같은 **쓰기 흔적**을 봐야 "정말 안 썼다"가 된다.

### 3.5 쿼리 개수 — N+1 회귀

N+1 은 기능 테스트를 전부 통과한다. 느릴 뿐이니까. **그래서 쿼리 수 자체를 단언한다.**

```kotlin
class QueryCounter : DefaultExecuteListener() {
    val count = AtomicInteger()
    fun reset() = count.set(0)
    override fun executeStart(ctx: ExecuteContext) { count.incrementAndGet() }
}

@Test
fun `주문 20건을 항목과 함께 읽어도 쿼리는 2회다`() {
    givenOrders(count = 20, itemsEach = 3)
    counter.reset()

    val orders = service.findOrdersWithItems(customerId)

    assertThat(orders).hasSize(20)
    assertThat(counter.count.get()).isEqualTo(2)   // 주문 1 + 항목 batch 1
}
```

**데이터 건수를 바꿔도 쿼리 수가 그대로인지**가 핵심이다. 20건과 40건으로 두 번 돌려 같은 수가 나오면 상수임이 증명된다. 숫자를 박아두면 나중에 누군가 루프 안에서 조회를 추가했을 때 이 테스트가 먼저 깨진다.

---

## 4. 안 하는 것

| 안 함 | 이유 |
|---|---|
| 인메모리 DB 로 대체 | 방언이 다르다. 통과해도 보증이 없다 |
| `DSLContext` mock | SQL 이 맞는지가 검증에서 빠진다 |
| 테스트 전용 DDL 을 따로 유지 | 스키마 진실이 둘이 된다 |
| 생성된 SQL 문자열 비교 | 포맷·바인드 표현이 바뀌면 깨진다. **동작**을 검증한다 |
| 자식 1개 픽스처로 조인 검증 | 카디널리티 버그가 안 잡힌다 |
| 성능 수치 단언 (`< 100ms`) | 환경 따라 흔들려 신뢰를 잃는다. 대신 **쿼리 수**를 단언한다 |

---

## 5. 체크리스트

```
□ 실제 DB(Testcontainers) 로 돈다 — 인메모리 대체 없음
□ 마이그레이션을 테스트에서도 그대로 실행한다
□ Adapter 테스트에서 DSLContext 를 mock 하지 않는다
□ 1:N 조인 쿼리는 자식 2개 이상 픽스처로 검증
□ cursor 페이지네이션은 동일 정렬값 픽스처 + 나누어떨어지지 않는 size 로 검증
□ 조회 메서드마다 soft delete 필터 테스트
□ upsert 는 재적용 시 물리 쓰기 없음을 증거로 검증
□ 늦게 도착한 이벤트가 최신 값을 덮지 않는지 검증
□ 연관 조회는 쿼리 수를 단언하고, 건수를 바꿔도 상수인지 확인
```
