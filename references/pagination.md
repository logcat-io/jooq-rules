# 페이지네이션

> `SKILL.md` §페이지네이션의 상세. cursor 설계·tie-breaker·서명·COUNT 를 실제로 구현할 때 읽는다.

---

## 1. 기본값은 cursor, OFFSET 은 유계일 때만

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

## 2. Cursor 기반 (기준값 방식)

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

## 3. Cursor 객체

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

---

## 4. 외부에 노출하는 cursor 는 서명한다

정렬 키 값을 날것으로 내보내면 클라이언트가 **그 값을 고쳐서 다른 범위를 긁을 수 있다.** cursor 는 "다음 페이지 위치"이기 이전에 **서버가 발급한 토큰**이다.

세 가지를 cursor 안에 봉인한다.

| 봉인 대상 | 없으면 생기는 일 |
|---|---|
| 정렬 키 값 | (본체) |
| **호출자 식별자** | 남의 `user_id` 로 바꿔치기 → 타인 데이터 열람 |
| **필터 조건 해시** | 같은 cursor 로 필터만 바꿔 재요청 → 페이지 경계가 어긋나 중복·누락 |

### 구현

```kotlin
class CursorCodec(secret: ByteArray) {
    private val key = SecretKeySpec(secret, "HmacSHA256")
    private val enc = Base64.getUrlEncoder().withoutPadding()
    private val dec = Base64.getUrlDecoder()

    fun encode(payload: CursorPayload): String {
        val body = enc.encodeToString(Json.encodeToString(payload).toByteArray())
        return "$body.${sign(body)}"
    }

    fun decode(token: String, callerId: UUID, filterHash: String): CursorPayload {
        val (body, mac) = token.split('.', limit = 2)
            .takeIf { it.size == 2 } ?: throw InvalidCursorException()

        // 1) 서명 먼저 — 위조면 파싱조차 하지 않는다
        if (!MessageDigest.isEqual(sign(body).toByteArray(), mac.toByteArray()))
            throw InvalidCursorException()

        val payload = Json.decodeFromString<CursorPayload>(String(dec.decode(body)))

        // 2) 서명이 맞아도 '누가 요청했는지' 는 별도 검증이다
        //    (자기 cursor 를 다른 필터로 재사용하는 것도 여기서 막힌다)
        if (payload.callerId != callerId) throw InvalidCursorException()
        if (payload.filterHash != filterHash) throw CursorFilterChangedException()

        return payload
    }

    private fun sign(body: String): String =
        enc.encodeToString(Mac.getInstance("HmacSHA256").apply { init(key) }
            .doFinal(body.toByteArray()))
}

@Serializable
data class CursorPayload(
    val callerId: UUID,
    val filterHash: String,      // 정렬·필터 조건을 정규화해 해시
    val createdAt: Instant,
    val id: UUID,
)
```

**세부에서 틀리기 쉬운 것 넷:**

- **비교는 `MessageDigest.isEqual`.** `==` 나 `equals` 는 첫 불일치에서 즉시 반환해 **타이밍 공격**에 노출된다
- **서명 검증을 파싱보다 먼저.** 위조된 페이로드를 역직렬화하는 것 자체가 공격면이다
- **서명이 맞아도 호출자 검증은 따로 한다.** 자기 cursor 를 훔쳐 쓰는 게 아니라 **자기가 받은 cursor 를 다른 필터로 재사용**하는 경우는 서명이 통과한다
- **Base64URL 을 쓴다.** 일반 Base64 의 `+` `/` 는 쿼리스트링에서 깨진다

**암호화가 아니라 서명이다.** 페이로드는 Base64 를 벗기면 읽힌다 — 막는 건 **위조**지 열람이 아니다. cursor 에 비밀을 넣지 않는다. 내부 ID 노출이 곤란하면 그건 애초에 cursor 가 아니라 ID 설계 문제다.

만료가 필요하면 `issuedAt` 을 넣고 decode 에서 확인한다. 목록 cursor 는 대개 만료가 불필요하지만, 필터가 무거워 캐시를 태우는 API 라면 넣는다.

## 5. COUNT 는 분리한다

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

## 6. 체크리스트

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
