+++
title = "MetaMask Gas Station의 거래 제출 경로와 IP 추적 가능성"
date = 2026-10-30
description = "바이낸스 침해 사건의 Gas Station 제보를 바탕으로 MetaMask Wallet과 Sentinel API 사이에서 IP와 거래가 연결될 수 있는 조건을 살펴봅니다."
draft = false

[taxonomies]
tags = ["RealWorld"]

+++

바이낸스 침해 사건을 분석하던 Justin Kalland는 공격자가 사용한 것으로 지목된 지갑에서 MetaMask Gas Station 사용 흔적을 발견했다고 제보했습니다. [글](https://x.com/justinkalland/status/2115946980665151955?a=ViZrzcnhc0LYIZrr_M4&s=20)에 따르면 같은 블록에서 Gas Station이 ETH를 보낸 뒤 해당 지갑은 Kyber 라우터에 USDT 사용 권한을 부여하고 Gas Station에 USDT를 지급했습니다. 지갑에 ETH가 있었는데도 토큰으로 가스비를 냈다는 설명입니다.

Kalland는 이 글에서 ETH 보충량이 뒤이은 두 거래의 최대 가스 예산 합과 wei 단위까지 일치했다고 설명했습니다. 같은 블록의 거래 순서와 이 수량 관계를 Gas Station 사용의 근거로 제시한 것입니다.

MetaMask 공식 계정은 해당 글에 “보안팀이 이 제보를 인지하고 조사 중”이라고 [답했습니다](https://x.com/MetaMask/status/2115977377989955778).

<figure>
<img src="/img/metamask-gas-station/metamask-official-reply.jpg" alt="MetaMask 공식 계정이 보안팀의 조사 사실을 알린 X 답변" width="332" height="153" style="height: auto;" loading="lazy">
<figcaption>제보에 대한 MetaMask 공식 계정의 답변</figcaption>
</figure>

이 글에서는 거래의 제출 경로를 살펴봅니다. Gas Station을 이용하면서 MetaMask 서버에 요청을 보냈다면 해당 서버의 접속 기록도 조사 대상이 됩니다. 공개 코드에서는 이 요청이 어디로 전송되고 어떤 정보를 담는지 확인할 수 있습니다.

## MetaMask Gas Station을 이용한 거래가 블록에 포함되기까지

Sentinel은 MetaMask Wallet이 거래 시뮬레이션과 제출을 요청하고 처리 상태를 조회하는 백엔드 API입니다. 이번에 살펴본 Gas Station 경로에서 거래가 정상적으로 처리될 때의 큰 흐름은 다음과 같습니다.

MetaMask Wallet에서 거래 준비와 서명 → Sentinel API로 요청 전송 → 백엔드의 거래 제출 처리 → 이더리움 블록에 포함

Ethereum Mainnet의 기본 API 주소는 `https://tx-sentinel-ethereum-mainnet.api.cx.metamask.io/`입니다. API 주소는 네트워크별 설정에 따라 달라집니다.

7702 Relay가 사용하는 [JSON-RPC 클라이언트 코드](https://github.com/MetaMask/core/blob/b8d07a5a7731c565ad6c49b555b21b7cceedce61/packages/sentinel-api-service/src/sentinel-api-service.ts#L392-L408)에는 다음과 같은 요청 구성이 있습니다.

```typescript
const response = await this.#fetch(url, {
  method: 'POST',
  headers: {
    ...headers,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    id: '1',
    jsonrpc: '2.0',
    method,
    params,
  }),
});
```

MetaMask는 이 코드로 Sentinel에 보낼 HTTP 요청을 구성합니다. `method`에는 호출할 API 이름이, `params`에는 요청 데이터가 들어갑니다. 7702 Relay에서는 이 방식으로 `eth_sendRelayTransaction`을 호출합니다.

이 제출 요청을 받는 쪽은 Sentinel입니다. 요청을 받는 HTTPS 수신 지점에서는 접속 IP와 요청 내용을 함께 확인할 수 있습니다. 요청 데이터에서 사용자 지갑을 식별할 수 있다면 해당 지갑의 요청과 접속 IP를 연결할 여지가 생깁니다.

![일반 RPC와 MetaMask Gas Station의 제출 경로 및 IP 관측 지점 비교](/img/metamask-gas-station/ip-trace-sketch-en.png)

이 비교 그림에서 실선은 클라이언트 코드에서 확인한 요청 경로를 나타냅니다. 점선으로 표시한 서버 내부의 처리 과정은 직접 확인하지 못했습니다. 노란 상자는 서버가 기록을 보관하고 이를 거래와 연결할 수 있을 때의 예시입니다.

## 일반 RPC 제출과 Gas Station 제출의 차이

일반적인 RPC 제출에서는 지갑이 설정된 RPC에 `eth_sendRawTransaction`을 보냅니다. 이번에 분석한 Gas Station의 두 경로는 제출 요청을 Sentinel로 보냅니다. MetaMask 지갑은 계정 유형과 네트워크 지원 조건 등에 따라 거래를 제출할 방식을 선택합니다.

그림의 주황색 상자는 서버가 접속 상대의 IP와 거래 요청을 함께 확인할 수 있는 지점입니다. 일반 RPC 서버와 Sentinel 서버 모두 여기에 해당합니다. IP는 네트워크 연결에서 관측하는 정보이며 거래의 기본 필드에는 들어 있지 않습니다.

Gas Station 사용 흔적은 요청을 받은 서비스를 좁히는 단서가 됩니다. 사건에서도 분석한 제출 경로를 사용했다면 접속 기록을 확인할 대상은 MetaMask의 Sentinel 서버입니다. Kalland가 MetaMask에 오프체인 로그 보존을 요청한 것도 이 제출 경로 때문입니다.

## Sentinel에 전달되는 정보

공개 코드에서 확인한 두 제출 경로는 Smart Transactions(STX) 번들 제출과 7702 Relay 제출입니다. 둘 다 MetaMask가 구현한 Sentinel 호출 경로입니다.

| 제출 경로 | Sentinel에 보내는 정보 | 지갑 주소를 확인하는 방법 |
|---|---|---|
| STX 번들 제출 | 사용자가 보내려는 트랜잭션과 가스비 결제용 트랜잭션을 각각 서명해 묶은 데이터. REST [`submitTransactions`](https://github.com/MetaMask/core/blob/5c82ed964b68ede7cd6d1c045ddb593ac8bff437/packages/smart-transactions-controller/src/SmartTransactionsController.ts#L950-L988)로 제출 | 서명된 트랜잭션에서 발신 주소 복구 |
| 7702 Relay 제출 | 서명된 위임과 실행 데이터. JSON-RPC [`eth_sendRelayTransaction`](https://github.com/MetaMask/metamask-extension/blob/0997172ccbed2dfd31c56cf070b244d44cfb42ed/app/scripts/lib/transaction/hooks/delegation-7702-publish.ts#L158-L240)으로 제출 | `data`에 인코딩된 위임에서 사용자 주소 확인 |

7702 Relay는 EIP-7702를 활용한 MetaMask의 릴레이 제출 방식입니다. 여기서 JSON-RPC는 요청과 응답의 형식을 뜻하며 이 요청의 수신처도 Sentinel입니다.

7702 Relay는 사용자 주소를 제출할 `data` 안의 위임에 담습니다. 아래 [위임 구성 코드](https://github.com/MetaMask/metamask-extension/blob/0997172ccbed2dfd31c56cf070b244d44cfb42ed/app/scripts/lib/transaction/delegation.ts#L298-L317)는 사용자 지갑 주소인 `transaction.txParams.from`을 `delegator`에 넣습니다.

```typescript
const unsignedDelegation: UnsignedDelegation = {
  delegator: transaction.txParams.from as Hex,
  delegate: delegatee ?? ANY_BENEFICIARY,
  authority: ROOT_AUTHORITY,
  salt,
  caveats,
};
```

`delegator`에 넣은 주소는 이후 위임을 서명하고 인코딩하는 과정에서 실행 데이터와 함께 제출할 `data`에 포함됩니다. Sentinel은 ABI 형식의 요청 데이터를 디코딩해 사용자 주소를 읽을 수 있습니다. 이 요청을 받은 HTTPS 수신 지점에서 접속 IP도 관측한다면 어느 지갑의 요청이 어느 IP에서 왔는지 함께 확인할 수 있습니다.

거래 승인 전에도 서버와 통신할 수 있습니다. 시뮬레이션이 켜져 있고 지원 조건이 맞으면 지갑은 거래 승인을 기다리는 동안 `infura_simulateTransactions`를 호출해 토큰으로 낼 가스비를 조회할 수 있습니다. 이 견적 요청에는 사용자 주소가 `from` 필드로 들어가며 목적지와 호출 데이터도 포함될 수 있습니다. 사용자가 나중에 거래를 거절해도 이미 전송한 요청을 되돌릴 수는 없습니다.

제출 경로는 지갑 버전과 네트워크 및 계정 유형에 따라 달라집니다. STX와 sendBundle 지원 여부나 7702 지원 조건도 영향을 주며 Transaction Pay 같은 별도 경로도 있습니다. Gas Station 사용 흔적만으로 사건 당시의 API와 지갑 버전까지 확정할 수는 없습니다.

## IP와 해당 거래의 연결점

HTTPS 연결을 받는 인프라는 접속 상대의 IP를 처리합니다. TLS 연결을 종료하는 서버에서는 복호화된 요청 내용도 읽을 수 있습니다. 요청에 지갑 주소가 들어 있거나 서명과 위임 데이터에서 주소를 알아낼 수 있다면 수신자는 접속 IP와 지갑 주소를 함께 확인할 수 있습니다. 이 IP가 내부 애플리케이션에도 전달되는지는 프록시 설정에 달려 있습니다.

사후에 이 관계를 추적하려면 기록이 남아 있어야 합니다. 예를 들어 접속 로그에 IP와 시각 및 요청 식별자를 남기고 거래 처리 로그에 지갑 주소와 제출 식별자 또는 해시를 따로 기록할 수 있습니다. 두 기록이 같은 요청을 가리킨다는 것을 확인할 식별자나 자료가 있어야 서로 연결할 수 있습니다.

Sentinel은 제출 요청의 처리 상태를 조회할 때 사용할 UUID를 반환합니다. 지갑은 이 UUID로 처리 결과를 조회하고 상태가 `PENDING`이면 조회를 반복합니다. 아래 [결과 조회 코드](https://github.com/MetaMask/metamask-extension/blob/0997172ccbed2dfd31c56cf070b244d44cfb42ed/app/scripts/lib/transaction/transaction-relay.ts#L58-L114)는 그중 한 번의 조회를 보여줍니다.

```typescript
const { transactions } = await getSentinelApiMessenger().call(
  'SentinelApiService:getSmartTransaction',
  { chainId, uuid },
);

log('Polling response', transactions);

const transaction = transactions[0];

return {
  status: transaction?.status as string,
  transactionHash: transaction?.hash as Hex | undefined,
};
```

이 코드는 UUID로 받은 조회 결과에서 첫 번째 거래의 상태와 해시를 읽습니다. `hash`가 아직 없으면 `transactionHash`는 `undefined`가 됩니다. 지갑은 이 응답에서 제출 UUID에 해당하는 온체인 거래 해시를 얻습니다.

`log(...)`는 조회 응답을 지갑 쪽 디버그 로그에 출력합니다.

서버에 UUID와 원래 제출 요청의 대응 관계가 보관돼 있고 그 요청을 접속 로그와 연결할 수 있다면 이미 알고 있는 온체인 거래 해시를 단서로 제출 요청과 접속 IP를 찾아볼 여지가 있습니다.

<figure id="relay-sequence">
<a href="/img/metamask-gas-station/7702-relay-sequence.svg" target="_blank" rel="noopener">
<img src="/img/metamask-gas-station/7702-relay-sequence.png" alt="MetaMask Wallet, Sentinel API, Blockchain 세 열로 나타낸 7702 Relay 제출과 UUID 결과 조회 구조" width="1500" height="2075" style="height: auto;" loading="lazy" decoding="async">
</a>
<figcaption>7702 Relay의 요청과 응답</figcaption>
</figure>

위쪽의 실선 가로 화살표는 요청을, 점선 가로 화살표는 응답을 나타냅니다. 아래쪽의 온체인 제출은 개념적으로 표시했으며 UUID 응답이나 결과 조회와의 실제 선후 관계를 뜻하지 않습니다. 거래 해시는 블록에 포함되기 전에도 존재합니다.

STX 경로에서는 서명된 트랜잭션이 보관돼 있다면 그 해시를 계산해 온체인 거래와 대조할 수 있습니다.

백엔드에 해당 거래와 연결할 수 있는 접속 로그가 보관돼 있다면 요청에 사용된 IP를 확인할 여지는 있습니다. 다만 VPN이나 프록시 같은 중계 수단을 거쳤다면 그 기록에 남는 주소는 중계 서버의 출구 IP일 수 있습니다.

[TRM Labs](https://www.trmlabs.com/resources/blog/north-korea-steals-usd-6-36-billion-from-binance-in-the-largest-hack-in-history)는 이번 사건의 배후를 북한의 TraderTraitor로 지목했습니다.

북한 연계 조직의 공격이라는 분석이 맞더라도 공격자가 VPN이나 프록시를 거쳐 요청을 보냈다면 MetaMask 서버에는 공격자의 원래 IP 대신 중계 서버의 IP가 기록될 수 있습니다. 이 경우 접속 기록만으로 공격자의 실제 위치나 신원을 특정하기는 어렵습니다.

## 확인한 범위

공개 클라이언트에서 확인한 것은 Sentinel에 보내는 요청과 그 안에 담기는 거래 정보입니다. 서버의 로그 항목이나 보관 방식은 이 코드로 확인할 수 없습니다.

[MetaMask 개인정보 처리방침](https://legal.consensys.io/metamask/privacy-policy/)은 기능 제공과 보안 등에 필요한 경우 IP를 일시적으로 처리한다고 설명합니다. Sentinel이 IP를 지갑 주소나 UUID와 연결해 보관하는지와 그 기간은 이 문서만으로 알 수 없습니다.

분석 기준은 2026년 10월 30일의 Extension 13.56.0 공개 소스와 잠금 파일에 고정된 패키지 버전입니다. 기준 커밋은 `0997172`입니다. 조사 당시 최신 공개 릴리스였던 13.55.0의 전체 의존 경로가 분석 기준 소스와 같은지는 검증하지 않았습니다. 사건 당시 설치본의 버전도 확인하지 못했습니다.

## 참고 자료

- [네트워크별 Sentinel API 주소](https://github.com/MetaMask/core/blob/b8d07a5a7731c565ad6c49b555b21b7cceedce61/packages/sentinel-api-service/src/constants.ts#L10-L89)
- [거래 제출 경로 선택](https://github.com/MetaMask/metamask-extension/blob/0997172ccbed2dfd31c56cf070b244d44cfb42ed/app/scripts/lib/transaction/hooks/index.ts#L133-L215)
- [위임 서명과 실행 데이터 인코딩](https://github.com/MetaMask/metamask-extension/blob/0997172ccbed2dfd31c56cf070b244d44cfb42ed/app/scripts/lib/transaction/delegation.ts#L192-L206)
- 승인 전 가스비 견적: [요청 시점](https://github.com/MetaMask/core/blob/0bd8c26105066911fb73465c95c39ceea1206ee4/packages/transaction-controller/src/TransactionController.ts#L1301-L1338)과 [요청 데이터](https://github.com/MetaMask/core/blob/0bd8c26105066911fb73465c95c39ceea1206ee4/packages/transaction-controller/src/utils/gas-fee-tokens.ts#L70-L123)
- [Ethereum 거래 생명주기](https://ethereum.org/developers/docs/transactions/#transaction-lifecycle)
- [분석 기준 소스](https://github.com/MetaMask/metamask-extension/tree/0997172ccbed2dfd31c56cf070b244d44cfb42ed) — MetaMask Extension 13.56.0
