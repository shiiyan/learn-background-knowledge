# Backend Knowledge Quick Reference

A bilingual, category-based reference for reviewing common backend engineering interview questions.

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

## Planned Categories

The reference will be expanded iteratively with these categories:

1. API and HTTP fundamentals
2. API security and testing
3. Databases, locking, and replication
4. Caching and session management
5. Messaging and background processing
6. Architecture and distributed systems
7. Scalability and performance
8. Debugging and observability
9. CI/CD and deployment
10. Real-time and data-intensive systems

