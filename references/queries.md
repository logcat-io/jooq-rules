# 쿼리 작성 — 코드 생성 · 프로젝션 · 매핑

> `SKILL.md` 의 근거와 예제. 쿼리를 새로 쓰거나 Adapter 를 만들 때 읽는다.

---

## 1. 기준 버전과 codegen 설정

이 문서는 아래를 전제로 쓰였다. **버전을 여기 박아두는 이유는 예제가 컴파일되는 최소 조건을 밝히기 위해서다** — 더 최신이면 대체로 그대로 동작한다.

| 항목 | 기준 | 비고 |
|---|---|---|
| jOOQ | **3.19+** | `MULTISET` 은 3.15+, `Records.mapping` 은 3.15+ |
| Kotlin | 2.0+ | |
| Spring Boot | 3.x 이상 | Boot BOM 이 jOOQ 버전을 관리한다 |
| JDK | 21 | |
| PostgreSQL | 14+ | `jsonb_agg`, partial unique index 사용 |
| 마이그레이션 | Flyway | codegen 이 마이그레이션 적용 후 스키마를 읽는다 |

**버전을 `build.gradle.kts` 에 직접 박지 않는다.** Boot BOM 이 관리하는 것은 BOM 에 맡긴다 — 개별 지정은 BOM 이 검증한 조합을 깨뜨린다.

### codegen 설정

핵심은 **codegen 이 "마이그레이션이 적용된 실제 DB"를 읽게 하는 것**이다. 손으로 유지하는 XML 스키마를 읽게 하면 그 순간 진실이 둘이 된다.

```kotlin
// build.gradle.kts
jooq {
    configurations {
        create("main") {
            jooqConfiguration.apply {
                jdbc.apply {
                    driver = "org.postgresql.Driver"
                    url = "jdbc:postgresql://localhost:5432/app"
                    user = "app"; password = "app"
                }
                generator.apply {
                    name = "org.jooq.codegen.KotlinGenerator"
                    database.apply {
                        name = "org.jooq.meta.postgres.PostgresDatabase"
                        inputSchema = "public"
                        // 마이그레이션 이력 테이블은 도메인이 아니다
                        excludes = "flyway_schema_history"
                    }
                    generate.apply {
                        isDeprecated = false
                        isRecords = true
                        isPojos = false          // 도메인 모델은 직접 만든다. POJO 생성은 안 쓴다
                        isDaos = false           // DAO 는 ORM 회귀다
                        isKotlinNotNullRecordAttributes = true
                    }
                    target.apply {
                        packageName = "com.example.jooq"
                        directory = "build/generated-src/jooq/main"
                    }
                }
            }
        }
    }
}

tasks.named("generateJooq") { dependsOn("flywayMigrate") }
```

**설정에서 읽어야 할 결정 3개:**

| 설정 | 왜 |
|---|---|
| `isPojos = false` | 도메인 모델은 생성물이 아니라 우리가 설계하는 것이다. 생성된 POJO 를 도메인으로 쓰면 스키마가 도메인을 정의하게 된다 |
| `isDaos = false` | 생성 DAO 는 `findAll` / `fetchByX` 를 그냥 준다. 쓰는 순간 프로젝션도 조건도 통제를 잃는다 — jOOQ 위에 ORM 을 얹는 것과 같다 |
| `generateJooq dependsOn flywayMigrate` | 순서를 사람이 기억하게 두면 언젠가 틀린다. 빌드가 강제하게 만든다 |

`build/` 에 생성하고 **커밋하지 않는다.** 생성물을 커밋하면 스키마와 코드가 어긋난 상태가 리포에 남을 수 있다.

CI 에서는 마이그레이션이 적용된 임시 DB(도커, Testcontainers)를 띄워 codegen 을 돌린다. 개발자 로컬 DB 상태에 빌드가 의존하면 "내 컴퓨터에서만 컴파일된다"가 생긴다.

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

---

## 7. 동적 쿼리와 재사용

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
