# Product Backend Project Deep-Dive Interview Guide

An anonymized, Japanese-first preparation guide for a product-engineering/backend interview. Questions and answers are grouped by project so that architecture, difficult decisions, reliability, ownership, and outcomes form one coherent story.

> **Accuracy rule:** `要確認` marks details that must be replaced with the real implementation before the interview. Never present a sensible design pattern as past experience unless it actually happened.

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

**要確認:** 実際のservice数とdatabase ownership、決済の同期・非同期境界、transactional outboxの有無、物理アクセス更新の許容遅延。

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

**要確認:** 実際のSaga方式、具体的なcompensation、idempotency keyの保存方法。

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

### Q1.8 — What was your personal contribution and outcome?

**要約**

私はarchitecture提案だけでなく、critical pathの実装、delivery、production operation、team leadershipまで担当しました。

**詳しい説明**

私は要件と既存依存を整理し、service boundaryとdata ownershipを提案し、Goのapplication/domain層、PostgreSQL transaction、非同期worker、integration codeを実装しました。IaC、GitOps deployment、dashboard、alert、incident investigationまで担当しました。5名のチームではarchitecture review、実装計画、code review、mentoringも行いました。

機能面では、100以上の予約対象と複数カテゴリを同じplatformで扱い、決済、本人認証、物理アクセス、データ活用までつながるworkflowを提供しました。Delivery automationではCI/CD全体を約30%短縮し、alert改善ではnoiseを大きく削減しました。

**具体例**

例えば新しいintegrationを追加するとき、私はdomain/API contractを設計し、Go codeとtestを実装し、Kubernetes/Istio設定とdashboardを追加し、release後のtraceまで確認しました。同時にteam memberのdesign reviewを行い、supportが障害状態を判断できるrunbookも整えました。

**要確認:** Architecture migration自体の測定結果。CI/CDやalertの成果と混同しない。

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

The following is a verified reference flow. Before presenting it as personal experience, align object names and ordering with the API version actually used in the project.

```text
Actors: Buyer/Browser | Merchant Frontend | Merchant Backend | Amazon Pay

1. Buyer -> Merchant Frontend: choose Amazon Pay at checkout
2. Merchant Backend -> Frontend: signed Checkout Session payload
3. Frontend -> Amazon Pay: render button and start Checkout Session
4. Amazon Pay -> Buyer: hosted page for address/payment-instrument selection
5. Amazon Pay -> Merchant review URL: redirect with checkoutSessionId
6. Frontend -> Merchant Backend: load checkout review
7. Merchant Backend -> Amazon Pay API: get/update Checkout Session
   - final amount
   - paymentIntent (for example, AuthorizeWithCapture)
   - checkoutResultReturnUrl
8. Merchant Backend -> Browser: redirect to amazonPayRedirectUrl
9. Amazon Pay -> Buyer: process the selected payment
10. Amazon Pay -> Merchant result URL: redirect with checkoutSessionId
11. Merchant Backend -> Amazon Pay API: Complete Checkout Session
    - same amount
    - idempotency key on POST
12. Amazon Pay -> Merchant Backend: ChargeId / ChargePermissionId or error
13. Merchant Backend -> Internal DB: persist provider IDs and final state
14. Merchant Backend -> Buyer: show confirmed, failed, or recovery-pending result
15. Webhook/polling/reconciliation -> Backend: converge later state changes
```

The browser redirect proves that the buyer returned; the backend API result is what should finalize the internal payment decision. The provider documents `CheckoutSession`, `ChargePermission`, `Charge`, and `Refund` as distinct objects. Depending on `paymentIntent`, completion may authorize and capture immediately, authorize only, or only confirm permission.

**OAuth 2.0 analogy and boundary**

The browser-redirect shape resembles an OAuth 2.0 Authorization Code flow: the user interacts on an external trusted page and returns with a short-lived reference that the application backend validates or exchanges. However, the semantics are different. OAuth returns an authorization code that is exchanged for tokens; this payment flow returns a checkout-session reference, and the merchant backend completes a payment operation. PKCE protects the OAuth code exchange with a `code_challenge` and `code_verifier`; it is not the mechanism that completes this payment session.

**Official references:** [Add the Amazon Pay button](https://developer.amazon.com/docs/amazon-pay-checkout/add-the-amazon-pay-button.html), [Set payment info](https://developer.amazon.com/docs/amazon-pay-checkout/v1-set-payment-info.html), [Verify and complete checkout](https://developer.amazon.com/docs/amazon-pay-checkout/verify-and-complete-checkout.html), [API object model](https://developer.amazon.com/docs/amazon-pay-api-v2/v1-introduction.html)

**要確認:** 実際のauthorize/capture順序、webhook/polling、idempotency mechanism。

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

**要確認:** 実際に使ったkey、unique constraint、provider機能。

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

**要確認:** 実際のreconciliation頻度、matching key、manual operation、data retention。

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

**要確認:** 実際のPCI scope、tokenization、key management、成果指標。

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

**要確認:** 実際のfreshness targetとpercentile。数字が確定するまでは「準リアルタイム」とだけ説明する。

### Q3.3 — How do you handle duplicates and late data?

**要約**

Stable IDとversionを使うidempotent mergeにより、duplicate、late arrival、out-of-order updateを安全に処理します。

**詳しい説明**

Pipelineはat-least-once deliveryや再実行を前提にします。Eventにはstable event ID、business key、event time、schema versionを持たせ、sink側でdeduplicationまたはidempotent mergeを行います。Late dataはevent timeとwatermarkで扱い、期限後に到着したrecordを捨てるか、過去partitionを更新するかをconsumer requirementで決めます。

Orderingが重要なentityではversionまたはsequenceを比較し、古いupdateが新しいstateを上書きしないようにします。

**具体例**

同じ`payment-captured` eventがconsumer restart後に二回届いても、`event_id`でdeduplicateし、売上を二重加算しません。またversion 12の`CANCELED`更新後に遅れてversion 11の`CONFIRMED`が届いても、merge条件を`incoming.version > current.version`にして巻き戻しを防ぎます。

**要確認:** 実際にDelta Lake MERGE、checkpoint、watermark、sequenceを使ったか。

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

**要確認:** Raw retention、checkpoint reset、backfill isolation、consumer切替方法。

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

**要確認:** Dashboard利用者、report作成時間、freshness改善、manual work削減などの実測値。

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

**要確認:** 実際のdeduplication table、conditional update、visibility timeout、DLQ運用。

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

**要確認:** 実際のrollout unit、comparison method、rollback design。

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

**要確認:** 実際の同期方式。この回答は標準設計であり、過去実装の事実として未確定。

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

**要確認:** 実際にdual write、CDC、application event、shadow readのどれを使ったか。

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
| Failure or lesson learned | `要確認: choose one real incident` | Architecture evolution |

## Facts to confirm before rehearsal

- [ ] Exact reservation-system topology and service boundaries
- [ ] Real synchronous/asynchronous boundaries for payment and access control
- [ ] Actual Saga, compensation, idempotency, outbox, and reconciliation mechanisms
- [ ] One concrete incident: symptom, impact, diagnosis, action, result, prevention
- [ ] Exact SLO definitions and measurements
- [ ] Payment authorize/capture, callback, unknown-outcome, and reconciliation flow
- [ ] Data-pipeline freshness, volume, deduplication, replay, and backfill mechanisms
- [ ] Order-worker deduplication, DLQ, rollout, and rollback mechanics
- [ ] Search-index synchronization mechanism and whether CQRS was intentional
- [ ] Exact NoSQL-to-SQL cutover sequence and validation evidence
- [ ] One rejected alternative and one lesson for each of Projects 1–5
