# Backend Knowledge Quick Reference

A bilingual, category-based reference covering 50 common backend engineering interview questions.

Each entry contains:

- the question in English and Japanese;
- a short core answer suitable for quick review;
- important English keywords;
- one likely follow-up question with its own core answer and keywords.

The questions are based on common backend interview topics, including the [roadmap.sh backend question collection](https://roadmap.sh/questions/backend). The answers here are intentionally concise and may refine or correct simplified explanations from external study material.

## API and HTTP Fundamentals

### 1. API Endpoint

**English question:** What is an API endpoint?

**日本語の質問:** API endpointとは何ですか？

**Core answer:**

Clientが特定のresourceやoperationへアクセスする入口です。通常はHTTP method、path、request/response contractの組み合わせで定義します。例えば、`POST /reservations`は予約を作成するendpointです。

**Keywords:** API endpoint, resource, operation, HTTP method, path, request, response, contract

**Follow-up question:** What should you consider when designing a good API endpoint? / 良いAPI endpointを設計するとき、何を考慮しますか？

**Follow-up core answer:**

Resource naming、HTTP methodとstatus code、input validation、authentication/authorization、idempotency、error format、versioning、observabilityを考慮します。実装方法より先に、clientとのcontractを明確にします。

**Follow-up keywords:** resource naming, HTTP semantics, validation, authorization, idempotency, error model, versioning, observability

---

### 2. RESTful API

**English question:** What is a RESTful API?

**日本語の質問:** RESTful APIとは何ですか？

**Core answer:**

ResourceをURIで表現し、HTTP methodのsemanticsを利用するstatelessなAPI設計です。Uniform interface、cacheability、layered systemなどの制約があります。単にJSONを返すHTTP APIという意味ではありません。

**Keywords:** REST, resource, URI, stateless, uniform interface, cacheability, layered system

**Follow-up question:** How would you choose between REST and RPC or gRPC? / RESTとRPCまたはgRPCをどう使い分けますか？

**Follow-up core answer:**

Publicまたはresource-orientedなAPIではRESTが扱いやすいことが多いです。Internal service間で厳密なschema、code generation、低latency、streamingが重要ならgRPCが有力です。Client compatibility、debuggability、browser support、運用経験も判断材料です。

**Follow-up keywords:** REST, RPC, gRPC, schema, code generation, streaming, compatibility, latency, debuggability

---

### 3. HTTPS Request/Response Cycle

**English question:** Describe a typical HTTPS request/response cycle.

**日本語の質問:** HTTPS requestがclientからserverへ届き、responseが返るまでを説明してください。

**Core answer:**

一般的には、DNS名前解決、connection確立、TLS handshake、HTTP request送信、load balancerやreverse proxy、application処理、databaseやdownstream serviceへのアクセス、HTTP responseという順に進みます。Connectionはkeep-aliveやpoolingによって再利用される場合があります。

**Keywords:** DNS, TCP, QUIC, TLS handshake, HTTP, load balancer, reverse proxy, connection reuse

**Follow-up question:** How would you investigate a slow request? / このrequestが遅い場合、どこから調査しますか？

**Follow-up core answer:**

まず影響範囲とlatency breakdownを確認します。DNS、connection、TLS、load balancer、application、database、downstream dependencyの順に、metrics、logs、distributed tracesを使ってbottleneckを特定します。直近のdeployやtraffic変化も確認します。

**Follow-up keywords:** latency breakdown, metrics, logs, traces, bottleneck, dependency, recent deployment

---

### 4. Idempotency

**English question:** What is idempotency?

**日本語の質問:** Idempotencyとは何ですか？

**Core answer:**

同じoperationを複数回実行しても、最終的なbusiness outcomeが変わらない性質です。HTTP semanticsではGET、PUT、DELETEは原則idempotentですが、POSTは通常idempotentではありません。

**Keywords:** idempotency, duplicate request, retry, business outcome, HTTP semantics

**Follow-up question:** How would you safely retry a reservation or payment POST request? / 予約作成や決済のPOSTを安全にretryするにはどうしますか？

**Follow-up core answer:**

Clientが一意なidempotency keyを送り、server側でkey、request identity、処理状態、resultを保存します。同じkeyの再送では処理を再実行せず、保存済みの結果を返します。Databaseのunique constraintやreconciliationも併用します。

**Follow-up keywords:** idempotency key, request identity, stored result, unique constraint, retry safety, reconciliation

---

### 5. HTTP Statelessness

**English question:** What does statelessness mean in HTTP?

**日本語の質問:** HTTPがstatelessであるとはどういう意味ですか？

**Core answer:**

各requestは独立しており、serverが以前のrequestを覚えていることをprotocolとして前提にしないという意味です。必要な状態はtoken、database、session storeなど、application側で明示的に管理します。

**Keywords:** stateless, independent request, application state, session, token, horizontal scaling

**Follow-up question:** How would you manage sessions in a load-balanced environment? / Load-balanced environmentでsessionをどう管理しますか？

**Follow-up core answer:**

Application instanceをstatelessにし、sessionをRedisなどの共有storeへ保存する方法が一般的です。Sticky sessionも利用できますが、load distribution、failover、scalingが難しくなる場合があります。Token-based方式ではrevocationやtoken sizeなども考慮します。

**Follow-up keywords:** centralized session store, Redis, sticky session, token, failover, horizontal scaling

---

### 6. API Versioning

**English question:** How do you version an API?

**日本語の質問:** API versioningをどのように行いますか？

**Core answer:**

まずbackward-compatibleなadditive changeを優先します。Breaking changeが必要な場合はURLやheaderなどでversionを分け、migration period、deprecation policy、usage monitoringを用意します。

**Keywords:** API versioning, backward compatibility, additive change, breaking change, deprecation, migration

**Follow-up question:** How would you evolve an event schema without breaking consumers? / Consumerを壊さずにevent schemaを変更するにはどうしますか？

**Follow-up core answer:**

Optional fieldの追加などbackward/forward compatibilityを保つ変更を優先します。Schema registry、versioned contract、consumer-driven testを利用し、producerとconsumerを段階的に移行します。Fieldの意味を変更したり、すぐ削除したりしません。

**Follow-up keywords:** schema evolution, backward compatibility, forward compatibility, schema registry, consumer-driven contract, gradual migration

---

## API Security and Testing

### 7. API Security

**English question:** How would you secure a newly developed API?

**日本語の質問:** 新しいAPIをどのように保護しますか？

**Core answer:**

TLS、authentication、resource-level authorization、input validation、rate limiting、least privilege、secret management、audit loggingを組み合わせます。機密情報をlogへ残さず、依存libraryとconfigurationも継続的に検査します。

**Keywords:** TLS, authentication, authorization, validation, rate limiting, least privilege, secrets, audit logging

**Follow-up question:** What is the difference between authentication and authorization? / Authenticationとauthorizationの違いは何ですか？

**Follow-up core answer:**

Authenticationは「誰であるか」を確認する処理です。Authorizationは、確認された主体が「そのresourceやoperationへアクセスできるか」を判断する処理です。認証済みでも、他のユーザーのdataへアクセスできるとは限りません。

**Follow-up keywords:** identity, permission, principal, resource-level authorization, access control

---

### 8. SQL Injection

**English question:** How do you prevent SQL injection?

**日本語の質問:** SQL injectionをどのように防ぎますか？

**Core answer:**

Parameterized queryまたはprepared statementを使用し、入力値をSQL文字列へ直接連結しません。Database accountのpermissionを最小化し、入力検証とsecurity testも行います。ORMを使うだけでは完全な保証になりません。

**Keywords:** SQL injection, parameterized query, prepared statement, input validation, least privilege

**Follow-up question:** When can SQL injection still occur when using an ORM? / ORMを使っていてもSQL injectionが起こるのはどのような場合ですか？

**Follow-up core answer:**

Raw SQL、dynamic query、文字列連結、危険なescape処理を利用すると発生し得ます。また、table名やsort columnなどparameter bindingできない部分を未検証の入力から作る場合も注意が必要です。

**Follow-up keywords:** ORM, raw SQL, dynamic query, string concatenation, allowlist, parameter binding

---

### 9. File Uploads

**English question:** How would you handle file uploads in a web application?

**日本語の質問:** Web applicationでfile uploadをどう設計しますか？

**Core answer:**

Size、file type、contentの検証、authorization、unique object key、quota、malware scan、metadata管理を行います。大きなfileはapplication serverに全て読み込まず、object storageへのdirect uploadやstreamingを検討します。

**Keywords:** object storage, presigned URL, streaming, validation, authorization, quota, malware scanning

**Follow-up question:** How would you upload a multi-gigabyte file without overloading the backend? / 数GBのfileをuploadする場合、backend負荷をどう抑えますか？

**Follow-up core answer:**

Presigned URLを発行してclientからobject storageへmultipart uploadさせます。Backendはauthorizationとmetadataを管理し、upload完了後のvalidationやscanをasynchronous jobとして実行します。Retry可能なpart単位に分割します。

**Follow-up keywords:** presigned URL, multipart upload, object storage, asynchronous processing, retry, metadata

---

### 10. API Testing

**English question:** What tests would you write for a new API endpoint?

**日本語の質問:** 新しいAPI endpointにどのようなtestを書きますか？

**Core answer:**

Business logicのunit test、databaseやexternal serviceとのintegration test、重要なuser flowのend-to-end testを使い分けます。Authorization、invalid input、duplicate request、concurrency、dependency failureなどのnegative caseも確認します。

**Keywords:** unit test, integration test, end-to-end test, contract test, negative case, concurrency

**Follow-up question:** Does 90% test coverage guarantee high software quality? / Test coverageが90%なら、software qualityが高いと言えますか？

**Follow-up core answer:**

いいえ。Coverageは実行されたcodeの割合であり、assertionの質、重要なbehavior、boundary condition、production riskを保証しません。Risk-based testing、mutation testing、incident historyなどと合わせて判断します。

**Follow-up keywords:** code coverage, assertion quality, risk-based testing, boundary condition, mutation testing

---

## Data and Backend Foundations

### 11. SQL vs. NoSQL Databases

**English question:** What is the difference between SQL and NoSQL databases?

**日本語の質問:** SQL databaseとNoSQL databaseの違いは何ですか？

**Core answer:**

SQL databaseはtable、relation、SQL、transactionを中心とし、複雑なqueryや強い整合性に向いています。NoSQLはkey-value、document、wide-column、graphなど複数のmodelがあり、access pattern、柔軟なschema、水平分散を重視する場合に有力です。名前だけで選ばず、data modelと要件で判断します。

**Keywords:** relational database, NoSQL, schema, transaction, join, access pattern, horizontal scaling

**Follow-up question:** When would you choose a NoSQL database over a relational database? / Relational databaseではなくNoSQLを選ぶのはどのような場合ですか？

**Follow-up core answer:**

単純で明確なaccess pattern、大量の分散write、柔軟なdocument構造、またはgraph traversalなど、特定のNoSQL modelが要件に合う場合です。Transaction、join、consistency、運用経験を犠牲にしてよいかも確認します。

**Follow-up keywords:** workload, access pattern, scale, consistency, transaction, data model, operational maturity

---

### 12. Session Management

**English question:** How does session management work in a web application?

**日本語の質問:** Web applicationのsession managementはどのように動作しますか？

**Core answer:**

Login後にserverがsession identifierを発行し、browserは通常cookieで送信します。Serverはidentifierに対応するuser stateをsession storeから取得します。Cookieには`Secure`、`HttpOnly`、`SameSite`を設定し、expiration、rotation、logout時のinvalidationを管理します。

**Keywords:** session ID, cookie, session store, Secure, HttpOnly, SameSite, expiration, rotation

**Follow-up question:** What are the trade-offs between server-side sessions and JWTs? / Server-side sessionとJWTのtrade-offは何ですか？

**Follow-up core answer:**

Server-side sessionは即時失効やstate管理が容易ですが、共有storeが必要です。Self-contained JWTはservice間で検証しやすい一方、失効、key rotation、claimの古さ、token sizeに注意が必要です。JWTを使えば完全にstate管理が不要になるわけではありません。

**Follow-up keywords:** server-side session, JWT, revocation, shared store, key rotation, stale claims

---

### 13. Containerization

**English question:** What is containerization, and how does it benefit backend development?

**日本語の質問:** Containerizationとは何で、backend developmentにどのような利点がありますか？

**Core answer:**

Applicationとdependencyをimageとしてpackageし、host OSのkernelを共有しながら隔離されたprocessとして実行する仕組みです。環境差を減らし、再現可能なbuild、迅速なdeploy、resource isolationを実現します。ただしVMと同じ強度の境界ではなく、image securityやorchestrationも必要です。

**Keywords:** container, image, process isolation, reproducible build, portability, Docker, orchestration

**Follow-up question:** What is the difference between a container and a virtual machine? / Containerとvirtual machineの違いは何ですか？

**Follow-up core answer:**

VMはguest OSとkernelを含み、hypervisor上でhardwareをvirtualizeします。Containerはhost kernelを共有してprocessを隔離するため、一般に軽量で起動が速いです。一方、security boundaryやOS選択の自由度はVMの方が強い場合があります。

**Follow-up keywords:** host kernel, guest OS, hypervisor, isolation, startup time, security boundary

---

### 14. Scaling During a Traffic Surge

**English question:** How would you scale a backend application during a traffic surge?

**日本語の質問:** Traffic surge時にbackend applicationをどのようにscaleしますか？

**Core answer:**

最初にbottleneckとservice capacityをmetricsで確認します。Stateless instanceのhorizontal scaling、load balancing、cache、CDN、queue、database read replicaなどを使い、rate limitingやload sheddingで過負荷を防ぎます。事前のcapacity planningとautoscaling設定も重要です。

**Keywords:** horizontal scaling, load balancer, autoscaling, cache, queue, read replica, load shedding

**Follow-up question:** What if the database is the bottleneck? / Databaseがbottleneckの場合はどうしますか？

**Follow-up core answer:**

Slow queryとindexを確認し、connection数やtransactionを最適化します。その後、cache、read replica、partitioning、asynchronous writeを検討します。単にapplication instanceを増やすとconnectionやqueryが増え、悪化する可能性があります。

**Follow-up keywords:** slow query, index, connection pool, cache, read replica, partitioning, backpressure

---

### 15. Backend Debugging

**English question:** What tools and techniques do you use to debug a backend application?

**日本語の質問:** Backend applicationのdebugにどのようなtoolや手法を使いますか？

**Core answer:**

症状、影響範囲、開始時刻を明確にし、再現または比較可能な状態を作ります。Structured logs、metrics、distributed traces、profiler、debugger、database execution planを使い、仮説を一つずつ検証します。直近のchangeと正常なrequestとの差分も確認します。

**Keywords:** reproduction, hypothesis, structured logging, metrics, tracing, profiler, execution plan

**Follow-up question:** How would you investigate an issue that occurs only in production? / Productionでしか発生しない問題をどう調査しますか？

**Follow-up core answer:**

Request IDやtraceで失敗例を絞り、configuration、data、traffic、dependency、resource saturationを安全に比較します。機密情報を避けた追加telemetry、feature flag、traffic replayを利用し、必要なら影響を止めるrollbackを先に行います。

**Follow-up keywords:** production-only issue, request ID, telemetry, configuration drift, feature flag, rollback

---

### 16. Maintainable Backend Code

**English question:** How do you keep backend code maintainable and easy to understand?

**日本語の質問:** Backend codeを保守しやすく理解しやすい状態にするにはどうしますか？

**Core answer:**

明確な責務とmodule boundary、意味のある名前、小さなinterface、一貫したerror handlingを重視します。重要なbehaviorをtestし、API contractと設計判断をdocument化します。重複排除だけでなく、過度なabstractionを避けることも重要です。

**Keywords:** separation of concerns, module boundary, naming, interface, tests, documentation, simplicity

**Follow-up question:** How do you decide whether to introduce an abstraction? / Abstractionを導入すべきか、どう判断しますか？

**Follow-up core answer:**

複数の具体例に安定した共通性があり、変更理由を一か所に閉じ込められる場合に導入します。将来を推測した早すぎるabstractionは避け、理解コスト、testability、変更頻度を比較します。

**Follow-up keywords:** abstraction, duplication, change boundary, cognitive load, testability, YAGNI

---

## Search, Processing, Messaging, and Delivery

### 17. Full-Text Search

**English question:** How would you implement full-text search?

**日本語の質問:** Full-text searchをどのように実装しますか？

**Core answer:**

Textをtokenize、normalizeし、termからdocumentを引くinverted indexを作ります。Database内蔵検索またはsearch engineを使い、relevance、language analysis、filter、更新遅延、index容量を要件に合わせます。`LIKE '%word%'`だけでは大規模検索に適しません。

**Keywords:** full-text search, tokenizer, analyzer, inverted index, relevance, indexing, search engine

**Follow-up question:** How do you keep the search index consistent with the primary database? / Search indexとprimary databaseの整合性をどう保ちますか？

**Follow-up core answer:**

Databaseをsource of truthとし、outboxやchange data captureからindex更新eventを発行します。Consumerはidempotentにし、retry、dead-letter queue、periodic reconciliationで欠落や重複を修復します。通常はeventual consistencyを明示します。

**Follow-up keywords:** source of truth, outbox, CDC, idempotent consumer, reconciliation, eventual consistency

---

### 18. Batch Processing

**English question:** How would you approach batch processing in a data-heavy backend application?

**日本語の質問:** Data量が多いbackendでbatch processingをどう設計しますか？

**Core answer:**

Jobをboundedなchunkやpartitionに分け、schedulerとworkerで処理します。Checkpoint、idempotency、retry、timeout、failure isolationを用意し、memoryへ全dataを読み込みません。Throughput、completion deadline、downstream capacityを測定します。

**Keywords:** batch processing, chunking, partitioning, checkpoint, idempotency, retry, scheduler

**Follow-up question:** How would you safely restart a failed batch job? / 失敗したbatch jobを安全に再開するにはどうしますか？

**Follow-up core answer:**

処理済みpartitionやcursorをcheckpointとして永続化し、未完了部分から再開します。各writeをidempotentまたはtransactionalにし、同じchunkが再実行されてもduplicate resultを作らないようにします。

**Follow-up keywords:** restartability, checkpoint, cursor, idempotent write, transaction, duplicate prevention

---

### 19. Message Queues

**English question:** What are the uses and benefits of a message queue in a distributed system?

**日本語の質問:** Distributed systemでmessage queueを使う目的と利点は何ですか？

**Core answer:**

Producerとconsumerを時間的・構造的に分離し、asynchronous processing、traffic buffering、retryを可能にします。一方、duplicate delivery、ordering、backlog、poison message、eventual consistencyへの対応が必要です。

**Keywords:** message queue, decoupling, asynchronous processing, buffering, retry, ordering, backpressure

**Follow-up question:** How do you handle duplicate message delivery? / Messageが重複deliveryされた場合、どう処理しますか？

**Follow-up core answer:**

Consumerをidempotentにし、message IDやbusiness keyをdeduplication storeまたはdatabase unique constraintで記録します。処理と状態更新をtransactionalに結び付け、acknowledgementのtimingも設計します。

**Follow-up keywords:** at-least-once delivery, idempotent consumer, deduplication, unique constraint, acknowledgement

---

### 20. Database Connections Under High Load

**English question:** How do you manage database connections in a high-load scenario?

**日本語の質問:** High load時にdatabase connectionをどう管理しますか？

**Core answer:**

Bounded connection poolを使い、connection timeout、query timeout、idle lifetimeを設定します。Pool sizeは各instanceではなくdatabase全体のcapacityから逆算します。Slow queryと長いtransactionを減らし、admission controlでdatabaseを保護します。

**Keywords:** connection pool, pool size, timeout, database capacity, slow query, admission control

**Follow-up question:** Why can increasing the connection pool size make performance worse? / Connection poolを大きくすると、なぜ性能が悪化することがありますか？

**Follow-up core answer:**

Databaseが処理できる並列数を超えると、CPU、memory、lock contention、context switchingが増えます。Requestを待たせる場所がapplicationからdatabaseへ移るだけで、latencyとfailureが連鎖する可能性があります。

**Follow-up keywords:** saturation, contention, context switching, queueing, latency, cascading failure

---

### 21. CI/CD Pipeline

**English question:** How would you set up a CI/CD pipeline for backend services?

**日本語の質問:** Backend serviceのCI/CD pipelineをどのように構築しますか？

**Core answer:**

Commitごとにlint、unit/integration test、security scanを実行し、一度buildしたimmutable artifactを各environmentへ昇格させます。Deploymentはstaged rollout、health check、migration control、automatic rollbackを含め、誰が何をdeployしたか追跡可能にします。

**Keywords:** CI/CD, automated test, security scan, immutable artifact, staged rollout, rollback, auditability

**Follow-up question:** How would you deploy safely when a database migration is included? / Database migrationを含むchangeを安全にdeployするにはどうしますか？

**Follow-up core answer:**

Old codeとnew codeの両方にcompatibleなexpand-contract方式を使います。最初にadditive schema change、次にapplication移行とbackfill、最後に旧column削除を別deployで行います。

**Follow-up keywords:** expand-contract, backward compatibility, additive migration, backfill, phased deployment

---

### 22. Distributed Caching

**English question:** Describe a distributed caching strategy for a highly available application.

**日本語の質問:** High availability applicationのdistributed cache戦略を説明してください。

**Core answer:**

Cache-asideなどのpatternを選び、key、TTL、invalidation、maximum sizeを定義します。Cache nodeはpartitionとreplicationで可用性を高め、cache miss時にsource of truthへ戻れるようにします。Cache stampedeとhot keyも保護します。

**Keywords:** distributed cache, cache-aside, TTL, invalidation, partitioning, replication, hot key

**Follow-up question:** How do you prevent a cache stampede? / Cache stampedeをどう防ぎますか？

**Follow-up core answer:**

TTLへjitterを加え、同じkeyの再計算をsingle-flightやdistributed lockで一つにまとめます。必要に応じてrefresh-ahead、stale-while-revalidate、request coalescingを利用します。

**Follow-up keywords:** cache stampede, TTL jitter, single-flight, distributed lock, refresh-ahead, stale data

---

### 23. Background Tasks

**English question:** What methods can you use to manage background tasks?

**日本語の質問:** Background taskをどのように管理しますか？

**Core answer:**

Durabilityが不要な短い処理はin-process executorでもよいですが、重要な処理はdurable queueとworkerを使います。Job state、retry policy、timeout、idempotency、dead-letter queue、concurrency limit、monitoringを設計します。

**Keywords:** background job, worker, durable queue, retry, timeout, dead-letter queue, concurrency limit

**Follow-up question:** How do you decide whether a task can run in-process? / Taskをin-processで実行してよいか、どう判断しますか？

**Follow-up core answer:**

Process crash時に失われてもよく、短時間で、resource使用量が小さいbest-effort処理なら候補です。必ず完了すべき処理、長時間処理、retryが必要な処理はexternal durable queueへ移します。

**Follow-up keywords:** in-process task, best effort, durability, process crash, external worker, retry

---

### 24. Data Encryption

**English question:** How do you handle data encryption and decryption in a privacy-focused application?

**日本語の質問:** Privacyを重視するapplicationでdataのencryptionとdecryptionをどう扱いますか？

**Core answer:**

通信中はTLS、保存時はstorage encryptionを使い、特に機密性の高いfieldはapplication-level encryptionを検討します。Keyはdataと分離してKMS/HSMで管理し、rotation、access control、auditを行います。Passwordは復号可能にせずsalt付きhashで保存します。

**Keywords:** encryption in transit, encryption at rest, KMS, HSM, key rotation, envelope encryption, password hashing

**Follow-up question:** How would you rotate encryption keys without long downtime? / 長いdowntimeなしでencryption keyをrotationするにはどうしますか？

**Follow-up core answer:**

Key versionをciphertext metadataに保存し、新keyでwriteしながら旧keyでもreadできる期間を設けます。Data keyを再wrapするか、background jobで段階的に再暗号化し、完了確認後に旧keyを無効化します。

**Follow-up keywords:** key version, dual read, new-key write, rewrapping, background re-encryption, key retirement

---

### 25. Webhooks

**English question:** What are webhooks, and how would you implement them?

**日本語の質問:** Webhookとは何で、どのように実装しますか？

**Core answer:**

Event発生時にproviderがconsumerのHTTP endpointへnotificationを送る仕組みです。Deliveryをqueue化し、signature、timestamp、retry with backoff、delivery ID、timeout、observabilityを用意します。Receiverは素早く応答し、処理を非同期化します。

**Keywords:** webhook, event notification, signature, timestamp, retry, delivery ID, asynchronous processing

**Follow-up question:** How does a webhook receiver prevent spoofing and replay attacks? / Webhook receiverはspoofingとreplay attackをどう防ぎますか？

**Follow-up core answer:**

Raw request bodyとtimestampをshared secretまたはpublic keyで検証し、constant-time comparisonを使います。許容時間外のtimestampと処理済みdelivery IDを拒否し、secretを安全にrotationします。

**Follow-up keywords:** HMAC, signature verification, raw body, replay protection, timestamp window, secret rotation

---

### 26. Privacy and GDPR

**English question:** What must a backend system consider for GDPR compliance?

**日本語の質問:** Backend systemでGDPR complianceのために何を考慮しますか？

**Core answer:**

Personal dataの所在と処理目的を把握し、data minimization、lawful basis、consent、retention、access control、auditを設計します。Access、correction、portability、deletionなどdata subjectのrequestへ対応し、processorやcross-border transferも管理します。

**Keywords:** personal data, data minimization, lawful basis, consent, retention, right to erasure, audit

**Follow-up question:** How would you implement a user's right-to-erasure request? / Userから削除要求を受けた場合、どう実装しますか？

**Follow-up core answer:**

User identityと対象dataを確認し、primary storeだけでなくcache、search index、derived data、downstream processorへ削除を伝播します。法的保存義務があるdataは隔離し、backupのexpiration方針と完了auditを明確にします。

**Follow-up keywords:** data inventory, deletion workflow, downstream propagation, backups, legal retention, audit trail

---

## Asynchronous APIs, Operations, and Architecture

### 27. Long-Running Processes

**English question:** How would you handle a long-running process triggered by a web request?

**日本語の質問:** Web requestから開始されるlong-running processをどう扱いますか？

**Core answer:**

Request内で完了を待たず、jobをdurable queueへ登録して`202 Accepted`とjob IDを返します。Clientはstatus endpoint、webhook、SSEなどで進捗を取得します。Jobはidempotentで、retry、cancellation、timeoutを扱います。

**Keywords:** 202 Accepted, job ID, durable queue, worker, polling, webhook, cancellation

**Follow-up question:** How do you prevent the same long-running job from being submitted twice? / 同じlong-running jobの二重登録をどう防ぎますか？

**Follow-up core answer:**

Client-supplied idempotency keyまたはbusiness keyにunique constraintを設定し、既存jobとresultを返します。受付記録とqueueへのpublishにはtransactional outboxなどを使い、部分失敗を避けます。

**Follow-up keywords:** idempotency key, unique constraint, job state, transactional outbox, duplicate submission

---

### 28. Rate Limiting

**English question:** How would you implement rate limiting to protect an API from abuse?

**日本語の質問:** APIをabuseから保護するrate limitingをどう実装しますか？

**Core answer:**

User、API key、IP、resourceなど適切なidentityごとにlimitを定義し、token bucketやsliding windowを使います。超過時は`429 Too Many Requests`と`Retry-After`を返します。Distributed環境ではRedisのatomic operationなどでcounterを共有します。

**Keywords:** rate limiting, token bucket, sliding window, quota, Redis, 429, Retry-After

**Follow-up question:** What are the trade-offs between token bucket and fixed window limiting? / Token bucketとfixed windowのtrade-offは何ですか？

**Follow-up core answer:**

Fixed windowは単純ですが、window境界でburstが倍増する可能性があります。Token bucketは一定rateを保ちつつ許容burstを表現できますが、state更新が少し複雑です。公平性、精度、storage costで選びます。

**Follow-up keywords:** token bucket, fixed window, burst, fairness, precision, state cost

---

### 29. Instrumentation and Monitoring

**English question:** How do you instrument and monitor backend application performance?

**日本語の質問:** Backend applicationのperformanceをどう計測・監視しますか？

**Core answer:**

Request rate、error rate、latency、resource saturationをmetricsとして収集し、structured logsとdistributed tracesをcorrelationします。Userに重要なSLI/SLOを定義してalertを設定し、dashboardだけでなくaction可能なsignalにします。

**Keywords:** metrics, logs, traces, RED method, saturation, SLI, SLO, alerting

**Follow-up question:** Why are p95 and p99 latencies more useful than the average? / Averageよりp95やp99 latencyが重要なのはなぜですか？

**Follow-up core answer:**

Averageは遅い少数のrequestを隠します。Percentileはuserのtail latencyを表し、dependency delay、queueing、resource contentionなど、一部requestだけに影響する問題を見つけやすくします。

**Follow-up keywords:** percentile, p95, p99, tail latency, queueing, outlier

---

### 30. Decomposing a Monolith

**English question:** What are microservices, and how would you decompose a monolith into them?

**日本語の質問:** Microservicesとは何で、monolithをどのように分割しますか？

**Core answer:**

Microserviceはbusiness capabilityごとに独立してdeploy・運用でき、data ownershipを持つserviceです。分割時はbounded contextと変更頻度を分析し、明確なboundaryからStrangler patternで段階的に切り出します。最初から一括で書き換えません。

**Keywords:** microservices, business capability, bounded context, data ownership, Strangler pattern, incremental migration

**Follow-up question:** What would you extract first from a monolith? / Monolithから最初に何を切り出しますか？

**Follow-up core answer:**

Boundaryが明確で他領域とのtransactionが少なく、独立scaleや変更速度に具体的な価値があるcapabilityを選びます。Shared database tableをそのまま共有せず、ownershipとcontractを先に決めます。

**Follow-up keywords:** service boundary, coupling, independent scaling, data ownership, contract, migration risk

---

### 31. API Dependencies

**English question:** How do you manage API dependencies in backend systems?

**日本語の質問:** Backend systemでAPI dependencyをどのように管理しますか？

**Core answer:**

API contractとversion policyを明確にし、timeout、bounded retry、circuit breaker、fallbackでruntime failureを隔離します。Client libraryとdependency versionを管理し、contract test、sandbox、usage monitoringでchangeを早期検出します。

**Keywords:** API dependency, contract, versioning, timeout, retry, circuit breaker, contract test

**Follow-up question:** When should you retry a failed downstream API call? / Downstream API callの失敗をいつretryすべきですか？

**Follow-up core answer:**

Timeout、connection error、`429`、一部の`5xx`など一時的な失敗で、operationがidempotentな場合に限定します。Exponential backoffとjitter、最大回数、全体deadlineを設定し、permanent errorはretryしません。

**Follow-up keywords:** transient failure, idempotency, exponential backoff, jitter, retry budget, deadline

---

### 32. Eventual Consistency

**English question:** What is eventual consistency, and what are its implications?

**日本語の質問:** Eventual consistencyとは何で、どのような影響がありますか？

**Core answer:**

Update直後はreplicaやservice間で異なる値を返す可能性がありますが、新しいupdateがなければ最終的に同じ状態へ収束するmodelです。Stale read、ordering、conflict、user experienceを設計し、reconciliationとobservabilityを用意します。

**Keywords:** eventual consistency, stale read, convergence, conflict, ordering, reconciliation

**Follow-up question:** How can you provide read-your-writes behavior in an eventually consistent system? / Eventual consistencyのsystemでread-your-writesをどう実現しますか？

**Follow-up core answer:**

Write後の一定期間はleaderまたはprimaryからreadする、sessionにversionを保持して必要なreplica lagを待つ、またはresponseに更新結果を返してclient stateへ反映する方法があります。追加latencyとのtrade-offを説明します。

**Follow-up keywords:** read-your-writes, primary read, version token, replica lag, session guarantee

---

### 33. Reverse Proxy

**English question:** What is a reverse proxy, and how is it useful in backend development?

**日本語の質問:** Reverse proxyとは何で、backendでどのように役立ちますか？

**Core answer:**

Clientのrequestを受け、内部のserverへ代理転送するcomponentです。TLS termination、routing、load balancing、compression、cache、rate limitingなど共通機能をapplicationから分離できます。High availabilityと正しいclient IP/header設定が必要です。

**Keywords:** reverse proxy, TLS termination, routing, load balancing, caching, rate limiting

**Follow-up question:** What is the difference between a reverse proxy and a forward proxy? / Reverse proxyとforward proxyの違いは何ですか？

**Follow-up core answer:**

Forward proxyはclient側の代理として外部serverへ接続し、serverからclientを隠します。Reverse proxyはserver群の入口としてclient requestを受け、内部構成を隠します。

**Follow-up keywords:** forward proxy, reverse proxy, client intermediary, server gateway, network boundary

---

### 34. Session State in a Load-Balanced Environment

**English question:** How would you handle session state in a load-balanced application?

**日本語の質問:** Load-balanced applicationでsession stateをどう扱いますか？

**Core answer:**

Application instanceのlocal memoryへ依存せず、Redisなどのshared session storeまたは検証可能なtokenを使います。Storeのreplication、TTL、failure behaviorを設計します。Sticky sessionは簡単ですが、uneven loadとfailoverの問題があります。

**Keywords:** shared session store, Redis, token, sticky session, TTL, failover, load balancing

**Follow-up question:** What happens if the shared session store becomes unavailable? / Shared session storeが利用不能になったらどうしますか？

**Follow-up core answer:**

既存requestが認証できず大量logoutやerrorになる可能性があります。Replication、automatic failover、bounded timeout、capacity planningを用意し、security上fail-openかfail-closedかを明示します。通常、認証は安全側に倒します。

**Follow-up keywords:** session-store failure, replication, failover, timeout, fail-closed, availability

---

## Advanced Databases and Distributed Systems

### 35. Database Replication

**English question:** What is database replication, and how does it support fault tolerance?

**日本語の質問:** Database replicationとは何で、fault toleranceにどう役立ちますか？

**Core answer:**

同じdataのcopyを複数nodeへ保持し、primary failure時のfailoverやread scalingを可能にします。Synchronous replicationはdata lossを抑える一方latencyが増え、asynchronous replicationは速い一方replication lagとfailover時のdata lossがあり得ます。

**Keywords:** replication, primary, replica, failover, replication lag, synchronous, asynchronous

**Follow-up question:** How do RPO and RTO affect your replication design? / RPOとRTOはreplication設計にどう影響しますか？

**Follow-up core answer:**

RPOは許容data loss、RTOは許容復旧時間です。小さいRPOにはsynchronousまたは低lag replication、小さいRTOにはautomatic failoverと事前準備されたcapacityが必要です。Costとlatencyが増える点を合意します。

**Follow-up keywords:** RPO, RTO, data loss, recovery time, synchronous replication, automatic failover

---

### 36. Blue-Green Deployment

**English question:** Describe a blue-green deployment strategy.

**日本語の質問:** Blue-green deployment strategyを説明してください。

**Core answer:**

Current versionとnew versionの二つの同等environmentを用意し、new側を検証してからtrafficを切り替える方法です。問題時はroutingを戻して迅速にrollbackできますが、二重capacityとdatabase compatibilityが必要です。

**Keywords:** blue-green deployment, parallel environments, traffic switch, rollback, capacity, compatibility

**Follow-up question:** Why can a database migration make blue-green rollback unsafe? / Database migrationがあるとblue-green rollbackが危険になるのはなぜですか？

**Follow-up core answer:**

New codeがschemaやdataを不可逆に変更すると、old codeが読めなくなるためです。Expand-contract migrationを使い、切替期間は両versionが同じschemaを扱えるようにします。

**Follow-up keywords:** backward-compatible schema, expand-contract, irreversible migration, rollback safety

---

### 37. CAP Theorem and Consistency Models

**English question:** Explain consistency models in distributed databases and the CAP theorem.

**日本語の質問:** Distributed databaseのconsistency modelとCAP theoremを説明してください。

**Core answer:**

CAP theoremはnetwork partition中に、linearizable consistencyとavailabilityを同時に完全保証できないことを示します。Distributed systemではpartitionへの備えが必要なので、発生時にrejectやwaitしてconsistencyを守るか、stale/conflicting responseを許してavailabilityを守るかを選びます。

**Keywords:** CAP theorem, network partition, consistency, availability, linearizability, trade-off

**Follow-up question:** Does CAP mean that a system simply chooses any two of C, A, and P? / CAPはC、A、Pから単純に二つ選ぶという意味ですか？

**Follow-up core answer:**

いいえ。実用的なdistributed systemはpartitionを想定し、その期間にconsistencyとavailabilityのどちらを優先するかをoperation単位で判断します。通常時のlatencyやconsistencyはPACELCなど別のtrade-offも関係します。

**Follow-up keywords:** partition scenario, operation-level choice, PACELC, latency, normal operation

---

### 38. Schema Migrations in Continuous Delivery

**English question:** How do you manage schema migrations in continuous delivery?

**日本語の質問:** Continuous delivery環境でschema migrationをどう管理しますか？

**Core answer:**

Migrationをversion controlし、自動化して一度だけ実行します。Expand-contractでadditive change、dual read/writeまたはbackfill、application切替、旧schema削除を分離します。Large tableではlock時間、replication lag、rollback手順を事前検証します。

**Keywords:** schema migration, version control, expand-contract, backfill, online migration, rollback

**Follow-up question:** How would you add a non-null column to a very large table? / 非常に大きいtableへnon-null columnをどう追加しますか？

**Follow-up core answer:**

まずnullableまたは安全なdefaultでcolumnを追加し、new writeで値を設定します。既存rowをsmall batchでbackfillして検証し、最後にconstraintを有効化します。Database固有のlocking behaviorも確認します。

**Follow-up keywords:** nullable column, batched backfill, dual write, constraint validation, table lock

---

### 39. Single Sign-On

**English question:** How would you implement a single sign-on solution?

**日本語の質問:** Single sign-onをどのように実装しますか？

**Core answer:**

Identity Providerを中心に、web/mobile loginでは通常OpenID Connect、enterprise federationではSAMLも利用します。Authorization Code Flow、`state`、`nonce`、PKCEを適切に使い、issuer、audience、signature、expirationを検証してlocal sessionを作ります。

**Keywords:** SSO, Identity Provider, OpenID Connect, OAuth 2.0, SAML, authorization code, PKCE

**Follow-up question:** What is the difference between OAuth 2.0 and OpenID Connect? / OAuth 2.0とOpenID Connectの違いは何ですか？

**Follow-up core answer:**

OAuth 2.0はresource accessのauthorization frameworkで、authentication protocolそのものではありません。OpenID ConnectはOAuth 2.0上にidentity layerを追加し、ID TokenとUserInfoでuser authenticationを標準化します。

**Follow-up keywords:** authorization, authentication, OAuth 2.0, OpenID Connect, ID Token, UserInfo

---

### 40. IoT Data Streams

**English question:** How would you design a backend system for IoT device data streams?

**日本語の質問:** IoT deviceのdata streamを扱うbackendをどう設計しますか？

**Core answer:**

Device identityと認証を行うingestion gateway、MQTTなどのbroker、durable stream、partitioned processing、time-series storeまたはdata lakeを組み合わせます。大量connection、out-of-order/duplicate data、offline device、backpressure、firmware compatibilityを考慮します。

**Keywords:** IoT, device identity, MQTT, ingestion, stream processing, time-series data, backpressure

**Follow-up question:** How would you handle duplicate or out-of-order device events? / 重複または順序が前後したdevice eventをどう扱いますか？

**Follow-up core answer:**

Device ID、sequence number、event timestampを含め、consumer側でdeduplicateします。必要な時間範囲のbufferとwatermarkを使って並べ替え、遅すぎるeventの処理policyを定義します。完全なglobal orderingは求めません。

**Follow-up keywords:** sequence number, event time, deduplication, watermark, late event, partition ordering

---

### 41. Real-Time Data Synchronization

**English question:** How would you support real-time data synchronization across devices?

**日本語の質問:** 複数device間のreal-time data synchronizationをどう実現しますか？

**Core answer:**

Changeをauthoritative storeへ書き、event streamとpub/subで各connection gatewayへ配信します。ClientとはWebSocket、SSE、またはpush notificationを使い、version、cursor、reconnect時のcatch-upを管理します。Concurrent updateのconflict policyも必要です。

**Keywords:** real-time sync, WebSocket, SSE, pub/sub, cursor, reconnect, conflict resolution

**Follow-up question:** How does a client recover events missed while it was offline? / Offline中に失ったeventをclientはどう復旧しますか？

**Follow-up core answer:**

Clientが最後に適用したversionまたはoffsetを保存し、再接続時に差分eventを取得します。Retention期間を超えた場合はsnapshotを取得してから新しいstreamを再開します。Event適用はidempotentにします。

**Follow-up keywords:** offset, version, catch-up, event retention, snapshot, idempotent apply

---

### 42. Microservice Trade-Offs

**English question:** What are the benefits and drawbacks of microservice architecture?

**日本語の質問:** Microservice architectureの利点と欠点は何ですか？

**Core answer:**

利点はindependent deployment/scaling、team autonomy、failure isolation、technology choiceです。欠点はnetwork failure、distributed transaction、data consistency、observability、testing、deployment platformの複雑さです。組織とsystem規模がcostを正当化するときに採用します。

**Keywords:** independent deployment, team autonomy, failure isolation, distributed complexity, observability

**Follow-up question:** When is a modular monolith a better choice? / Modular monolithの方が良いのはどのような場合ですか？

**Follow-up core answer:**

Teamが小さい、domain boundaryがまだ不明、独立scaleの必要が少ない、強いtransactionが多い場合です。Process内でもmodule boundaryとownershipを明確にすれば、将来の分割余地を残しながら運用complexityを抑えられます。

**Follow-up keywords:** modular monolith, small team, domain boundary, transaction, operational simplicity

---

### 43. Load Testing

**English question:** How would you approach load testing a backend API?

**日本語の質問:** Backend APIのload testingをどう進めますか？

**Core answer:**

Expected traffic、peak、request mix、data size、SLOを先に定義し、本番に近いenvironmentで段階的にloadを上げます。Throughput、p95/p99 latency、error rate、CPU、memory、database、queueを測定し、bottleneckとbreaking pointを確認します。

**Keywords:** load test, workload model, ramp-up, throughput, p95, p99, saturation, breaking point

**Follow-up question:** What is the difference between load, stress, spike, and soak testing? / Load、stress、spike、soak testの違いは何ですか？

**Follow-up core answer:**

Load testは想定負荷、stress testは限界超過、spike testは急増、soak testは長時間継続を検証します。それぞれcapacity、degradation、autoscaling、memory leakなど異なるriskを見ます。

**Follow-up keywords:** load test, stress test, spike test, soak test, capacity, degradation, memory leak

---

### 44. Cache Eviction

**English question:** How would you implement a server-side cache eviction strategy?

**日本語の質問:** Server-side cacheのeviction strategyをどう設計しますか？

**Core answer:**

Freshness要件にはTTL、容量制限にはLRU、LFUなどを使い、write時のinvalidationも検討します。Workload、object size、hit rate、再計算costを測り、memory上限を設定します。Evictionとbusiness data deletionは別概念です。

**Keywords:** cache eviction, TTL, LRU, LFU, invalidation, hit rate, memory limit

**Follow-up question:** How do you choose between LRU and LFU? / LRUとLFUをどう選びますか？

**Follow-up core answer:**

LRUは最近使われたitemを重視し、access patternが時間とともに変わる場合に適します。LFUは継続的に人気のitemを残せますが、過去の人気が残る問題があります。実際のhit rateとmemory costで検証します。

**Follow-up keywords:** LRU, LFU, recency, frequency, workload shift, cache hit rate

---

### 45. Correlation IDs and Tracing

**English question:** What are correlation IDs, and how are they used across services?

**日本語の質問:** Correlation IDとは何で、service間でどう使いますか？

**Core answer:**

一つのlogical requestに共通identifierを付け、service、queue、logをまたいで処理を検索できるようにします。入口で生成または検証し、downstream headerとmessage metadataへ伝播します。Distributed tracingではtrace IDとspan IDでより詳細な因果関係を表します。

**Keywords:** correlation ID, request ID, trace ID, span ID, context propagation, structured logging

**Follow-up question:** What should you be careful about when accepting a correlation ID from a client? / Clientからcorrelation IDを受け取る際の注意点は何ですか？

**Follow-up core answer:**

Lengthとformatを検証し、log injectionや高cardinality abuseを防ぎます。External IDとinternal trace IDを分ける方法もあります。Personal dataやsecretをIDやtracing baggageへ入れません。

**Follow-up keywords:** input validation, log injection, cardinality, external ID, sensitive data, tracing baggage

---

### 46. Optimistic vs. Pessimistic Locking

**English question:** What is the difference between optimistic and pessimistic locking?

**日本語の質問:** Optimistic lockingとpessimistic lockingの違いは何ですか？

**Core answer:**

Optimistic lockingはversionやtimestampを比較してupdate時にconflictを検出し、競合が少ない場合に適します。Pessimistic lockingは先にrowなどをlockし、競合を防ぎますが、待ち時間、deadlock、throughput低下が発生し得ます。

**Keywords:** optimistic locking, version column, compare-and-swap, pessimistic locking, row lock, conflict

**Follow-up question:** How do you handle an optimistic-lock conflict? / Optimistic lockのconflictをどう処理しますか？

**Follow-up core answer:**

最新dataを再読込し、安全ならbounded retryします。User編集なら差分を表示してmergeまたは再入力を求めます。Blind retryは他者の変更を上書きするため、operationのsemanticsを確認します。

**Follow-up keywords:** version conflict, reload, bounded retry, merge, lost update, user feedback

---

### 47. Preventing Database Deadlocks

**English question:** How do you prevent and handle deadlocks in database transactions?

**日本語の質問:** Database transactionのdeadlockをどう予防・処理しますか？

**Core answer:**

複数resourceを常に同じ順序でlockし、transactionを短く保ち、適切なindexでlock範囲を小さくします。External callをtransaction内で行いません。それでも発生し得るため、databaseが選んだvictim transactionをrollbackし、jitter付きでretryします。

**Keywords:** deadlock, lock order, short transaction, index, rollback, retry, jitter

**Follow-up question:** How is a deadlock different from lock contention? / Deadlockとlock contentionの違いは何ですか？

**Follow-up core answer:**

Lock contentionは一方がlock解放を待つ状態で、通常は進行可能です。Deadlockはtransaction同士が循環待ちになり、そのままでは進めません。Databaseはcycleを検出し、一つをabortして解消します。

**Follow-up keywords:** lock contention, circular wait, wait-for graph, deadlock detection, victim transaction

---

### 48. Inter-Service Communication Security

**English question:** How would you secure inter-service communication in a microservices architecture?

**日本語の質問:** Microservices間のcommunicationをどう保護しますか？

**Core answer:**

Network内部も信用せず、mTLSで相互認証と暗号化を行い、workload identityに基づく細粒度authorizationを適用します。Short-lived credential、secret rotation、network policy、audit loggingを使い、serviceごとにleast privilegeを守ります。

**Keywords:** zero trust, mTLS, workload identity, authorization, network policy, short-lived credential

**Follow-up question:** Is TLS alone enough to secure service-to-service calls? / Service間通信はTLSだけで十分ですか？

**Follow-up core answer:**

いいえ。TLSは通信の暗号化を提供しますが、相手のservice identity、operationごとのpermission、credential lifecycle、application-level validationも必要です。mTLSとauthorization policyを組み合わせます。

**Follow-up keywords:** encryption, mutual authentication, authorization policy, identity, credential lifecycle

---

### 49. Data Anomaly Prevention and Detection

**English question:** How do you prevent and detect data anomalies in large-scale systems?

**日本語の質問:** Large-scale systemでdata anomalyをどう予防・検出しますか？

**Core answer:**

Schema、type、range、referential integrity、unique constraintを入口とstorageで検証します。Data contract、versioning、idempotencyで発生を減らし、quality metrics、distribution monitoring、reconciliation job、lineageで異常を検出・追跡します。疑わしいdataはquarantineします。

**Keywords:** data validation, constraint, data contract, data quality, reconciliation, lineage, quarantine

**Follow-up question:** How would you detect a silent change in an upstream data distribution? / Upstream data distributionの静かな変化をどう検出しますか？

**Follow-up core answer:**

Null rate、cardinality、range、category比率、statistical distributionをbaselineと比較し、schema checkだけでは見えないdriftをalertします。Source versionとlineageを記録し、sampleをquarantineしてdownstream影響を確認します。

**Follow-up keywords:** data drift, baseline, null rate, cardinality, distribution monitoring, lineage

---

### 50. Global High-Availability Data Storage

**English question:** How would you design global, highly available data storage for a multinational application?

**日本語の質問:** Multinational application向けのglobal high-availability data storageをどう設計しますか？

**Core answer:**

最初にregion別traffic、latency、consistency、RPO/RTO、data residencyを定義します。Multi-AZを基本に、必要ならmulti-region replication、global routing、automatic failoverを構成します。Write topologyとconflict policyを明確にし、backup restoreとregion failoverを定期的に演習します。

**Keywords:** multi-region, multi-AZ, data residency, replication, global routing, RPO, RTO, disaster recovery

**Follow-up question:** How would you choose between single-writer and multi-writer replication? / Single-writerとmulti-writer replicationをどう選びますか？

**Follow-up core answer:**

Single-writerはconflictを避けてconsistencyを保ちやすい一方、remote write latencyとleader failureの影響があります。Multi-writerはlocal write availabilityを高めますが、conflict resolution、ordering、運用が複雑です。Business invariantごとに選びます。

**Follow-up keywords:** single-writer, multi-writer, write latency, conflict resolution, ordering, business invariant
