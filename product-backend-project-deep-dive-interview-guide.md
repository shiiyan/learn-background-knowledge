# Product Backend Project Deep-Dive Interview Guide

An anonymized, Japanese-first preparation guide for a product-engineering/backend interview. Questions and answers are grouped by project so that architecture, difficult decisions, reliability, ownership, and outcomes form one coherent story.

> **Accuracy rule:** `確認済みの外部仕様` means the behavior is supported by official product documentation. `推奨設計案` means the design is a strong proposed answer, not proof of the historical implementation. Only describe a proposed design in the past tense after personal confirmation.

## Quick navigation

| Unit | Project | Main interview signals |
|---:|---|---|
| 0 | [Opening narrative](#unit-0-opening-narrative) | Career summary, project selection |
| 1 | [Distributed reservation and access platform](#project-1-distributed-reservation-and-access-platform) | Architecture, consistency, reliability, ownership |
| 2 | [Multi-provider payment integration](#project-2-multi-provider-payment-integration) | Idempotency, unknown outcomes, reconciliation |
| 3 | [Near-real-time operational data pipeline](#project-3-near-real-time-operational-data-pipeline) | Correctness, replay, schema evolution |
| 4 | [Order and shipment processing modernization](#project-4-order-and-shipment-processing-modernization) | Async processing, scaling, incremental migration |
| 5 | [Marketplace search and zero-downtime data migration](#project-5-marketplace-search-and-zero-downtime-data-migration) | Search/read models, migration judgment |
| 6 | [Recommendation-backed product integration scenario](#project-6-recommendation-backed-product-integration-scenario) | ML integration, fallback, experimentation |

## How to answer a project deep dive

Use three layers instead of giving the five-minute version immediately:

1. **30–60 seconds:** product, problem, personal ownership, result.
2. **Two minutes:** architecture, hardest constraint, decision, trade-off.
3. **Deep dive:** consistency, failure modes, alternatives, operation, and lessons.

Within each question, use the same three-part format:

- **要約:** answer the question directly in one or two sentences.
- **詳しい説明:** explain the reasoning, trade-offs, and implementation.
- **具体例:** show actors, requests, state changes, failure handling, or measurable evidence.

Inside the detailed explanation, use this order:

> **Problem → Constraints → Options → Decision → Execution → Outcome → Lesson**

Always distinguish what **I** decided or implemented, what **we** delivered as a team, what another team owned, and what is a production fact versus a later design idea.

---

## Unit 0: Opening narrative

### Q0.1 — Tell me about yourself

**質問:** 自己紹介をお願いします。

**English:** Tell me about yourself.

**要約**

私は、分散workflow、決済、データ基盤を中心に、要件定義から本番運用まで担当してきたproduct-oriented backend engineerです。

**詳しい説明**

私は、モビリティ、Eコマース、マーケットプレイス領域で、6年以上バックエンドとプロダクト開発に携わってきました。強みは、APIを実装するだけではなく、要件整理、アーキテクチャ設計、実装、リリース、本番運用まで一貫してオーナーシップを持つことです。

現在の代表的な経験は、Go、PostgreSQL、Kafka、Kubernetesを使った予約プラットフォームです。予約状態を決済、本人認証、物理アクセス、データ基盤と連携させるため、部分障害、冪等性、補償処理、可観測性を重視して設計しました。それ以前には、10種類以上の決済手段との連携、注文・出荷処理の非同期化、検索基盤、NoSQLからSQLへの無停止移行も経験しました。

私は、複雑なシステムを作ること自体ではなく、プロダクト上の課題に対して、保守可能で信頼できる仕組みを選び、運用まで改善し続けることを大切にしています。

**具体例**

例えば現在の予約platformでは、Go APIだけでなく、決済、本人認証、物理アクセス、data pipelineとの境界まで設計しました。別のcommerce projectでは10種類以上の決済手段と障害復旧を担当しました。この二つを挙げると、「広く触った」ではなく、end-to-end ownershipの共通軸が伝わります。

### Q0.2 — What is your most challenging project?

**要約**

最も難しかったのは、独立した複数systemの状態を、一つの安全な予約体験として成立させるprojectです。

**詳しい説明**

最も難しかったのは、予約、決済、本人認証、物理アクセスを一つの体験として成立させる分散予約プラットフォームです。各システムは独立しており、単一のデータベーストランザクションでは守れません。それでも、支払い済みなのに予約がない、予約が有効なのに入場できない、といった状態を放置できません。私はサービス境界、状態遷移、非同期処理、補償、監視、手動復旧まで含めた設計と実装を主導しました。

**具体例**

例えば、決済providerが成功した直後にnetwork timeoutが起きると、予約serviceからは結果が分かりません。このとき即座に失敗として再課金するのではなく、`PAYMENT_UNKNOWN`のような状態で保持し、照会やreconciliationで確定させる必要があります。この「結果不明」を安全に扱うことが、単純なCRUDとの違いです。

---

## Project 1: Distributed reservation and access platform

### Project card

| Item | Summary |
|---|---|
| Product | Five categories and more than 100 reservable resources |
| Scale | More than 1,000 reservation workflows annually |
| Core stack | Go, PostgreSQL, Kafka, Kubernetes, service mesh, GitOps |
| Integrations | Payment, OIDC identity, physical access, analytics/data |
| Personal scope | Requirements, architecture, implementation, testing, delivery, operation; technical leadership in a five-engineer team |
| Reliability | Successful-request and user-facing latency SLOs; confirm exact definitions before quoting figures |

### Conceptual flow

This is a discussion map, not a claim that every box is a separate production service:

```text
Client
  -> authenticated reservation API
  -> reservation transaction and source-of-truth database
  -> synchronous response for the user-visible decision
  -> events / asynchronous workers
       -> payment state
       -> physical-access authorization
       -> operational and analytical data
  -> monitoring, tracing, reconciliation, and recovery
```

### Q1.1 — Briefly introduce the project

**質問:** このプロジェクトについて簡単に説明してください。

**English:** Please give me a brief overview of the project.

**要約**

100以上の予約対象を、決済、identity、物理アクセス、analyticsと連携させる分散予約platformです。

**詳しい説明**

このプロジェクトは、会議室、イベント、設備など、5カテゴリ・100以上の予約対象を扱うプラットフォームです。ユーザーにとっては予約操作ですが、バックエンドでは本人認証、決済、物理アクセス権、分析データまで連携する必要があります。

私はバックエンド側で、要件整理からGoサービスの設計・実装、PostgreSQLのデータモデル、Kafkaを使う非同期フロー、Kubernetesへのデプロイ、本番監視まで担当しました。特に、サービス境界をまたぐ状態整合性と、部分障害から安全に回復できる設計に重点を置きました。

**具体例**

会議室予約なら、空き確認とreservation record作成だけでは完了ではありません。認証された利用者か確認し、必要なら決済を完了し、予約時間に有効な入場権限を連携し、運用・分析用eventも届けます。一つの「予約する」操作の裏に、複数systemとのcontractがあります。

### Q1.2 — Describe the architecture

**要約**

予約databaseを業務上のsource of truthとし、domain内はlocal transaction、外部連携は明示的な同期APIまたは非同期eventで接続します。

**詳しい説明**

中心には予約ドメインがあり、予約の状態と業務ルールを管理します。ユーザー操作は認証されたAPIとして受け、予約ドメイン内の整合性はPostgreSQLトランザクションで守ります。一方、決済、物理アクセス、分析基盤は別システムなので、すべてを一つのトランザクションには含めません。

外部境界では、同期処理が必要な判断と、非同期でよい更新を分けました。ユーザーが待つ必要がある結果は同期的に返し、後続連携はイベントやワーカーで処理します。失敗時は状態を残し、retry、compensation、reconciliationのどれで回復するかを業務ルールごとに決めます。運用時にはmetrics、logs、tracesを相関させ、どの境界で失敗したか追跡します。

**具体例（conceptual sequence）**

```text
User/Client -> Reservation API: create reservation
Reservation API -> Identity service: validate user/context
Reservation API -> PostgreSQL: check rule and create PENDING record
Reservation API -> Payment boundary: start/complete user payment when required
Payment boundary -> Reservation API: confirmed / failed / unknown
Reservation API -> PostgreSQL: move to CONFIRMED or recovery state
Reservation API -> Event broker: ReservationConfirmed
Workers -> Access system: provision time-bounded access
Workers -> Notification system: send confirmation
Workers -> Data platform: publish operational event
```

The interview should use the real state names and ordering. This sequence only demonstrates the level of detail expected.

**推奨設計案 — service boundary、data ownership、sync/async**

- **Ownership:** Reservation serviceだけがreservation tableをwriteする。Identity、payment、access、analyticsはそれぞれのteam/systemが自分のstoreを所有し、他serviceのtableへ直接writeしない。
- **Synchronous path:** Authenticate user、availability/invariant check、temporary hold、user-interactive payment completion、final reservation decision。Userが次画面へ進むために必要な結果だけを含める。
- **Asynchronous path:** Access provisioning、email/push、analytics、partner reporting。失敗してもreservation factを失わず、retry/reconciliationできる処理を置く。
- **Outbox:** Reservation state変更と`outbox_event` insertを同じPostgreSQL transactionでcommitする。RelayがKafkaへpublishし、publish済みmarkまたはoffsetを進める。Consumerはduplicateを前提にidempotentにする。
- **Access SLO proposal:** 通常はconfirmed eventの99.9%を60秒以内に反映し、予約開始時刻をhard deadlineとする。直前予約ではaccess確認を完了条件にするか、明示的な`ACCESS_PENDING`状態とsupport escalationを用意する。

**本人確認:** 実際のservice数、outbox採用有無、payment/accessの本当の順序、許容遅延。Outboxを使っていない場合は、同じ目的を持つintegration-job tableやreconciliation processを説明する。

### Q1.3 — What made it technically difficult?

**要約**

最も難しかったのはpartial failureです。一つのuser actionが、共通のatomic transactionを持てない複数systemをまたいでいました。

**詳しい説明**

難しさは、一つのユーザー操作が独立した複数システムへ影響することです。ネットワーク越しでは、一方が成功し他方がtimeoutになる部分障害が起こります。また、timeoutは失敗を意味せず、相手では成功している「結果不明」の状態もあります。

そのため、happy pathだけでなく、各stepの状態遷移、冪等性、retry可能性、補償可能性、最終確認方法を設計しました。重要なのは、分散環境で瞬間的にすべてを一致させることではなく、不整合を検出可能・説明可能・回復可能にすることでした。

**具体例**

予約DBへのcommit後、access-control APIがtimeoutしたとします。予約自体を消して終わりにすると、すでに成功した他の連携と矛盾する可能性があります。そこでaccess provisioningを`PENDING`として残し、idempotent retryし、期限までに解決しなければalertとmanual recoveryへ送ります。

### Q1.4 — What options did you consider?

**要約**

共有DBの延命、全面的なmicroservice分割、domain境界ごとの段階移行を比較し、deliveryを止めない段階移行を選びました。

**詳しい説明**

第一は既存の共有DBとtransaction scriptを拡張する方法で、短期的には速い一方、所有権と変更影響がさらに曖昧になります。第二は最初から細かいmicroservicesへ分割する方法ですが、distributed transactionと運用コストが急増します。第三は、業務境界を明確にしながら段階的に分離する方法です。

私たちは第三の方法を選びました。既存挙動を先に理解し、予約の重要なinvariantをドメイン内へ集め、外部連携を明示的なportとして分離しました。これにより、全面刷新せずfeature deliveryを続けながら、変更範囲とdata ownershipを明確化できました。

**具体例**

例えば、最初から「payment service」「access service」へ細分化するのではなく、既存codeから予約可否と状態遷移をdomain layerへ集めます。その後、外部payment callをinterfaceの後ろへ分離し、contract testを追加してからservice boundaryを移します。各段階で既存API behaviorを保つためrollback可能です。

### Q1.5 — How did you maintain consistency across services?

**要約**

Local transaction、idempotent state transition、Saga/compensation、reconciliationを組み合わせ、最終的に正しい業務状態へ収束させます。

**詳しい説明**

一つのdatabase内で守れるinvariantはlocal transactionで守り、serviceをまたぐ処理はstate machineとして扱います。各stepにはbusiness keyを持たせ、同じ要求が再送されても二重処理にならないようにします。途中で失敗した場合は、単純retryでよい処理、反対操作によるcompensationが必要な処理、人の確認が必要な処理を分けます。

外部処理がtimeoutした場合はすぐに失敗と確定せず、結果不明の状態として保存し、照会またはreconciliationで確定します。Exactly-once deliveryを前提にせず、at-least-onceでもeffectively-onceの業務結果になるようconsumerとstate transitionを設計します。

**具体例**

`ReservationConfirmed` eventが二回配信されても、access workerは`reservation_id + access_window`をoperation keyとして扱います。一回目で権限作成済みなら二回目はno-opにします。逆に権限作成が失敗したら同じkeyでretryし、二重権限を作らずに収束させます。

**推奨設計案 — orchestrated Sagaとcompensation**

User-facingでfinancial consequenceがあるため、workflow stateを一か所で追跡できるorchestrationを第一候補にする。

```text
CREATE_PENDING_RESERVATION
  -> HOLD_RESOURCE
  -> AUTHORIZE_OR_CAPTURE_PAYMENT
  -> CONFIRM_RESERVATION
  -> PROVISION_ACCESS (async, deadline-bound)

Compensation:
payment fails        -> release resource hold
confirmation fails   -> void authorization or initiate refund
access fails         -> retry -> alert -> manual grant/cancel policy
user cancels         -> revoke access -> refund according to policy
```

`workflow_step(workflow_id, step_name, attempt_no, state, provider_reference, last_error)`を保存し、`UNIQUE(workflow_id, step_name)`またはoperation-specific keyでduplicate実行を防ぐ。External side effectには同じstable keyを渡し、compensation自体もidempotentにする。

**本人確認:** 実際にはorchestrationかchoreographyか、resource holdの有無、void/refund rule、table/key名。

### Q1.6 — Which flows should be synchronous or asynchronous?

**要約**

ユーザーが次へ進むために必要な判断は同期、後から再実行できる副作用は非同期にします。ただし一つのworkflow内でも両方を組み合わせます。

**詳しい説明**

判断基準は、ユーザーがその場で結果を必要とするか、後続処理を遅延できるか、相手の障害をrequest pathへ伝播させてよいかです。予約可能性の確認や受付結果は同期処理が中心です。一方、分析データの配送などは非同期化し、user requestのlatencyと外部障害の影響を分離します。

ただし、非同期化は複雑さを消すのではなく移動させます。Queue lag、duplicate、ordering、poison message、replay、observabilityを設計する必要があります。

**具体例**

- **Synchronous:** ユーザーが外部決済画面で承認し、結果が確認できなければ予約を確定できないcheckout部分。Clientをproviderへredirectし、return後にbackendがpayment sessionをcompleteしてから予約結果を表示します。
- **Asynchronous:** 予約確認email、push notification、analytics event。失敗しても予約の事実は変わらず、queueから後でretryできます。
- **Hybrid:** Payment authorizationはuser interactionを含む同期flowでも、webhook、reconciliation、refund完了通知は非同期です。Sagaは「すべて同期」という意味ではなく、複数stepとcompensationをcoordinationする考え方です。

### Q1.7 — What were the non-functional requirements?

**要約**

Availabilityとlatencyだけでなく、予約・決済・入場権限のcorrectness、recoverability、auditabilityが重要でした。

**詳しい説明**

重要だったのはavailabilityだけでなく、correctness、latency、recoverability、auditabilityです。予約と物理アクセスが連携するため、APIが200を返すだけではユーザー体験が成功したとは言えません。Request successとlatencyに加え、連携遅延、失敗step、未解決状態、reconciliation backlogも監視対象にします。

SLOは、対象endpoint、measurement window、除外条件、percentile、error budgetとセットで説明します。高い数字だけでなく、どのユーザー行動を守る指標かを示します。

**具体例**

API success rateが高くても、`CONFIRMED`予約のaccess provisioningが10分遅れれば利用者は入場できません。そのため、HTTP error rateに加え、`confirmed reservation with access_pending > threshold`の件数と最古ageをSLIとして監視します。

**推奨SLO定義（実際のtargetへ合わせる）**

| Concern | Proposed SLI definition | Example target |
|---|---|---|
| Request success | Eligible user requests that finish with the correct non-5xx outcome / all eligible requests | 99.99% or the documented historical target over 28/30 days |
| Query latency | End-to-end latency measured at the API boundary for successful user-facing queries | p99 under 2 seconds; quote p99.99 only if dashboards and traffic volume support it |
| Reservation correctness | Confirmed reservations with no unresolved payment/access contradiction | 99.99%+, with every contradiction reconciled |
| Access readiness | Confirmed reservations provisioned before their access deadline | 99.9% within 60 seconds and 100% before start, if realistic |
| Recovery | Unknown/integration-pending workflows resolved automatically within a bounded time | 99% within 15 minutes; page on oldest-age breach |

Exclude health checks, load tests, and invalid client requests only through a documented rule; do not remove dependency failures merely because another team owns the dependency. Measure at the user-facing boundary, then break down internal causes separately.

If quoting `99.999% request success`, explain that it is request-based: at one million eligible requests, the error budget is about ten failed requests. It is not automatically equivalent to seconds of downtime. If quoting p99.99, confirm there is enough traffic for a meaningful percentile and describe the measurement window.

### Q1.8 — What was your personal contribution and outcome?

**要約**

私はarchitecture提案だけでなく、critical pathの実装、delivery、production operation、team leadershipまで担当しました。

**詳しい説明**

私は要件と既存依存を整理し、service boundaryとdata ownershipを提案し、Goのapplication/domain層、PostgreSQL transaction、非同期worker、integration codeを実装しました。IaC、GitOps deployment、dashboard、alert、incident investigationまで担当しました。5名のチームではarchitecture review、実装計画、code review、mentoringも行いました。

機能面では、100以上の予約対象と複数カテゴリを同じplatformで扱い、決済、本人認証、物理アクセス、データ活用までつながるworkflowを提供しました。Delivery automationではCI/CD全体を約30%短縮し、alert改善ではnoiseを大きく削減しました。

**具体例**

例えば新しいintegrationを追加するとき、私はdomain/API contractを設計し、Go codeとtestを実装し、Kubernetes/Istio設定とdashboardを追加し、release後のtraceまで確認しました。同時にteam memberのdesign reviewを行い、supportが障害状態を判断できるrunbookも整えました。

**推奨する効果測定 — architecture migration**

Migrationの前後8–12週間など同程度のwindowで、次を比較する。

- Change lead time: requirement readyからproductionまで
- Deployment frequency and change-failure rate
- Mean time to restore after integration failure
- 一つのfeature変更で触るmodule/service数
- Shared-tableへのdirect access数とcircular dependency数
- Regression incident、rollback、support escalationの件数
- Onboarding後に最初のproduction changeを出すまでの日数

**Interview-safe wording:** 「CI/CD 30%短縮とalert noise削減は確認できた別の成果です。Architecture migration自体については、`[実測した指標]`で評価しました」と分ける。実測値がなければ「測るべきだった改善点」として話し、数字を作らない。

### Q1.9 — What would you change today?

**要約**

Integration stateをproduct teamやsupportも理解できる形で、初期段階から可視化します。

**詳しい説明**

今なら、業務stateとintegration stateの可視化をより早く標準化します。失敗状態がlogにあっても、product teamやsupportが理解できる形でなければ復旧に時間がかかります。State transition、retry回数、最終同期時刻を共通dashboardとoperational APIで示すことで、復旧時間と問い合わせ対応を短縮できます。

**具体例**

Support画面で `reservation=CONFIRMED / payment=CAPTURED / access=PENDING / last_retry=10:32` と確認できれば、利用者へ状況を説明し、必要なstepだけ再実行できます。Engineerが複数logを手作業でcorrelateする時間も減ります。

---

## Project 2: Multi-provider payment integration

### Project card

| Item | Summary |
|---|---|
| Context | Commerce orders with more than ten payment methods |
| Examples | Credit card and external wallet integrations |
| Personal scope | Integration delivery, incident response, reconciliation, recovery-flow design |
| Main invariant | No duplicate charge; internal and provider states remain explainable and recoverable |

### State model to explain

```text
CREATED -> PROCESSING -> SUCCEEDED
                      -> FAILED
                      -> UNKNOWN -> reconciliation -> SUCCEEDED / FAILED
SUCCEEDED -> REFUND_PENDING -> REFUNDED
```

Use the real production state names. `UNKNOWN` is the key concept: a timeout does not prove that the provider failed.

### Q2.1 — Walk me through one payment integration

**要約**

Payment integrationは、user redirect、provider-side authorization、merchant backendでのcompletion、internal order state更新を一つのrecoverable workflowとして設計します。

**詳しい説明**

このプロジェクトでは、複数種類の注文を扱うcommerce platformで、クレジットカードや外部walletを含む10種類以上の決済手段との連携を担当しました。責任はprovider APIを呼ぶことだけではなく、order stateとpayment stateの境界、失敗時の復旧、社内記録とprovider記録の照合まで設計することでした。

基本flowは、注文に対応するpayment attemptを内部に記録し、一意なbusiness keyを使ってproviderへ要求を送り、同期responseまたは非同期notificationで状態を更新します。Responseを受け取れない場合はfailedへ即決せず、unknownとして照会・reconciliation対象にします。

**具体例 — Amazon Pay Checkout v2 reference flow**

Before presenting this as personal experience, align the API version and payment intent with the project that was actually implemented.

**Memory line:** `Create → buyer selects → Update → buyer approves → Complete → save/reconcile`

```text
Actors: Buyer/Browser | Merchant Backend | Amazon Pay

1. Create checkout
   Buyer clicks Amazon Pay. The signed button payload starts a Checkout Session,
   and Amazon Pay shows its hosted address/payment-selection page.
   API equivalent: POST /v2/checkoutSessions

2. Set the final payment
   Amazon Pay returns checkoutSessionId to the merchant review page.
   Backend sets amount, paymentIntent, and result URL, then receives
   amazonPayRedirectUrl.
   API: PATCH /v2/checkoutSessions/{checkoutSessionId}

3. Buyer approves
   Browser follows amazonPayRedirectUrl. Buyer confirms or completes MFA.
   Amazon Pay redirects to checkoutResultReturnUrl with checkoutSessionId.

4. Complete checkout
   Backend verifies the amount and finalizes the payment.
   API: POST /v2/checkoutSessions/{checkoutSessionId}/complete
   Header: x-amz-pay-idempotency-key
   Result: ChargeId, ChargePermissionId, or an error/pending state

5. Save and converge
   Backend stores provider IDs and the internal payment/order state.
   Pending changes are resolved by IPN or polling:
   GET /v2/charges/{chargeId}
```

For a Japan Checkout v2 integration using an environment-prefixed endpoint, the API base is:

```text
https://pay-api.amazon.jp/{environment}/v2

POST  /checkoutSessions
PATCH /checkoutSessions/{checkoutSessionId}
POST  /checkoutSessions/{checkoutSessionId}/complete
GET   /charges/{chargeId}
```

The browser redirect proves that the buyer returned; the backend API result is what should finalize the internal payment decision. The provider documents `CheckoutSession`, `ChargePermission`, `Charge`, and `Refund` as distinct objects. Depending on `paymentIntent`, completion may authorize and capture immediately, authorize only, or only confirm permission.

**OAuth 2.0 analogy and boundary**

The browser-redirect shape resembles an OAuth 2.0 Authorization Code flow: the user interacts on an external trusted page and returns with a short-lived reference that the application backend validates or exchanges. However, the semantics are different. OAuth returns an authorization code that is exchanged for tokens; this payment flow returns a checkout-session reference, and the merchant backend completes a payment operation. PKCE protects the OAuth code exchange with a `code_challenge` and `code_verifier`; it is not the mechanism that completes this payment session.

**Official references:** [Add the Amazon Pay button](https://developer.amazon.com/docs/amazon-pay-checkout/add-the-amazon-pay-button.html), [Set payment info](https://developer.amazon.com/docs/amazon-pay-checkout/v1-set-payment-info.html), [Verify and complete checkout](https://developer.amazon.com/docs/amazon-pay-checkout/verify-and-complete-checkout.html), [API object model](https://developer.amazon.com/docs/amazon-pay-api-v2/v1-introduction.html)

**確認済みの外部仕様と推奨integration policy**

- **Immediate payment:** 商品・予約をその場で確定し、取消riskが低い場合は`AuthorizeWithCapture`を選べる。Complete Checkout Sessionの成功responseから`ChargeId`と必要な`ChargePermissionId`を保存する。
- **Deferred capture:** 在庫・出荷・最終予約確定を後で行う場合は`Authorize`を使い、確定後にcaptureする。長時間のauthorization保持、expiry、void/cancel policyを定義する。
- **Permission only:** 将来の請求同意だけを得るuse caseでは`Confirm`を使い、`ChargePermissionId`を保存して後続Chargeを作る。
- **Asynchronous state:** Pending authorization、遅いcapture、refundなどはIPNまたはGET API pollingでfinal stateを取得する。IPNは通知に含まれるobject IDを手がかりにGET APIを呼び、通知payloadだけでfinal stateを決めない。
- **Polling fallback:** IPN遅延・欠損に備え、`PROCESSING/UNKNOWN` objectをperiodic pollingする。Official guideの一例はhourly pollingだが、checkout UXではbusiness deadlineに合わせて短いintervalとexponential backoffを使う。
- **Idempotency:** Resourceを作るPOSTにはstable `x-amz-pay-idempotency-key`を設定する。同じoperationのretryではbodyとkeyを変えない。
- **Retention:** Checkout Sessionと関連情報はprovider側で30日後に削除されるため、refund/reconciliation/auditに必要なCharge/Permission ID、merchant reference、amount、currency、state、timestampsをinternal storeへ保持する。

**推奨する選択:** Physical goodsならauthorize at order confirmation / capture at shipment、即時提供するdigital serviceや確定予約ならauthorize-with-captureを第一候補にし、cancel/refund policyで最終決定する。

**Official references:** [Verify and complete checkout](https://developer.amazon.com/docs/amazon-pay-checkout/verify-and-complete-checkout.html), [Asynchronous processing](https://developer.amazon.com/docs/amazon-pay-checkout/v1-asynchronous-processing.html), [Instant Payment Notifications](https://developer.amazon.com/docs/amazon-pay-checkout/v1-set-up-instant-payment-notifications.html)

**本人確認:** 過去projectの`paymentIntent`、capture timing、IPN/polling、unknown-state recoveryが実際にどう実装されていたか。

### Q2.2 — What was the hardest technical problem?

**要約**

最も難しいのは「結果不明」です。私たちのserviceがtimeoutを受けても、provider側では決済が成功している可能性があります。

**詳しい説明**

最も難しいのは、内部transactionと外部provider operationをatomicにcommitできないことです。Provider側でchargeが成功した直後にnetwork timeoutが起きると、こちらは結果を受け取れません。そこで安易にretryするとduplicate chargeの危険があります。

Payment attemptとrequest identityを永続化し、結果不明を明示的なstateとして扱い、provider照会や後続notificationで確定できるようにします。定期reconciliationでは内部とproviderの記録を比較し、自動復旧できない差分を調査可能にします。

**具体例**

`Complete Checkout Session`を送信した直後にconnectionが切れた場合、clientへは成功responseが届きません。しかしproviderではChargeが作成済みかもしれません。Internal stateを`FAILED`へせず`UNKNOWN`にし、同じcheckout sessionとidempotency keyで安全に確認します。確認できるまで新しいchargeを開始しません。

### Q2.3 — How do you prevent duplicate charges?

**要約**

Stable business identity、internal uniqueness、provider idempotency、state checkの多層防御で二重課金を防ぎます。

**詳しい説明**

防御は複数層にします。同じbusiness operationには安定したidempotency keyを使い、内部では一意制約またはstate checkで同じattemptの重複実行を防ぎます。Providerがidempotencyを提供する場合は同じkeyを渡します。Workerはduplicate deliveryを前提に、現在stateを確認して安全にno-opできるようにします。

結果が不明なら、新しいchargeを作る前に既存requestを照会します。Retry policyは回数よりも、どのfailure classなら安全にretryできるかを定義することが重要です。

**具体例**

`order-123`のcapture operationに毎回同じidempotency keyを使います。Userがdouble-clickして二つのHTTP requestが来ても、internal unique constraintは一つのpayment attemptだけを作ります。ProviderへのPOSTも同じkeyなので、network retryが二回目のchargeを作りません。Amazon Pay APIではPOST requestに`x-amz-pay-idempotency-key`が必要です。

**確認済みの外部仕様と推奨internal constraint**

Amazon Payはresource-creating requestのidempotencyを提供する。Official guidanceではUUID v4を推奨し、最大32文字、英数字・dash・underscoreを許可する。通常のhyphen付きUUID文字列は36文字なので、32文字のUUID v4 hex表現など、制限内の衝突しにくい値を生成する。同じkeyの最初のresponseは保存され、payloadを変えて再利用すると`DuplicateIdempotencyKey`になる。

```text
payment_attempt
  id                    UUID PRIMARY KEY
  order_id              UUID NOT NULL
  attempt_no            INT NOT NULL
  operation             AUTHORIZE | CAPTURE | REFUND
  idempotency_key       VARCHAR(32) UNIQUE NOT NULL
  provider_object_id    VARCHAR(...) UNIQUE NULL
  amount_minor          BIGINT NOT NULL
  currency              CHAR(3) NOT NULL
  state                 ...
  request_hash          CHAR(64) NOT NULL
  UNIQUE(order_id, attempt_no, operation)
```

- Same logical operation: reuse the same key and identical request body.
- New intentional attempt after a final decline: allocate a new `attempt_no` and key.
- Concurrent request: insert the attempt first; the unique constraint elects one caller as owner.
- Retry after timeout: read the stored attempt and reuse its key; never generate a fresh key inside retry code.

**Official reference:** [Amazon Pay idempotency](https://developer.amazon.com/docs/amazon-pay-api-v2/idempotency.html)

**本人確認:** 実際のkey format、table/constraint、request-hashの有無。

### Q2.4 — What if the database commits but the provider times out?

**要約**

Commit済みのinternal attemptをrecoveryの起点にし、provider結果が判明するまでterminal failureへ進めません。

**詳しい説明**

内部recordがcommit済みなら、それをrecoveryの起点にできます。Provider callがtimeoutした時点では成功・失敗を確定せず、processingまたはunknownとして残します。その後、同じrequest identityで安全に照会またはretryし、provider側の事実に基づいてfinal stateへ進めます。

逆にprovider成功後に内部更新が失敗した場合も、webhook、polling、scheduled reconciliationなど、providerの事実を再取得する経路が必要です。一回のrequest-responseではなく、最終的に収束するprotocolとして設計します。

**具体例**

Databaseに`attempt_id=42, order_id=123, state=PROCESSING`をcommitした後、provider callがtimeoutしたとします。APIは「決済失敗」ではなく「確認中」を返します。Recovery workerがprovider referenceを照会し、成功なら同じtransaction内でattemptを`SUCCEEDED`、orderを`PAID`へ進めます。Declinedなら`FAILED`、まだ不明ならretry scheduleを更新します。

### Q2.5 — How does reconciliation work?

**要約**

Internal ledgerとprovider recordを定期比較し、差分を分類して自動修復またはmanual investigationへ送ります。

**詳しい説明**

期間やpayment methodの範囲で内部transactionとprovider recordを比較し、missing、amount mismatch、status mismatch、duplicateなどに分類します。差分ごとに、自動修正、安全なretry、refund、manual investigationのrunbookを分けます。

差分件数だけでなく、最古の未解決時間、金額影響、payment method別error rateを監視します。修正操作自体にもaudit logとidempotencyが必要です。

**具体例**

前日のcaptured transactionsについて、`merchant_reference_id`、provider charge ID、amount、currency、statusをjoinします。Providerは`CAPTURED`だがinternalは`PROCESSING`ならinternal stateを安全に修正します。Amount mismatchやprovider側だけに存在するchargeは自動変更せず、金額影響付きのcaseとして担当者へ送ります。

**推奨設計案 — reconciliation**

| Layer | Frequency | Purpose |
|---|---:|---|
| Hot recovery | Every 1–5 minutes | Resolve checkout attempts in `PROCESSING/UNKNOWN` before user/support impact grows |
| Daily transaction reconciliation | Daily after provider reporting closes | Compare Charge/Refund state, amount, currency, and merchant reference |
| Settlement reconciliation | Per settlement period | Compare captured/refunded totals, fees, adjustments, and actual disbursement |
| Long-tail audit | Weekly/monthly | Detect old unresolved items, chargebacks, and manual corrections |

Matching priority:

1. Provider `ChargeId` or `RefundId`
2. Stored merchant reference / order ID
3. Amount + currency + bounded timestamp only as investigation support, never as the sole automatic-match key

Differences become typed cases: `PROVIDER_ONLY`, `INTERNAL_ONLY`, `STATUS_MISMATCH`, `AMOUNT_MISMATCH`, `DUPLICATE`, `STALE_PENDING`. Safe status updates may be automated; money-moving corrections require approval, audit log, and an idempotent command. Store report ID, source row, before/after states, operator, reason, and provider response.

Retention must follow legal/accounting/privacy policy. At minimum, keep identifiers and financial audit fields long enough for refund, dispute, chargeback, settlement, and statutory windows; delete unnecessary buyer profile data earlier. Provider-side 30-day availability is not a substitute for internal retention.

**Official references:** [Settlement report fields](https://developer.amazon.com/docs/amazon-pay-reports/settlement-reports.html), [Report use for reconciliation](https://developer.amazon.com/docs/amazon-pay-checkout/set-up-reports.html)

**本人確認:** Actual reporting schedule, legal retention period, approval workflow, and automatic-repair scope.

### Q2.6 — How do provider differences affect the design?

**要約**

共通payment lifecycleはdomainに置き、provider固有object、status、署名、callbackはadapterへ閉じ込めます。

**詳しい説明**

各providerはauthorize/capture、callback、timeout、error code、refund、idempotency supportが異なります。そこで共通domain modelとprovider adapterを分けます。ただし、最小公倍数に抽象化しすぎると重要なcapabilityを失うため、共通lifecycleとprovider-specific metadataの両方を持たせます。

Domain側は「支払いが確定したか」「取消可能か」という業務語彙で判断し、provider固有のstatus mappingはadapter境界へ閉じ込めます。

**具体例**

Domain commandは`AuthorizePayment`や`RefundPayment`ですが、あるprovider adapterは`CheckoutSession -> ChargePermission -> Charge`へ変換し、別のcard adapterは`PaymentIntent`相当のobjectを扱うかもしれません。Providerの`Authorized`、`Captured`、`Declined`をinternal `AUTHORIZED`、`PAID`、`FAILED`へmappingし、raw provider statusとreferenceも調査用に保持します。

### Q2.7 — Security, outcome, and lesson

**要約**

Payment flowではdata minimization、signed server-to-server calls、least privilege、auditabilityを守り、成功率だけでなく不整合と復旧時間で成果を測ります。

**詳しい説明**

機密なpayment dataは必要最小限だけ扱い、tokenized referenceを利用します。Secretはcentral managementで管理し、logへcard dataやcredentialを出しません。Refundやmanual correctionにはauthorizationとaudit trailを設けます。

成果は、10種類以上の決済手段を注文flowへ統合し、障害時にも差分を検出・調査・復旧できる運用を整えたことです。最大の学びは、payment integrationは成功responseを処理する機能ではなく、不明な結果と長期間向き合うstate machineだということです。

**具体例**

Browserからprovider secretを送らず、merchant backendがprivate keyでAPI requestを署名します。Internal DBにはcard numberではなくprovider token/referenceと必要なstatusだけを保存します。Manual refundを行う場合は、operator、reason、before/after state、provider responseをaudit logへ残します。

**確認済みの外部仕様と推奨security design**

**PCI scope and tokenization**

- In the hosted checkout flow, the buyer selects the payment instrument on the provider-hosted page. The merchant backend should not receive or store PAN/CVV.
- Store provider references (`CheckoutSessionId` temporarily, `ChargePermissionId`, `ChargeId`, `RefundId`), merchant reference, amount, currency, timestamps, and normalized state—not raw card data.
- For a separate direct-card integration, use the PSP's hosted fields or client-side tokenization so raw PAN does not traverse the application server.
- Do not claim a specific PCI SAQ level from architecture alone. Confirm it with the security/compliance owner, acquirer, or QSA because scripts, hosting model, operational access, and other payment methods affect scope.

**Key management**

- Keep the asymmetric private key only in a managed secret store or HSM/KMS-backed secret path; never ship it to the browser, repository, container image, log, or analytics system.
- Give the payment workload a least-privilege runtime identity; restrict human read access and audit every secret access.
- Load the key in memory only for request signing; redact signatures, buyer details, and provider payloads from logs.
- Rotate by creating/uploading a new key, deploying support for the new Public Key ID, verifying traffic, then revoking the old key. Alert on unexpected signature failures and stale keys.
- Separate sandbox and production configuration and access even when a provider technically permits credential reuse.

Amazon Pay documents asymmetric request signing, secure storage of the private key, and a key-rotation strategy. It does not publish the merchant's final PCI classification.

**Recommended success metrics**

- Checkout completion and authorization/capture success by payment method
- Duplicate charge count: target zero
- `UNKNOWN/PENDING` count, oldest age, and percentage resolved automatically
- Internal/provider mismatch count and financial amount at risk
- P95/P99 provider latency and timeout rate
- Refund completion time and failure rate
- Payment-related support contacts and mean time to resolution

**Official references:** [Hosted buyer experience](https://developer.amazon.com/docs/amazon-pay-checkout/introduction.html), [Signing requests](https://developer.amazon.com/docs/amazon-pay-api-v2/v1-signing-requests.html), [Key creation and secure storage](https://developer.amazon.com/docs/amazon-pay-api-v2/manually-generating-key-pairs.html), [API security and rotation guidance](https://developer.amazon.com/docs/amazon-pay-api-v2/introduction.html)

**本人確認:** Direct-card integration architecture, actual secret store/KMS, rotation cadence, PCI assessment, and real metric values.

---

## Project 3: Near-real-time operational data pipeline

### Project card

| Item | Summary |
|---|---|
| Sources | Reservation and payment data |
| Platform | Databricks-based near-real-time pipeline |
| Consumers | Operational dashboards, product analysis, user insights, authorized partner reporting |
| Main risks | Duplicate, late, missing, out-of-order, and schema-incompatible data |

### Conceptual flow

```text
Transactional sources
  -> change/event ingestion
  -> durable raw layer
  -> validated and normalized records
  -> business aggregates / serving tables
  -> dashboards, analysis, partner reports

Control plane:
schema checks + freshness + completeness + replay + reconciliation + access control
```

### Q3.1 — What problem did the pipeline solve?

**要約**

分散した予約・決済dataを一つのtrusted flowへ統合し、運用・分析・partner reportingで再利用できるようにしました。

**詳しい説明**

予約と決済の情報はtransactional systemに分かれており、運用状況、product usage、user behavior、partner reportを一貫して把握することが難しい状態でした。そこで、両方のdataを準リアルタイムで統合し、用途ごとに再利用できるpipelineを構築しました。

私が重視したのは速度だけではありません。意思決定やpartner reportingに使うため、freshness、completeness、correctness、access controlを明示的なrequirementとして扱いました。

**具体例**

運用担当者が「本日の予約数」と「支払済み予約数」を別systemから手作業で集計していたとします。Pipelineでreservation IDをcanonical keyとして統合すれば、`CONFIRMED but payment not settled`の件数をdashboardで継続監視でき、分析担当者も同じcurated datasetを利用できます。

### Q3.2 — What does “near real time” mean?

**要約**

Near real timeは曖昧な表現ではなく、source eventからconsumer-visible dataまでのfreshness SLOとして定義します。

**詳しい説明**

「Near real time」は技術名ではなくSLOとして定義すべきです。例えば、sourceで確定したeventの99%が何分以内にserving tableへ反映されるか、という形です。そのうえでend-to-end lagをingestion、processing、publicationに分けて計測します。

**具体例**

仮に「99%が5分以内」というSLOなら、eventの`occurred_at`とserving tableの`available_at`の差を計測します。P99が8分へ悪化したら、broker lag、stream processing time、table publicationのどこで3分増えたか分解して調査します。

**推奨設計案 — freshness SLO**

Consumerごとにfreshnessを分ける。すべてを最も厳しいSLOへ合わせない。

| Consumer | Proposed SLO | Failure action |
|---|---|---|
| Operational reservation/payment dashboard | 99% within 5 minutes; no item older than 15 minutes | Page on sustained breach; show data-last-updated time |
| Product analytics | 99% within 15 minutes | Ticket/alert; preserve correctness rather than drop late data |
| Authorized partner report | Complete by contracted delivery deadline | Block publication if completeness/reconciliation check fails |

Measure `available_at - source_committed_at`, not job runtime alone. Publish p50/p95/p99, max age, input lag, processing lag, and publication lag. A dashboard must display its last successful refresh so consumers do not treat stale data as current.

**Interview-safe wording:** If these were not historical targets, say「私ならconsumer impactからこのようにSLOを設定します」rather than「5分SLOでした」。

**本人確認:** Actual consumer deadlines, event volume, and historical p99.

### Q3.3 — How do you handle duplicates and late data?

**要約**

Stable IDとversionを使うidempotent mergeにより、duplicate、late arrival、out-of-order updateを安全に処理します。

**詳しい説明**

Pipelineはat-least-once deliveryや再実行を前提にします。Eventにはstable event ID、business key、event time、schema versionを持たせ、sink側でdeduplicationまたはidempotent mergeを行います。Late dataはevent timeとwatermarkで扱い、期限後に到着したrecordを捨てるか、過去partitionを更新するかをconsumer requirementで決めます。

Orderingが重要なentityではversionまたはsequenceを比較し、古いupdateが新しいstateを上書きしないようにします。

**具体例**

同じ`payment-captured` eventがconsumer restart後に二回届いても、`event_id`でdeduplicateし、売上を二重加算しません。またversion 12の`CANCELED`更新後に遅れてversion 11の`CONFIRMED`が届いても、merge条件を`incoming.version > current.version`にして巻き戻しを防ぎます。

**推奨設計案 — Delta/Structured Streaming implementation**

```text
Bronze: append raw event + event_id + event_time + ingest_time + source offset
Silver: validate schema, quarantine invalid rows, deduplicate by event_id
Gold: MERGE latest entity version and build business aggregates
```

- Use an explicit durable checkpoint location per query. It tracks offsets, committed micro-batches, and state needed after restart.
- Use `foreachBatch` + `MERGE` for upsert, and make the MERGE idempotent because a restarted stream can apply a micro-batch again.
- Deduplicate by stable `event_id`; use entity `version/sequence` in the MERGE condition so older events cannot overwrite newer state.
- Use a watermark to bound state for late-data handling, but do not use it as permission to silently discard financially important events. Send beyond-watermark records to correction/reconciliation flow when completeness matters.
- Monitor backlog bytes/files, batch duration, input rows, processed rows, dropped/quarantined rows, watermark delay, and checkpoint failure.

Databricks documents checkpoint recovery, stateful deduplication, idempotent `MERGE` in `foreachBatch`, and the need to handle schema changes carefully.

**Official references:** [Structured Streaming checkpoints](https://docs.databricks.com/aws/en/structured-streaming/checkpoints), [Delta streaming reads and writes](https://docs.databricks.com/aws/en/structured-streaming/delta-lake), [Schema evolution](https://docs.databricks.com/aws/en/data-engineering/schema-evolution)

**本人確認:** Actual Bronze/Silver/Gold layout, checkpoint storage, MERGE keys, watermark, and late-event policy.

### Q3.4 — How do you verify correctness?

**要約**

Freshnessとdata correctnessを別々に測り、source-to-target reconciliationで黙った欠損や誤集計を検出します。

**詳しい説明**

Record countだけでは不十分なので、sourceとのreconciliationを複数levelで行います。件数、business key、status別集計、金額合計、更新時刻を比較し、差分をdrill downできるようにします。各stageのinput、output、quarantine countを記録し、schemaやbusiness ruleに違反したrecordを黙って捨てません。

Dashboardが更新されていても内容が誤っていれば障害なので、freshness SLIとquality SLIを分けます。

**具体例**

日次でtransactional sourceとcurated tableの`reservation_count`、`captured_amount`、currency別合計を比較します。件数が一致しても金額が違えば、join duplicationやcurrency conversionのbugを疑います。Invalid recordはquarantine tableへ送り、理由と再処理状況を可視化します。

### Q3.5 — How do replay and backfill work?

**要約**

Raw inputとversioned transformを保持し、修正後のlogicを特定範囲へidempotentに再適用できるようにします。

**詳しい説明**

Replay可能にするには、raw inputをdurableかつimmutableに近い形で保持し、transformをdeterministicかつversionedにします。修正したjobを特定期間へ再実行し、idempotent writeまたは新しいtable versionへ出力してから切り替えます。通常処理とbackfillが競合しないようpartition、version、write policyを決めます。

**具体例**

Tax fieldのmapping bugが8月1日から3日まで存在した場合、raw eventsは変更せず、transform v2をその三日分へ実行します。新しいpartition/tableへ書き、row countとamount aggregateを検証してからconsumer viewを切り替えます。Streaming jobが同じpartitionへ同時writeしないよう範囲を隔離します。

**推奨設計案 — retention, checkpoint, and backfill**

- **Raw retention:** Choose a replay window from correction/chargeback/reporting needs and privacy policy. A practical starting point is 90 days online plus cheaper archival if policy permits, but do not state this as the historical setting without evidence.
- **Checkpoint:** Never delete/reset a production checkpoint for an ordinary redeploy. Resume from it when query/state schema is compatible. If an incompatible source, stateful operator, or state schema change requires a new checkpoint, start a controlled new query and define its starting version/time explicitly.
- **Backfill isolation:** Run backfill with a separate job identity, checkpoint, compute quota, and output staging table. Rate-limit it so it cannot starve the live stream.
- **Idempotent output:** MERGE by business key + version or write a new immutable table version. Record `backfill_run_id`, code version, source range, row counts, and validation result.
- **Cutover:** Compare source/old target/new target counts and aggregates, then atomically switch a view/table alias. Keep the old target for a rollback window.
- **Concurrency:** Freeze overlapping partitions briefly or use version-aware MERGE so live events win correctly while backfill runs.

**Concrete recovery example:** A mapping bug affected August 1–3. Deploy transform v2, replay immutable raw records for that range into `gold_v2`, reconcile counts/amounts, pause only publication, switch the consumer view, and retain `gold_v1` until the rollback window expires.

**本人確認:** Actual retention, whether checkpoint reset was ever required, table/view cutover mechanism, and backfill resource isolation.

### Q3.6 — How do you evolve schemas?

**要約**

Additive and backward-compatible changesを基本とし、producerとconsumerを独立して安全にdeployできるようにします。

**詳しい説明**

Producerとconsumerを同時deployできるとは限らないため、基本はadditive changeを使います。Schema versionとcompatibility ruleを持ち、新fieldをoptionalとして追加し、consumer移行後にold fieldを廃止します。Breaking changeは新topicまたは新datasetとして並行稼働し、quality comparison後に切り替えます。

**具体例**

`payment_method`を追加するとき、まずoptional fieldとしてschema registryへ登録し、old consumerが無視できることを確認します。全consumerがnew fieldへ対応した後でproducerの送信を必須化します。既存`amount`の型変更が必要なら、`amount_minor_units`を新fieldとして追加し、即時置換を避けます。

### Q3.7 — Why this platform, and what alternatives existed?

**要約**

Streaming、batch、replay、analytics、governanceをまとめて必要としたためdata platformを選び、より小さい構成とも比較しました。

**詳しい説明**

選定理由は、streaming処理だけでなく、履歴dataに対するbatch、replay、analytics、governanceを同じdata platformで扱えることです。一方、単純なmessage relayだけならKafka consumerとdatabaseの方が小さく、低latency servingならonline storeが必要です。既存ecosystem、team skill、運用cost、vendor couplingも比較します。

**具体例**

「予約eventを一つのdashboardへ送るだけ」なら、Kafka consumerからPostgreSQLへ書く方が単純です。しかし複数年の履歴を再集計し、schema evolutionとpartner別access controlを持ち、analystがSQLで探索するなら、lakehouse型platformの利点が大きくなります。

### Q3.8 — What was the outcome?

**要約**

予約と決済を横断する共通datasetを作り、複数の運用・product・reporting use caseを支えました。

**詳しい説明**

予約と決済を横断する共通data flowを作り、運用dashboard、product analysis、user-activity insight、authorized partner reportを支えました。「Pipelineを作った」で終わらず、誰がどの判断に使ったかを説明します。

**具体例**

運用teamは未解決の予約・決済差分をdaily dashboardで確認し、product teamは予約category別利用傾向を分析し、許可されたpartnerには自分の対象dataだけをreportとして提供できます。同じbusiness definitionを共有することで、部署ごとの数字のずれを減らします。

**推奨するoutcome evidence**

Collect one baseline and one after-value for each relevant consumer:

- Operations: time to find unresolved reservation/payment mismatch; manual spreadsheet steps; incident detection delay
- Product: time from question to available dataset; number of reusable curated metrics; conflicting metric definitions
- Partner reporting: preparation hours, late reports, correction count, unauthorized-access incidents
- Platform: p99 freshness, completeness percentage, failed job/replay frequency, cost per processed event/GB

**Detailed answer template:**

> Before the pipeline, `[consumer]` needed `[manual process/time]` and data was delayed by `[baseline]`. After release, `[dataset/dashboard]` refreshed at `[actual p99]`, reduced `[manual work/error]` by `[actual result]`, and enabled `[decision or operational action]`. I owned `[specific design/code/operation]`.

If no numerical baseline exists, use auditable evidence such as eliminating a named manual handoff, supporting a new partner report, or reducing investigation steps. Do not invent percentages.

**本人確認:** Real users, baseline, after-value, and one decision made from the data.

---

## Project 4: Order and shipment processing modernization

### Project card

| Item | Summary |
|---|---|
| Domain | Seven order types connected to shipment, delivery, and payments |
| Architecture | Clearer order-domain boundaries and backward-compatible API contracts |
| Processing | Database-scanning batch to fine-grained messages and parallel workers |
| Result | Less database load and tenant head-of-line blocking; most small tenants completed within minutes |

### Before and after

```text
Before:
one scheduler -> scan shared table -> process many tenants sequentially

After:
scheduler/producer -> bounded queue of fine-grained jobs
                   -> parallel idempotent workers
                   -> retry / dead-letter / operational visibility
```

### Q4.1 — What problem were you solving?

**要約**

一つのlarge batchが全tenantを直列に処理する構造を、failure-isolatedで公平なjob processingへ変えました。

**詳しい説明**

出荷処理が一つのdatabase scan型batchに集中しており、大きなtenantの処理が後続tenantを待たせるhead-of-line blockingが起きていました。処理量が増えるとscanと更新がdatabaseへ負荷をかけ、失敗時のretry単位も大きくなります。

目標はworker数を増やすことではなく、tenant間の影響を分離し、retryを小さくし、progressとfailureを観測できる構造にすることでした。

**具体例**

Tenant Aが10万件、Tenant Bが100件の出荷を持つ場合、直列batchではBの100件がAの完了を待ちます。Orderまたはshipment単位のmessageへ分割すれば、worker poolがinterleaveして処理でき、Bは数分以内に完了できます。

### Q4.2 — Why a queue and parallel workers?

**要約**

Queueでjob creationとexecutionを分離し、小さいfailure unit、controlled parallelism、visible backlogを得ました。

**詳しい説明**

Queueによりjob生成と実行をdecoupleし、fine-grained messageを複数workerで処理できます。小さなtenantが大きなtenantを待つ必要がなくなり、失敗したunitだけretryできます。Queue depthとoldest-message ageからbacklogも直接観測できます。

Trade-offとして、duplicate delivery、ordering、visibility timeout、poison message、backpressureを設計する必要があります。Queue導入だけではcorrectnessは保証されないため、workerをidempotentにします。

**具体例**

Workerがshipment label作成後、message acknowledge前にcrashすると同じmessageが再配信されます。Workerはshipment IDを調べ、label作成済みならproviderを再度呼ばず成功としてackします。繰り返し失敗するmessageはDLQへ移し、他のjobを止めません。

### Q4.3 — How did you make workers idempotent?

**要約**

Stable job identityとconditional state transitionにより、同じmessageを何度処理しても業務結果が一回になるようにします。

**詳しい説明**

Messageにはstable job identityと対象entityを持たせ、処理前にcurrent stateを確認します。Database updateにはexpected stateまたはunique constraintを使い、完了済みjobは安全にno-opします。外部side effectがある場合は、そのoperation固有のidempotency mechanismも必要です。

**具体例**

`UPDATE shipments SET state='REQUESTED' WHERE id=? AND state='READY'`のようなconditional updateを使います。Affected rowが0なら、別workerが処理済みかinvalid stateなので再実行しません。External APIには`shipment_id`由来のidempotency keyを渡します。

**確認済みのqueue behaviorと推奨worker design**

Standard SQS is at-least-once: the same message can be delivered more than once. Visibility timeout only hides a received message temporarily; if the worker does not delete it before timeout, it becomes visible again. Therefore, correctness belongs in the consumer.

```text
processed_job
  job_id          VARCHAR PRIMARY KEY
  entity_id       VARCHAR NOT NULL
  operation       VARCHAR NOT NULL
  state           PROCESSING | SUCCEEDED | FAILED
  owner_token     UUID
  result_ref      VARCHAR NULL
  updated_at      TIMESTAMP
```

- Insert/claim `job_id` with a unique constraint, or use a conditional business-state update. `SUCCEEDED` means duplicate delivery can return success without repeating the side effect.
- Set visibility timeout above normal processing time—for example p99 processing time plus a safety margin. For variable long work, heartbeat with `ChangeMessageVisibility`; do not set an extremely long timeout that delays recovery.
- Delete the message only after durable business state is committed.
- Configure bounded retry and a DLQ. A proposed starting policy is `maxReceiveCount=5`, then tune from failure data.
- Alarm on oldest-message age, visible/in-flight count, DLQ depth, receive count, processing latency, and success/failure by operation.
- Redrive from DLQ only after fixing the cause; preserve original message ID and attach a redrive audit record.

**Official references:** [SQS at-least-once delivery](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html), [Visibility timeout and DLQ guidance](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)

**本人確認:** Actual queue type, timeout, max receive count, DLQ alarm/redrive, and deduplication implementation.

### Q4.4 — How did you control load and fairness?

**要約**

Bounded concurrencyとtenant-aware schedulingで、throughputを上げてもdatabaseと小規模tenantを守ります。

**詳しい説明**

Parallelismを無制限に上げるとdatabaseやdownstreamを壊すため、worker concurrency、connection pool、queue inflight数に上限を設けます。Tenantごとのjob sizeが異なる場合は、message粒度、partitioning、per-tenant concurrencyでfairnessを調整します。

AutoscalingにはCPUだけでなくqueue depth、oldest age、processing latencyを使い、downstream capacityを超えない上限を設定します。

**具体例**

Database connection poolが50ならworker concurrencyを200へしても性能は上がらず、待ちとtimeoutが増えます。まず40 concurrent jobsに制限し、queue ageが上昇したらreplicaを増やします。ただし一tenantのinflightを5件までに制限し、全workerを占有させません。

### Q4.5 — How did you migrate safely?

**要約**

Old/new processorを比較し、tenantまたはorder type単位で段階移行し、明確なrollback ownershipを持たせます。

**詳しい説明**

既存batchの挙動をtestで固定し、新producerとworkerを小さい範囲から導入します。可能であればshadow processingまたは結果比較を行い、tenantやorder type単位で段階的に移します。Rollback時に二つのprocessorが同じjobを実行しないようownership flagやrouting ruleを明確にします。

**具体例**

最初にinternal test tenantだけを`processor=v2`へrouteし、処理件数、error、duration、final shipment stateをold batchと比較します。次にsmall tenantの5%、25%、100%へ広げます。Rollbackはflagをv1へ戻しますが、v2 queueに残るmessageをdrainまたはinvalidateする手順も必要です。

**推奨設計案 — staged rollout and rollback**

1. **Characterization:** Capture old batch behavior with production-derived test cases.
2. **Shadow:** Generate v2 decisions but suppress external side effects; compare target entity, payload, and expected final state.
3. **Internal tenant:** Route only test/internal tenant traffic.
4. **Canary:** Deterministic tenant hash or allowlist: 1% → 5% → 25% → 50% → 100%. Keep a tenant entirely on one processor to avoid split ownership.
5. **Gates:** At each stage compare completion count, duplicate side effects, p95/p99 duration, DB query/load, queue age, DLQ count, and support issues.
6. **Rollback:** Stop new v2 enqueue, identify and drain/cancel already queued v2 jobs, then switch routing to v1. Do not let v1 and v2 own the same shipment simultaneously.
7. **Finalize:** After a stable observation window, remove v1 scheduling and retain reconciliation for late differences.

Use a `processing_owner/version` column or routing ledger so ownership is durable rather than only an in-memory feature flag.

**本人確認:** Actual rollout percentages/unit, whether shadow execution was possible, queue-drain procedure, and rollback window.

### Q4.6 — Outcome and lesson

**要約**

Tenant間blockingとdatabase loadを減らし、小規模tenantのcompletion timeとfailure isolationを改善しました。

**詳しい説明**

Database負荷とtenant間のhead-of-line blockingを軽減し、多くの小規模tenantでは処理が数分以内に完了するようになりました。Failureとretryの単位も小さくなり、運用状況を把握しやすくなりました。

学びは、非同期化の価値はthroughputだけでなくfailure isolationとfairnessにあることです。一方、規模が小さいうちは単純batchの方が保守しやすいため、queueは負荷と運用要件が正当化するときに導入すべきです。

**具体例**

従来は一件のslow tenantや一つの失敗でbatch全体の完了が遅れました。移行後は該当messageだけretry/DLQへ分離され、他tenantは処理を継続できます。Outcomeはaverageだけでなく、tenant別p95 completion timeとoldest queue ageで示します。

---

## Project 5: Marketplace search and zero-downtime data migration

### Project card

| Item | Summary |
|---|---|
| Product | Marketplace MVP that grew from zero to 100,000 users in two years |
| Application | Kotlin, Spring Boot, modular monolith, web frontend |
| Capabilities | Authentication, matching, full-text search, real-time messaging |
| Data change | NoSQL to SQL migration with zero downtime |
| Search | Managed search and OpenSearch technologies for full-text discovery |

### Important wording boundary

Do **not** call this CQRS unless the real implementation intentionally separated command and query models. A search index beside a transactional database creates a separate read model, but that alone does not prove a full CQRS architecture.

### Q5.1 — Why a modular monolith for the MVP?

**要約**

小規模teamの高速なproduct validationを優先し、deploymentは一つでもdomain boundaryが明確なmodular monolithを選びました。

**詳しい説明**

初期productではtraffic予測よりも、少人数で仮説検証とfeature deliveryを速く回すことが重要でした。そのため、network boundaryを増やすmicroservicesではなく、authentication、matching、messagingなどのdomain boundaryをcode上で分けたmodular monolithを選びました。

同一processとtransactionを活かしながらmodule間dependencyを明示し、必要になればservice分離できる余地を残しました。System sizeではなくteam size、change pattern、operational maturityに合わせた判断です。

**具体例**

`identity`、`job`、`matching`、`messaging`をpackage/moduleとして分け、他moduleのtableへ直接queryせずapplication interfaceを通します。最初から四serviceにするとdeployment、network failure、distributed tracingが必要ですが、一processなら少人数でMVPを速く変更できます。

### Q5.2 — Why a search engine instead of only SQL?

**要約**

Transactional correctnessはSQLに残し、full-text relevanceとdiscoveryに特化した再構築可能なsearch read modelを追加しました。

**詳しい説明**

Marketplace discoveryでは、全文検索、relevance、複数条件filter、typo tolerance、rankingが重要です。SQL indexでも単純条件は扱えますが、検索品質を継続的に調整する要件にはsearch engineが適しています。

Search indexをsource of truthにはしません。業務上の正しいrecordはtransactional storeが持ち、searchは再構築可能なread modelとして扱います。これによりindex delayや欠損が起きても元dataからrepairできます。

**具体例**

Userが「Go backend remote」と検索したとき、title、description、skill tagへ異なるweightを付け、typoやsynonymも扱います。一方、応募可能か、案件が公開中かという最終判断はtransactional DBから確認し、古いindex結果だけでbusiness actionを許可しません。

### Q5.3 — How do you keep the index synchronized?

**要約**

Databaseをsource of truthにし、versioned and idempotent indexingとreconciliationでsearch read modelを収束させます。

**詳しい説明**

Database changeとindex updateを別々に行うとdual-write failureが起きます。安全な方法はdatabase commitをsource of truthにし、outboxまたはchange streamからindexing eventを配信することです。Consumerはversionを比較してidempotentにupsertし、古いeventが新しいdocumentを上書きしないようにします。

Lag、failure count、source/index countを監視し、定期reindexまたは差分repairを用意します。

**具体例**

Job version 8の`published` event後に、遅れてversion 7の`draft` eventが届いても、indexerはdocument versionを比較してversion 7を無視します。Indexingが一時間停止した場合はoutbox offsetから再開し、必要ならDB snapshotから新indexを作ってaliasを切り替えます。

**推奨設計案 — search synchronization and repair**

```text
PostgreSQL transaction:
  update job(version=8) + insert outbox(job_id, version=8)
        |
        v
Outbox relay -> event broker -> indexer -> OpenSearch jobs_v3
                                         -> alias: jobs_read
```

- PostgreSQL remains the source of truth. Avoid an application-level DB + index dual write because either side can succeed alone.
- Insert an outbox row in the same transaction as the domain change. Publish at least once; indexer upsert/delete must be idempotent.
- Use domain entity ID as document ID and database version as external version. Ignore an event whose version is older than the indexed document.
- Measure outbox age, consumer lag, indexing failure, source/index count by partition, and sampled field checksum.
- Keep a repair job that reads changed DB rows or compares versions and reindexes missing/stale documents.
- For mapping/analyzer changes, build `jobs_v4`, backfill it, validate search quality/counts, then atomically move stable alias `jobs_read` from v3 to v4. Keep v3 for rollback.
- Search results are candidates. Recheck critical mutable business conditions—published status, authorization, availability—against the source of truth before a write action.

OpenSearch supports external versioning and aliases that can switch between indexes without application downtime.

**Official references:** [Index document external versioning](https://docs.opensearch.org/latest/api-reference/document-apis/index-document/), [Index aliases](https://docs.opensearch.org/latest/im-plugin/index-alias/), [Reindex API](https://docs.opensearch.org/latest/api-reference/document-apis/reindex/)

**本人確認:** Actual source of truth, event/outbox/CDC mechanism, document version, repair job, and alias use. Call it CQRS only if command/query models were intentionally separated beyond merely adding a search index.

### Q5.4 — Why move from NoSQL to SQL?

**要約**

Product成長でrelationとtransactional invariantが増えたため、application側で整合性を再実装するよりSQLが適切になりました。

**詳しい説明**

初期段階ではschema flexibilityと開発速度の面でNoSQLが有効でした。しかしproductが成長すると、entity間relation、transaction、constraint、複雑なqueryが増え、application側で整合性を管理するcostが高くなりました。そこで、業務invariantをdatabase constraintとtransactionで明確に表現できるSQL modelへ移行しました。

これはNoSQLが悪いのではなく、product requirementが変化し、access patternとconsistency requirementにSQLが適合したという判断です。

**具体例**

初期はuser document内にprofileをまとめると高速に開発できます。しかしjob application、company membership、permissionが増えると、同じuser情報が複数documentへ重複し、atomic updateが難しくなります。SQLならforeign key、unique constraint、transactionで「同じjobへ一回だけ応募できる」といったruleを表現できます。

### Q5.5 — How do you perform a zero-downtime migration?

**要約**

Backfillと継続更新を両立させ、shadow comparison、段階cutover、rollback windowを通じて無停止でsource of truthを移します。

**Migration steps（実際の順序へ修正）**

1. Target schema and invariant design
2. Historical backfill with checkpoints
3. Capture changes during backfill
4. Shadow read and source/target comparison
5. Gradual read cutover
6. Rollback window
7. Stop old writes and retire the old model

**詳しい説明**

無停止移行ではbulk copyだけでは不十分です。Backfill中にも更新が続くため、初期copyとincremental change captureを組み合わせます。Target writeをidempotentにし、件数だけでなくbusiness key、重要field、relation、aggregateでsourceと比較します。

Read pathは一部trafficから段階的に切り替え、差分やerror rateを監視します。問題があればold sourceへ戻せる期間を確保し、十分な検証後にold writeを停止します。

**具体例**

User ID 1から100,000までcheckpoint付きでbackfillします。その間のprofile updateはchange eventまたはtemporary dual writeでSQL側にも適用します。Readの1%をSQLへ送り、response fieldとerrorをshadow compareします。Mismatchがthreshold以下になってからtrafficを増やし、旧storeをread-onlyで保持してrollback可能にします。

**推奨設計案 — zero-downtime NoSQL-to-SQL migration**

Prefer a durable change log/CDC over uncoordinated dual writes. If CDC is unavailable, use an application outbox; use direct dual write only with explicit failure recording and repair.

```text
Phase 1  Create SQL schema, constraints, mapping/version table
Phase 2  Start change capture from NoSQL -> durable migration log
Phase 3  Backfill snapshot in key ranges with checkpoints
Phase 4  Replay changes after each range's snapshot position
Phase 5  Shadow-read and compare normalized responses
Phase 6  Canary SQL reads: internal -> 1% -> 10% -> 50% -> 100%
Phase 7  Make SQL authoritative; keep old store read-only for rollback
Phase 8  Reconcile, close rollback window, delete data by retention policy
```

Validation must cover record counts plus business invariants: unique email/member keys, required relations, nullability, status mapping, orphan records, and representative API response equality. Record each migrated key range with `snapshot_position`, `last_change_position`, counts, checksum, and status so retries are idempotent.

During canary, route one user/entity consistently to one read source. Writes continue through the authoritative old path plus durable change capture until cutover. If mismatch exceeds threshold, stop canary and rebuild affected ranges; do not reverse-copy uncertain data automatically.

**本人確認:** Actual source database capability, capture mechanism, backfill key/range, comparison tooling, cutover percentages, and rollback duration.

### Q5.6 — What was the outcome?

**要約**

MVPの0から100,000 usersへの成長を支えながら、検索体験と長期的に保守しやすいrelational data modelを実現しました。

**詳しい説明**

Productは二年間で0から100,000 usersへ成長しました。検索とmessagingを含むMVPをend to endでdeliveryし、data modelはservice continuityを保ったままNoSQLからSQLへ移行しました。User growth全体を自分だけの成果とは言わず、自分のarchitectureとdeliveryがgrowthを支えた、と表現します。

**具体例**

面接では「私が100,000 usersを獲得した」ではなく、「teamのproduct growthに対し、私はsearch、backend、infrastructure、zero-downtime migrationを担当し、trafficを止めずにfeature開発を継続できる基盤を作った」とownershipを正確に表現します。

---

## Project 6: Recommendation-backed product integration scenario

> **This is a system-design scenario, not a claim of previous production ownership.** It shows how the experience above transfers to a content product that consumes ML or ranking capabilities.

### Scenario and architecture

A product backend requests ranked content from an ML/recommendation platform, applies product and policy rules, and returns a reliable feed. The product team owns serving integration and user experience; the ML team owns model training and scoring.

```text
Client
  -> Product Feed API
       -> identity / consent / experiment assignment
       -> candidate or ranking API
       -> product rules, filtering, deduplication
       -> hydration from content store/cache
       -> response assembly

Fallback: cached personalized -> segment ranking -> global/trending
Feedback: impression/click events -> durable pipeline -> analytics / ML
```

### Q6.1 — How would you define the ML-service boundary?

**要約**

ML serviceはrankingを、product backendはpolicyとuser-visible correctnessを所有する明確なcontractにします。

**詳しい説明**

Product backendとML serviceのcontractをmodel内部ではなくproduct-level input/outputで定義します。Requestにはuser/context、candidate constraint、deadline、experiment metadataを含め、responseにはranked content IDs、model version、必要なdebug metadataを含めます。

Product backendはauthorization、policy filtering、content availability、deduplication、response assemblyを所有し、ML serviceはrankingを所有します。どちらがuser-visible correctnessを守るかを曖昧にしません。

**具体例**

Feed APIが`user_context, locale, surface, max_items, deadline, experiment_id`をranking APIへ送り、`content_ids, scores, model_version`を受け取ります。返されたIDの一つが削除済みならproduct backendが除外し、必要数を補充します。Model serviceが削除policyまで所有しているとは仮定しません。

### Q6.2 — What if the recommendation service is slow?

**要約**

短いdependency deadline、failure isolation、段階的fallbackで、ranking障害をfeed全体の障害にしません。

**詳しい説明**

End-to-end latency budgetからdependency deadlineを決め、短いtimeoutを設定します。Retryは残りbudgetとidempotencyを考慮し、同一request pathで無制限に行いません。Circuit breakerとbulkheadでfailure propagationを抑えます。

Fallbackはcached personalized result、segment-level ranking、global/trending listの順に品質を段階的に下げます。200を返すことだけでなく、fallback率、freshness、empty response、user engagementへの影響を監視します。

**具体例**

Feed全体のbudgetが500msならrankingへ200msを割り当てます。200msでtimeoutしたら同じrequest内で何度もretryせず、直近10分のpersonalized cacheを返します。それもなければlocale別trending listを返し、response metadataとmetricへ`fallback=trending`を記録します。

### Q6.3 — How would you cache results?

**要約**

Resultを変えるcontextをcache keyへ含め、freshness、privacy、cardinality、invalidationをtrade-offします。

**詳しい説明**

Cache keyにはuser/segment、locale、surface、experiment、model versionなど結果を変える要素を含めます。ただしcardinalityとprivacy costを考えます。TTLはfreshnessとdependency loadのtrade-offで決め、popular keyにはrequest coalescingやstale-while-revalidateを使います。

Deleted、expired、policy-blocked contentはcache hit後にもfilterし、invalidation pathを用意します。

**具体例**

`feed:{segment}:{locale}:{surface}:{experiment}:{modelVersion}`をkeyにし、TTLを5分とします。同じpopular segmentへrequestが集中したら一つのrefreshだけを実行し、他requestにはstale resultを短時間返します。緊急削除contentはdenylistでresponse直前にも除外します。

### Q6.4 — How would you roll out a new model?

**要約**

Shadow、canary、A/B testを段階的に使い、quality metricとreliability guardrailの両方で判断します。

**詳しい説明**

Offline metricだけでproduction rolloutを決めません。Model versionをresponseとlogへ残し、shadow trafficでlatency、error、result validityを確認します。その後、小さなcanaryまたはA/B experimentでguardrail metricとproduct metricを比較します。

GuardrailにはAPI error、p95/p99 latency、empty/duplicate result、fallback rate、content complaintを含めます。問題があればmodel versionまたはrouting configを戻せるようにします。

**具体例**

Model v2をまず5%のshadow trafficへ送り、user responseにはv1を使いながらlatencyとinvalid ID rateを比較します。次に1%のusersへv2を返し、click-throughだけでなくp99 latency、complaint、fallback率を確認します。Guardrailを超えたらrouting configをv1へ戻します。

### Q6.5 — What would you monitor?

**要約**

Serving、dependency、data quality、product outcomeの四層をmodel versionとexperimentごとに観測します。

**詳しい説明**

- **Serving:** rate, error, latency, saturation
- **Dependency:** ranking latency/error, timeout, circuit state
- **Data quality:** empty result, duplicate, invalid ID, stale content
- **Product:** impression, click, retention/conversion, complaint and policy guardrails

Model version、experiment、surface、localeで比較しますが、user IDのようなhigh-cardinality metric labelは避けます。

**具体例**

`feed_requests_total{model=v2,result=fallback}`と`ranking_latency_seconds{model=v2}`でsystem behaviorを見ます。Empty resultが増えたらtrace IDでranking responseとfilter結果を調べます。User単位の調査情報はsecure logへ置き、Prometheus labelには入れません。

### Q6.6 — How does your past experience transfer?

**要約**

Model trainingの経験を誇張せず、MLを安全なproduct experienceへ統合するbackend・data・reliability experienceを示します。

**詳しい説明**

私はmodel training自体をproductionで所有したとは主張しません。一方、独立serviceとのAPI contract、timeout、fallback、非同期data flow、schema evolution、SLO、incident responseは、予約、決済、data pipelineで扱ってきました。

Recommendation integrationでも、ML outputをそのまま返すのではなく、product rule、failure isolation、observability、safe rolloutを含むuser-facing systemとして設計できます。Feedback eventのcorrectnessとfreshnessはmodel improvementにも影響するため、transactional systemとdata platformの橋渡し経験を活かせます。

**具体例**

Payment providerとの連携で使ったtimeout、unknown outcome、reconciliationの考え方はranking dependencyのfailure handlingへ応用できます。またProject 3のschema version、late event、replayの経験は、impression/click feedback pipelineのcorrectnessへ直接応用できます。

### Q6.7 — Optional partner/content-data scenario

**質問:** Partnerまたはcampaign dataを受け取り、product surfaceとdownstream data/ML systemへ安全に届けるbackendをどう設計しますか？

**要約**

Canonical identity、versioned contract、idempotent ingestion、audit/replayを中心に、partner dataをproductとdownstream consumerへ安全に届けます。

**詳しい説明**

- Canonical partner/content/campaign identifiers
- Versioned API and event contracts
- Idempotent ingestion and immutable audit records
- Validation and quarantine rather than silent drops
- Late corrections, replay, and attribution changes
- Authorization and data minimization
- Freshness, completeness, and correctness SLOs
- Incremental rollout with reconciliation and rollback

Connect this answer to Project 3 for pipeline correctness and Project 2 for reconciliation, while keeping it clearly hypothetical.

**具体例**

Partnerが同じcampaign correctionを三回送っても、`partner_id + external_campaign_id + version`で一回だけ適用します。Invalid currencyのrecordは捨てずquarantineへ送り、partnerへreasonを返します。Correction後はraw eventを残したままderived attributionを再計算し、変更前後をauditできるようにします。

---

## Cross-project follow-up matrix

| Interviewer probe | Best primary project | Backup project |
|---|---|---|
| Most challenging project | Reservation/access platform | Payment integration |
| Current architecture | Reservation/access platform | Order processing |
| Distributed consistency | Reservation/access platform | Payment integration |
| Idempotency and retries | Payment integration | Order processing |
| Data correctness and replay | Data pipeline | Data migration |
| Architecture alternatives | Reservation evolution | Modular monolith |
| Scaling and backpressure | Order processing | Recommendation scenario |
| Search/read model | Marketplace search | Recommendation caching |
| Zero-downtime change | Data migration | Order-processing rollout |
| Reliability/SLO | Reservation/access platform | Recommendation scenario |
| Product impact | Marketplace MVP | Reservation/data pipeline |
| ML or ranking integration | Recommendation scenario | Data pipeline experience |
| Failure or lesson learned | Reservation/access recovery incident — use the proposed skeleton below only after adapting it to a real event | Architecture evolution |

### Proposed incident-story skeleton

Do not present this as historical fact until it matches a real incident.

| Stage | Detailed example to adapt |
|---|---|
| Situation | Access-system latency increased; reservation events accumulated and some confirmed users approached their start time without provisioned access |
| Detection | User-facing API remained mostly healthy, but `oldest access_pending age` and queue lag crossed the operational deadline |
| Immediate action | Stopped aggressive retries, opened the circuit/bounded concurrency, prioritized reservations nearest to start time, and prepared manual access for imminent users |
| Diagnosis | Traces showed downstream timeout; retry amplification consumed worker/connection capacity and delayed healthy work |
| Fix | Added exponential backoff with jitter, retry budget, per-dependency concurrency limit, idempotent provisioning, and a deadline-aware priority/recovery queue |
| Prevention | Added `confirmed-but-access-pending` SLI, oldest-age alert, runbook, dependency dashboard, and a game-day/load test for slow downstream behavior |
| Outcome | Replace with real affected-user count, recovery time, backlog drain time, and recurrence result |

## Proposed alternatives and lessons by project

| Project | Main alternative rejected | Why the proposed choice is stronger | Lesson to state |
|---|---|---|---|
| Reservation/access | Continue shared-DB transaction scripts or perform a big-bang microservice rewrite | Incremental domain separation preserves delivery and rollback while clarifying ownership | Distributed consistency means detectable and recoverable state, not pretending all services share one transaction |
| Payment integration | Treat provider call as one synchronous request and mark timeout as failure | Persistent attempts, idempotency, unknown state, callbacks/polling, and reconciliation prevent duplicate financial side effects | A payment integration is a long-lived state machine, not an API wrapper |
| Data pipeline | Build one point-to-point export per dashboard/consumer | A governed raw-to-curated pipeline supports replay, shared definitions, quality checks, and multiple consumers | Fresh but wrong data is still an incident; freshness and correctness need separate SLIs |
| Order processing | Increase the size/frequency of the database-scanning batch | Fine-grained queued work gives fairness, bounded concurrency, isolated retries, and visible backlog | Async processing moves complexity into delivery, idempotency, and operation; it does not remove it |
| Search/data migration | Keep all search in SQL and all relational workflow in NoSQL, or switch stores in one event | Specialized read model plus staged migration matches changing access/consistency needs without downtime | Technology choice should change when product access patterns change; no database is universally better |

## Personal confirmation ledger

The architecture questions now have proposed detailed answers. The remaining work is historical verification—information that external documentation and general engineering knowledge cannot establish.

| Personal fact to verify | Proposed answer location | Minimum evidence before using past tense |
|---|---|---|
| Exact reservation topology and ownership | Q1.2 | Real service names/count, database owner, integration contracts |
| Sync/async payment and access sequence | Q1.2 and Q1.6 | Real state diagram or code/API flow |
| Saga, compensation, outbox, and idempotency | Q1.5 | One real workflow ID, step record, compensation, and retry mechanism |
| Production incident | Proposed incident-story skeleton | Actual date/context, impact, action you took, measured recovery and prevention |
| Request and business SLOs | Q1.7 | Dashboard/query definition, window, exclusions, percentile, real target |
| Payment intent and callback/recovery | Q2.1–Q2.5 | Actual API version, intent, provider object IDs, IPN/polling, reconciliation job |
| PCI and secret management | Q2.7 | Security assessment, tokenization boundary, secret store, rotation process |
| Pipeline freshness/dedup/replay | Q3.2–Q3.5 | Job/query configuration, checkpoint, merge key, retention, actual SLO |
| Pipeline impact | Q3.8 | Named consumer, before/after process, one measured outcome |
| Queue worker and rollout | Q4.3–Q4.5 | Queue type, visibility timeout, DLQ/redrive, idempotency record, rollout unit |
| Search synchronization/CQRS | Q5.3 | Actual source, event mechanism, versioning, repair/reindex process; intentional CQRS decision if claimed |
| NoSQL-to-SQL cutover | Q5.5 | Actual backfill/change-capture/read-switch/rollback sequence |

Until a row is verified, introduce it with「この要件なら私はこう設計します」or「改善案としては」rather than「私は実装しました」。
