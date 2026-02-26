# Orbs TWAP (DCA) 컨트랙트 아키텍처

## 개요

Orbs TWAP은 탈중앙화된 DCA(Dollar Cost Averaging) 주문을 가능하게 하는 스마트 컨트랙트 시스템입니다. PancakeSwap 등 DEX와 연동하여 대량 주문을 시간에 걸쳐 분할 체결합니다.

> **핵심 개념**: Orbs는 별도의 L3 블록체인이 아닙니다. L1(Ethereum, BSC 등)에 컨트랙트를 배포하고, Orbs Guardian 네트워크가 오프체인에서 합의 후 트랜잭션을 실행하는 구조입니다.

---

## 컨트랙트 구조

```
┌─────────────────────────────────────────────────────────────────┐
│                          TWAP.sol                               │
│  - 주문 저장소 (book[])                                          │
│  - ask(): 주문 생성                                              │
│  - bid(): 입찰                                                   │
│  - fill(): 체결                                                  │
│  - cancel(): 취소                                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ fill() 시 호출
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       ExchangeV2.sol                            │
│  - IExchange 인터페이스 구현                                     │
│  - getAmountOut(): 예상 출력량 계산                              │
│  - swap(): 실제 스왑 실행                                        │
│  - router: DEX Router 주소 (PancakeSwap 등)                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ functionCall(router, swapData)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      DEX Router                                 │
│               (PancakeSwap, Uniswap 등)                         │
└─────────────────────────────────────────────────────────────────┘
```

### 주요 컨트랙트 파일

| 파일 | 역할 |
|-----|------|
| `src/TWAP.sol` | 메인 컨트랙트 - 주문 관리 및 체결 |
| `src/OrderLib.sol` | Order 구조체 및 유틸리티 |
| `src/exchange/ExchangeV2.sol` | DEX 어댑터 |
| `src/periphery/Lens.sol` | 주문 조회 헬퍼 |
| `src/periphery/Taker.sol` | Guardian용 실행 헬퍼 |

---

## 데이터 구조

### Order

```solidity
struct Order {
    uint64 id;               // 주문 ID
    uint32 status;           // 상태 (진행중/취소/완료)
    uint32 time;             // 생성 시간
    uint32 filledTime;       // 마지막 체결 시간
    uint256 srcFilledAmount; // 총 체결된 금액
    address maker;           // 주문자
    Ask ask;                 // 주문 파라미터
    Bid bid;                 // 현재 낙찰 정보
}
```

### Ask (사용자 주문 파라미터)

```solidity
struct Ask {
    address srcToken;      // 판매할 토큰
    address dstToken;      // 구매할 토큰
    uint256 srcAmount;     // 총 판매 금액
    uint256 srcBidAmount;  // 1회 체결 금액 (청크 크기)
    uint256 dstMinAmount;  // 최소 수령량 (슬리피지/지정가)
    uint32 deadline;       // 주문 만료 시간
    uint32 bidDelay;       // 입찰 대기 시간 (최소 30초)
    uint32 fillDelay;      // 청크 간 간격
    address exchange;      // 특정 DEX 지정 (0 = 아무거나)
    bytes data;            // 추가 데이터
}
```

### Bid (입찰 정보)

```solidity
struct Bid {
    uint32 time;           // 입찰 시간
    address taker;         // 입찰자
    address exchange;      // 사용할 Exchange
    uint256 dstAmount;     // 약속 출력량 (수수료 제외)
    uint256 dstFee;        // Taker 수수료
    bytes data;            // 스왑 경로 데이터
}
```

---

## DCA 작동 예시

**시나리오**: 1000 USDT로 BNB를 10일간 분할 매수

| 파라미터 | 값 | 의미 |
|---------|-----|------|
| srcToken | USDT | 판매 토큰 |
| dstToken | BNB | 구매 토큰 |
| srcAmount | 1000 | 총 금액 |
| srcBidAmount | 100 | 1회 체결량 (10청크) |
| fillDelay | 86400 | 체결 간격 (1일) |
| deadline | 10일 후 | 만료 시간 |
| dstMinAmount | 1 wei | 시장가 주문 |

→ **100 USDT씩 10번**, 하루 간격으로 체결

---

## 체결 흐름

### 전체 프로세스

```
[1] User: ask() 호출
    │
    │  주문 정보 저장 (토큰 이동 없음)
    │  approve만 되어 있으면 됨
    ▼
[2] fillDelay 대기 (청크 간 간격)
    │
    ▼
[3] Taker: bid() 호출
    │
    │  "이 경로로 스왑하면 X만큼 줄 수 있어" 약속
    │  토큰 이동 없음, 상태만 저장
    ▼
[4] bidDelay 대기 (최소 30초)
    │
    │  이 사이에 더 좋은 bid 가능 (경쟁)
    ▼
[5] Taker: fill() 호출 (낙찰자만 가능)
    │
    │  실제 토큰 이동 발생
    │  srcToken: Maker → TWAP → Exchange → DEX
    │  dstToken: DEX → Exchange → TWAP → Maker + Taker
    ▼
[6] 다음 청크로 반복 (srcAmount 소진까지)
```

### 자금 흐름 상세

```
[Maker Wallet]
      │
      │ ① safeTransferFrom (srcToken, 1청크 분량)
      ▼
[TWAP Contract]
      │
      │ ② approve + swap 호출
      ▼
[ExchangeV2]
      │
      │ ③ functionCall(router, swapData)
      ▼
[DEX Router]
      │
      │ ④ 스왑 결과 dstToken 반환
      ▼
[ExchangeV2]
      │
      │ ⑤ safeTransfer → TWAP
      ▼
[TWAP Contract]
      │
      ├── ⑥ dstFee → Taker (수수료)
      │
      └── ⑦ dstAmount → Maker (수령액)
```

---

## Bid 경쟁 메커니즘

### 핵심 상수

```solidity
uint32 public constant MIN_OUTBID_PERCENT = 101_000;  // 1% 더 높아야 함
uint32 public constant STALE_BID_SECONDS = 60 * 10;   // 10분 지나면 stale
uint32 public constant MIN_BID_DELAY_SECONDS = 30;    // 낙찰 후 30초 대기
```

### 경쟁 규칙

```solidity
require(
    dstAmountOut > (o.bid.dstAmount * MIN_OUTBID_PERCENT) / PERCENT_BASE  // 1% 이상 높거나
        || block.timestamp > o.bid.time + STALE_BID_SECONDS,              // 기존 bid가 10분 지남
    "low bid"
);
```

| 규칙 | 조건 |
|-----|------|
| **신규 입찰** | `dstAmountOut`이 기존보다 **1% 이상** 높아야 함 |
| **Stale Bid** | 기존 bid가 **10분** 지났으면 더 낮아도 입찰 가능 |
| **낙찰 대기** | 최종 bid 후 **30초(bidDelay)** 대기해야 fill 가능 |
| **체결 권한** | `msg.sender == o.bid.taker` → 낙찰자만 fill 가능 |

### 경쟁 시나리오

```
T+0s:    Taker A bid (사용자 수령 98)
              │
T+10s:   Taker B bid (사용자 수령 101) → 1% 이상 ✓ 낙찰자 변경
              │
T+15s:   Taker C bid (사용자 수령 100) → 1% 미만 ✗ 실패
              │
              │ 30초 대기 (T+10s 기준)
              ▼
T+40s:   Taker B fill() → 체결 완료
```

**주의**: 새로운 최고가 bid가 들어올 때마다 30초 타이머가 리셋됩니다.

---

## Bid 실패 시나리오

### 가격 변동으로 인한 실패

```
Bid 시점: ETH $3,000 → "0.033 ETH 주겠다"
    │
    │ 30초 대기
    │ ETH 가격 $3,500로 급등
    ▼
Fill 시점: 실제 스왑하면 0.028 ETH만 나옴
```

```solidity
function performFill(...) {
    ...
    uint256 minOut = o.dstExpectedOutNext();  // bid 때 약속한 금액
    ...
    require(dstAmountOut >= minOut, "min out");  // 약속 못 지키면 revert
}
```

| 상황 | 결과 |
|-----|------|
| 가격 불리하게 변동 | `fill()` **revert** → Taker 가스비 손실 |
| 가격 유리하게 변동 | 성공, 초과분은 Maker에게 |
| Taker가 fill 안 함 | 10분 후 Stale → 다른 Taker가 새 bid 가능 |

**리스크는 Taker가 부담** → 그래서 `dstFee`(수수료)를 받음

---

## 설계 철학

### "한 명의 정직한 Taker만 있으면 된다"

경쟁 입찰 구조 덕분에, 가스비 + 최소 마진만 받는 Taker 하나만 있으면 사용자는 시장가에 가까운 가격에 체결됩니다.

### 주요 보안 특징

| 특징 | 설명 |
|-----|------|
| **No Fund Holding** | 컨트랙트가 자금 보관 안 함 - 실행 시점에 직접 스왑 |
| **Immutable** | Owner, admin 없음 - 완전 불변 |
| **Permissionless** | 누구나 Taker로 참여 가능 |
| **Cancellable** | Maker가 언제든 주문 취소 가능 |

---

## Orbs Guardian 역할

실제 운영에서는 Orbs Guardian 네트워크가 Taker 역할을 수행합니다:

```
┌─────────────────────────────────────────┐
│         Orbs Guardian Network           │
│                                         │
│  1. Lens 컨트랙트로 실행 가능 주문 조회   │
│  2. 최적 경로 계산                       │
│  3. 오프체인 합의                        │
│  4. bid() + fill() 트랜잭션 제출         │
└─────────────────────────────────────────┘
```

Guardian들은 경쟁적으로 좋은 가격을 제시하며, 이를 통해 사용자에게 공정한 가격을 보장합니다.

---

## 참고 자료

- [Orbs TWAP GitHub](https://github.com/orbs-network/twap)
- [dTWAP Protocol Docs](https://docs.orbs.network/v3/protocols/dtwap-protocol)
- [TWAP SDK](https://docs.orbs.network/orbs-integrations/twap-sdk)
