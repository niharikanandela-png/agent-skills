---
name: lambda-java-analysis
description: >
  Analysis skill for AWS Lambda functions written in Java.
  Use when the Code Analyst Agent is reading Java Lambda source files
  and pom.xml to populate the 10-dimension analysis manifest.
  Covers trigger identification, SDK detection, integrations, runtime
  patterns, config/secrets access, build artefacts, security signals,
  observability, and functional summarisation.
  Not for Azure translation, IaC analysis, or CI/CD pipeline reading
  — those are handled by the Context Analyst Agent.
---

# Lambda Java Analysis Skill

## How to use this skill

This skill is structured as a **fetch-first reference**. For each analysis
section below:

1. Read the compact reference in this file to understand what to look for
2. If you encounter an ambiguous case, fetch the linked official documentation
3. Never guess — if a pattern is not in this file and not in the docs, record `UNKNOWN`

**To fetch documentation:** Use `web_fetch` with the URL provided in each section.
Add `?from=agent-skill` as a query hint if supported by your platform.

---

## Section 1 — Trigger Identification

**When:** First thing to establish. Determines the entire invocation model.  
**Official docs to fetch if needed:**
- Handler interfaces: https://docs.aws.amazon.com/lambda/latest/dg/java-handler.html
- All event source types: https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventsourcemapping.html
- Java events library (full event class list): https://github.com/aws/aws-lambda-java-libs/tree/main/aws-lambda-java-events/src/main/java/com/amazonaws/services/lambda/runtime/events

### 1.1 Handler Interface

Read the `implements` clause:

| Clause | Record `handler_interface` |
|---|---|
| `RequestHandler<I, O>` | `RequestHandler` |
| `RequestStreamHandler` | `RequestStreamHandler` — add flag `STREAM_HANDLER_REVIEW`, set `trigger_type: UNKNOWN` |

### 1.2 Trigger Type from Handler Signature

Read the **first parameter type** of `handleRequest()`.

**Compact lookup table** — these are stable and finite:

| Parameter type (short name) | `trigger_type` |
|---|---|
| `SQSEvent` | `SQS` |
| `SNSEvent` | `SNS` |
| `APIGatewayProxyRequestEvent` | `HTTP_v1` |
| `APIGatewayV2HTTPEvent` | `HTTP_v2` |
| `APIGatewayV2WebSocketEvent` | `WebSocket` |
| `ApplicationLoadBalancerRequestEvent` | `ALB` |
| `ScheduledEvent` | `Schedule` |
| `S3Event` | `S3` |
| `S3BatchEvent` | `S3Batch` |
| `KinesisEvent` | `Kinesis` |
| `KinesisFirehoseEvent` | `KinesisFirehose` |
| `DynamodbEvent` | `DynamoDBStream` |
| `MSKEvent` | `MSK` |
| `KafkaEvent` | `SelfManagedKafka` |
| `CloudWatchLogsEvent` | `CloudWatchLogs` |
| `SimpleEmailEvent` | `SES` |
| `CognitoUserPoolEvent` | `Cognito` |
| `CloudFormationCustomResourceEvent` | `CloudFormation` |
| `ConfigEvent` | `AWSConfig` |
| `IoTButtonEvent` | `IoT` |
| `Map<String, Object>` or `Object` | `AMBIGUOUS` — flag `GENERIC_EVENT_TYPE` |
| No match | `UNKNOWN` |

**Special cases requiring reasoning (not in table):**

- **EventBridge custom bus:** No typed Java class exists. Detect by: `Map` input with keys `source`, `detail-type`, `detail`; or comments referencing EventBridge or event bus ARN. Set `trigger_type: EventBridge`, flag `EVENTBRIDGE_NO_TYPED_CLASS`.
- **DocumentDB trigger:** No typed class. Input is `Map<String, Object>` with `events[]` containing `operationType`, `ns.db`, `ns.coll` keys. Set `trigger_type: DocumentDB`, flag `DOCUMENTDB_TRIGGER`.
- **Step Functions task:** No typed class. Input is arbitrary JSON. Detect by comments or env vars referencing state machine ARN or `taskToken`. Set `trigger_type: OrchestratorInvoke`.

If the full list of all event types is needed: fetch https://github.com/aws/aws-lambda-java-libs/tree/main/aws-lambda-java-events/src/main/java/com/amazonaws/services/lambda/runtime/events

### 1.3 HTTP Trigger Metadata (when trigger_type is HTTP_v1, HTTP_v2, or ALB)

Also capture from code:
- `event.getHttpMethod()` checks → `http_methods[]`
- `event.getPathParameters().get("x")` → `http_path_params[]` (key names)
- `event.getQueryStringParameters().get("x")` → `http_query_params[]` (key names)
- `event.getHeaders().get("x")` → `http_headers_read[]`
- `event.getBody()` → `http_reads_body: true`
- `SQSBatchResponse` as return type → `sqs_partial_batch_failure: true`

---

## Section 2 — AWS SDK Detection

**When:** After trigger. Identifies AWS service integrations inside the function body.  
**Official docs to fetch if needed:**
- SDK v2 developer guide: https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/
- SDK v2 API reference (all service clients): https://sdk.amazonaws.com/java/api/latest/
- SDK v1 developer guide: https://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/
- SDK v1 API reference: https://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/

### 2.1 Version Detection Rule

| Import root | `sdk_version` |
|---|---|
| `com.amazonaws.*` | `v1` |
| `software.amazon.awssdk.*` | `v3` |
| Both present | `MIXED` — flag `MIXED_SDK_VERSIONS` |
| Neither | `none` |

> SDK v1 reached end-of-support December 31 2025. Flag any v1 SDK client usage (except `aws-lambda-java-*` runtime libs) with `SDK_V1_EOL`.

### 2.2 Service Client Naming Rules (derive, don't memorise)

**SDK v3 pattern:** `software.amazon.awssdk.services.{service}.{ServiceName}Client`
Extract the `{service}` segment from the import path as the service name.

**SDK v1 pattern:** `com.amazonaws.services.{service}.Amazon{ServiceName}`
Extract the `{service}` segment or the `{ServiceName}` suffix.

**High-level wrappers** (don't follow the naming pattern — note explicitly):
- `DynamoDBMapper` (v1) / `DynamoDbEnhancedClient` (v3) → service: `DynamoDB`, pattern: `enhanced`
- `TransferManager` (v1) / `S3TransferManager` (v3) → service: `S3`, pattern: `transfer_manager`

If you encounter a client name not derivable from the pattern, fetch the SDK v2 API reference above.

### 2.3 Operations to Capture

For every service client: record **all distinct method calls** made on it.  
Do not use a predefined list — read what is actually called.

```java
s3Client.putObject(...)    → operations: ["putObject"]
dynamo.query(...)          → operations: ["query"]
sqsClient.sendMessage(...) → operations: ["sendMessage"]
```

### 2.4 Cross-Invoke Detection (High Priority)

`LambdaClient.invoke()` (v3) or `AWSLambda.invoke()` (v1) = Lambda calling another Lambda.  
Always flag `CROSS_LAMBDA_INVOKE`.

Extract the `FunctionName` parameter:
- String literal → `invokes_functions: ["literal-name"]`
- `System.getenv("VAR")` → `invokes_functions: ["DYNAMIC:VAR"]`
- Computed → `invokes_functions: ["DYNAMIC"]`, note variable name

### 2.5 Client Instantiation Location

Where the client is created affects Dimension 5 (static state):

```java
private static final S3Client s3 = S3Client.create();  // → warm_start_clients: ["S3Client"]
public Void handleRequest(...) { S3Client s3 = S3Client.create(); }  // → not warm-start
public Handler(S3Client s3) { this.s3 = s3; }  // → injected_clients: ["S3Client"]
```

### 2.6 S3 Presigned URL (distinct pattern)

`S3Presigner.presignPutObject(...)` or `s3.generatePresignedUrl(...)` means no actual S3 operation occurs in this Lambda — it generates a URL for a client to use.
→ `service: S3, pattern: presigned_url`, flag `S3_PRESIGNED_URL`

---

## Section 3 — External HTTP and Non-AWS Integrations

**When:** After AWS SDK detection. Identifies all outbound non-AWS HTTP calls.  
**Official docs to fetch if needed:**
- Java HttpClient (JDK 11+): https://docs.oracle.com/en/java/api/java.net.http/java/net/http/HttpClient.html
- Apache HttpClient 5: https://hc.apache.org/httpcomponents-client-5.2.x/
- OkHttp: https://square.github.io/okhttp/
- Spring WebClient: https://docs.spring.io/spring-framework/reference/web/webflux-webclient.html
- Elasticsearch Java client: https://www.elastic.co/guide/en/elasticsearch/client/java-api-client/current/

### 3.1 Detection by Import Root

| Import root | `http_client` |
|---|---|
| `java.net.http.*` | `JavaHttpClient` |
| `java.net.HttpURLConnection` | `HttpURLConnection` |
| `org.apache.http.*` | `ApacheHttpClient` |
| `org.apache.hc.*` | `ApacheHttpClient5` |
| `okhttp3.*` | `OkHttp` |
| `org.springframework.web.client.*` | `RestTemplate` |
| `org.springframework.web.reactive.function.client.*` | `WebClient` |
| `feign.*` | `Feign` |
| `retrofit2.*` | `Retrofit` |
| `io.grpc.*` | `gRPC` — flag `GRPC_INTEGRATION` |

### 3.2 Elasticsearch — Flag Explicitly (Critical for This Migration)

| Import root | `http_client` | Flag |
|---|---|---|
| `org.elasticsearch.client.*` | `ElasticsearchRestClient` or `ElasticsearchHighLevelClient` | `ELASTICSEARCH_INTEGRATION` |
| `co.elastic.clients.*` | `ElasticsearchJavaClient` | `ELASTICSEARCH_INTEGRATION` |
| `org.opensearch.client.*` | `OpenSearchRestClient` | `OPENSEARCH_INTEGRATION` |
| HTTP calls to port 9200 or paths `/_doc`, `/_search` | `ElasticsearchHTTP` | `ELASTICSEARCH_INTEGRATION` |

> `ELASTICSEARCH_INTEGRATION` = no-touch in Azure. The HTTP call is identical. Only the function trigger changes.

### 3.3 Endpoint Extraction

For every HTTP client:
- String literal URL → `endpoint_refs: ["https://..."]`
- Env var reference → `endpoint_refs: ["env:VAR_NAME"]`
- `localhost:2772` → `endpoint_refs: ["localhost:2772"]`, flag `LAMBDA_EXTENSION_CONFIG`
- Dynamic/built → `endpoint_refs: ["DYNAMIC"]`, note variable names

### 3.4 Async and TLS Signals

- `sendAsync(...)` / `WebClient` subscription / `CompletableFuture` → `async: true`, flag `ASYNC_HTTP_CALL`
- Custom truststore / `KeyStore.getInstance(...)` → flag `CUSTOM_TLS_REVIEW`
- Trust-all `X509TrustManager` with empty methods → flag `INSECURE_TLS_CRITICAL`
- `KeyManagerFactory` usage → flag `MTLS_REVIEW`

---

## Section 4 — Configuration and Secrets

**When:** Read every method in the class, not just `handleRequest`.  
**Official docs to fetch if needed:**
- Lambda environment variables: https://docs.aws.amazon.com/lambda/latest/dg/configuration-envvars.html
- SSM Parameter Store Java: https://docs.aws.amazon.com/systems-manager/latest/userguide/integration-ps-secretsmanager.html
- Secrets Manager Java: https://docs.aws.amazon.com/secretsmanager/latest/userguide/retrieving-secrets_cache-java.html
- AppConfig: https://docs.aws.amazon.com/appconfig/latest/userguide/appconfig-retrieving-the-configuration.html

**Critical rule: record key names only — never values, even if visible as defaults.**

### 4.1 Environment Variables

```java
System.getenv("KEY")                    → env_vars: ["KEY"]
System.getenv().getOrDefault("K", "v")  → env_vars: ["K"]
@Value("${KEY}")                        → env_vars: ["KEY"]
@ConfigProperty(name = "key")           → env_vars: ["key"]
```

### 4.2 AWS SSM Parameter Store

```java
ssmClient.getParameter(...name("/path/to/param")...)   → secrets_access: [{type: SSM, name: "/path/to/param"}]
ssmClient.getParameter(...name(System.getenv("P"))...) → secrets_access: [{type: SSM, name: "DYNAMIC:P"}]
ssmClient.getParametersByPath(...path("/prefix/")...)  → secrets_access: [{type: SSM, name: "PATH:/prefix/"}]
```

### 4.3 Secrets Manager

```java
secretsClient.getSecretValue(...secretId("name")...) → secrets_access: [{type: SecretsManager, name: "name"}]
```

### 4.4 AppConfig and Bundled Config

```java
AppConfigDataClient.startConfigurationSession(...)  → config_sources: [{type: AppConfig, ...}], flag APPCONFIG_DYNAMIC_CONFIG
getClass().getResourceAsStream("/config.properties") → flag BUNDLED_CONFIG_FILE
new FileInputStream("/tmp/...")                      → flag FILESYSTEM_CONFIG
localhost:2772 HTTP call                             → flag LAMBDA_EXTENSION_CONFIG
```

### 4.5 Security Flags

- Env var name contains (case-insensitive): `SECRET`, `KEY`, `TOKEN`, `PASSWORD`, `CREDENTIAL`, `CERT`, `PRIVATE` → flag `SENSITIVE_ENV_VAR:<name>`
- Possible hardcoded credential (long alphanumeric string in credential-named field) → flag `POSSIBLE_HARDCODED_SECRET:<file>:<line>` — line only, never reproduce value
- `IamClient` or `AmazonIdentityManagement` usage → flag `IAM_SDK_USAGE`

---

## Section 5 — Runtime and Code Patterns

**When:** Structural analysis of how the code is written, not what it calls.  
**Official docs to fetch if needed:**
- Lambda best practices: https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html
- Lambda context object: https://docs.aws.amazon.com/lambda/latest/dg/java-context.html
- Lambda SnapStart: https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html
- Lambda X-Ray tracing: https://docs.aws.amazon.com/lambda/latest/dg/java-tracing.html
- Lambda Powertools for Java: https://docs.powertools.aws.dev/lambda/java/

### 5.1 Static State

Any `static` field declared outside a method body → `has_static_state: true`

```java
private static final S3Client s3 = S3Client.create();  // → warm_start_clients: ["S3Client"]
private static Map<String,String> cache = new HashMap<>(); // → static_fields: [{name,type}]
static { System.setProperty(...); }                    // → has_static_initialiser: true
```

Flag `STATIC_STATE_REVIEW` for any non-final mutable statics.

### 5.2 Error Handling

```java
catch (S3Exception e) { ... }          → exception_types_caught: ["S3Exception"]
catch (Exception e) { /* ignored */ }  → flag SWALLOWED_EXCEPTION:<file>:<line>
throw new RuntimeException(...)        → exception_types_thrown: ["RuntimeException"]
```

Retry patterns (flag `has_explicit_retry: true` for any of these):
- Manual for-loop with catch + sleep → `retry_pattern: manual_loop`
- `@Retryable` (Spring) → `retry_pattern: Spring_Retryable`
- `Retry.ofDefaults` (Resilience4j) → `retry_pattern: Resilience4j`
- `.retryPolicy(...)` on SDK client builder → `retry_pattern: SDK_retry_policy`

### 5.3 Logging

| Import | `logging_library` |
|---|---|
| `com.amazonaws.services.lambda.runtime.LambdaLogger` | `LambdaLogger` |
| `org.slf4j.Logger` | `SLF4J` |
| `org.apache.logging.log4j.Logger` | `Log4j2` |
| `java.util.logging.Logger` | `JUL` |
| `software.amazon.lambda.powertools.logging` | `LambdaPowertools` |

Log levels called (record which of these appear): `debug()`, `info()`, `warn()`, `error()` → `log_levels[]`

Structured logging: `MDC.put(...)` → `structured_logging: true, method: MDC` | JSON string in log calls → `method: manual_json`

### 5.4 Observability

```java
import com.amazonaws.xray.AWSXRay           → tracing: XRay
AWSXRay.beginSubsegment("name")             → xray_subsegments: ["name"]
@XRayEnabled                                → tracing: XRay, method: annotation
cloudWatchClient.putMetricData(... .namespace("A").metricData(... .metricName("B")...)) 
                                            → custom_metrics: [{namespace: "A", metric_name: "B"}]
```

### 5.5 Context Object Usage

```java
context.getRemainingTimeInMillis()  → context_usage: ["remainingTime"], flag TIMEOUT_AWARE_LOGIC
context.getAwsRequestId()           → context_usage: ["requestId"]
context.getFunctionName()           → context_usage: ["functionName"]
context.getMemoryLimitInMB()        → context_usage: ["memoryLimit"]
```

### 5.6 SnapStart

```java
import org.crac.Resource
implements ... Resource { beforeCheckpoint(...) afterRestore(...) }
→ snapstart: true, crac_hooks: ["beforeCheckpoint", "afterRestore"]
→ flag SNAPSTART_DETECTED, HUMAN_REVIEW_REQUIRED
```

Docs: https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html  
Azure Functions has no SnapStart equivalent. CRaC hooks have no counterpart.

### 5.7 VPC Signals from Code

```java
DriverManager.getConnection("jdbc:mysql://...")  → flag VPC_LIKELY, service: RDS_JDBC
new Jedis(...)                                   → flag VPC_LIKELY, service: ElastiCache_Redis
```

VPC config is resolved by the Context Analyst from IaC files. Code signals here are supplementary.

### 5.8 Thread Pool and Parallelism

```java
Executors.newFixedThreadPool(...)   → has_thread_pool: true, flag THREAD_POOL_REVIEW
items.parallelStream().forEach(...) → has_parallel_stream: true, flag PARALLEL_STREAM_REVIEW
```

---

## Section 6 — Build and Deployment (pom.xml)

**When:** If pom.xml is provided alongside the Java file.  
**Official docs to fetch if needed:**
- Maven POM reference: https://maven.apache.org/pom.html
- AWS Lambda Java libs (all library versions): https://github.com/aws/aws-lambda-java-libs
- AWS SDK v3 BOM versions: https://github.com/aws/aws-sdk-java-v2/blob/master/bom/pom.xml
- Maven Central search: https://central.sonatype.com/

**Skip all `<scope>test</scope>` dependencies entirely.**

### 6.1 Java Version

```xml
<java.version>17</java.version>                    → java_version: "17"
<maven.compiler.source>11</maven.compiler.source>  → java_version: "11"
```

### 6.2 AWS Dependency Rules

Lambda runtime (always present, expected):
- `com.amazonaws:aws-lambda-java-core` → `lambda_runtime: core`
- `com.amazonaws:aws-lambda-java-events` → `lambda_events_version: <version>` (important — determines available event classes)
- `com.amazonaws:aws-lambda-java-log4j2` → `logging: lambda_log4j2`

SDK version rule (cross-check with Section 2):
- `groupId: com.amazonaws` → `sdk: v1`
- `groupId: software.amazon.awssdk` → `sdk: v3`
- Both → `sdk: MIXED`

For v3: artifactId IS the service name: `software.amazon.awssdk:{service}` → service: `{service}`  
For v1: parse from artifactId: `aws-java-sdk-{service}` → service: `{service}`

Powertools:
- `software.amazon.lambda.powertools:powertools-{feature}` → `framework_features: ["{feature}"]`

### 6.3 Framework Detection

```
org.springframework.boot → framework: Spring_Boot
io.quarkus              → framework: Quarkus
io.micronaut            → framework: Micronaut
```

### 6.4 VPC-Implying Dependencies (flag `VPC_LIKELY` for any of these)

```
mysql:mysql-connector-java         → service: RDS_MySQL
com.mysql:mysql-connector-j        → service: RDS_MySQL
org.postgresql:postgresql          → service: RDS_PostgreSQL
com.microsoft.sqlserver:mssql-jdbc → service: RDS_SQLServer
redis.clients:jedis                → service: ElastiCache_Redis
io.lettuce:lettuce-core            → service: ElastiCache_Redis
```

### 6.5 Build Plugins

```xml
maven-shade-plugin           → packaging: shade_fat_jar
spring-boot-maven-plugin     → packaging: spring_boot_jar
quarkus-maven-plugin         → packaging: quarkus_fast_jar
native-maven-plugin          → packaging: native_image, flag NATIVE_IMAGE_REVIEW
```

### 6.6 Multi-Module Project

```xml
<packaging>pom</packaging><modules>...</modules>
→ project_type: multi_module, sibling_modules: [...], flag MULTI_MODULE_PROJECT
```

---

## Section 7 — Functional and Technical Summary (Dimension 10)

**When:** Last. Synthesise after all other dimensions are complete.  
**This section has no external documentation — it is reasoning guidance.**

### 7.1 Technical Summary (2–4 sentences)

Describe the **mechanics**. Present tense. Name actual services and operations.

Template: `[Trigger type] triggers [class], which reads [input] from [source]. It [transforms/computes] [what]. Results are written to [outputs]. [Error/retry behaviour if notable.]`

Good: "An SQS-triggered function that reads one order record per message, retrieves the full order from DynamoDB using the message body's order ID, enriches it with pricing via POST to (env: PRICING_API_URL), then writes to S3 (env: OUTPUT_BUCKET) and notifies via SNS."

Bad: "Processes messages from a queue and stores results." (too vague)  
Bad: "Handles the e-commerce checkout flow." (invented context not in code)

### 7.2 Functional Summary (1–2 sentences)

Describe the **business purpose**. No technical terms. No AWS service names.

Template: `[Business process] by [plain description], resulting in [business outcome].`

Good: "Processes incoming customer orders by enriching them with current pricing and storing the result for downstream reporting."

### 7.3 Evidence Priority

1. Javadoc on the class
2. Comments on `handleRequest`
3. Method names (`processOrder`, `enrichRecord`, `indexDocument`)
4. Variable names (`orderId`, `pricingResponse`)
5. Log message strings
6. Service + operation combinations
7. Event type + endpoint URLs

### 7.4 When Purpose is Unclear

```
"Purpose unclear from code analysis — insufficient signal.
Evidence found: [trigger], [services used], [operations found].
Human review recommended."
```

---

## Flags Reference

All flags produced by this skill:

| Flag | Meaning |
|---|---|
| `STREAM_HANDLER_REVIEW` | RequestStreamHandler — trigger type cannot be determined from code |
| `GENERIC_EVENT_TYPE` | Handler uses Map<String,Object> — ambiguous trigger |
| `CUSTOM_WRAPPER_EVENT` | Custom event class wraps an AWS event — ambiguous trigger |
| `EVENTBRIDGE_NO_TYPED_CLASS` | EventBridge trigger detected without typed class |
| `DOCUMENTDB_TRIGGER` | DocumentDB trigger detected without typed class |
| `MIXED_SDK_VERSIONS` | Both com.amazonaws and software.amazon.awssdk present |
| `SDK_V1_EOL` | SDK v1 in use (end-of-support Dec 2025) |
| `CROSS_LAMBDA_INVOKE` | This Lambda directly invokes another Lambda |
| `S3_PRESIGNED_URL` | Lambda generates presigned URL, does not write S3 directly |
| `ELASTICSEARCH_INTEGRATION` | ES client or HTTP to ES detected — no-touch in Azure |
| `OPENSEARCH_INTEGRATION` | OpenSearch client detected |
| `GRPC_INTEGRATION` | gRPC client detected — no direct Azure Functions binding |
| `ASYNC_HTTP_CALL` | Async HTTP — may complete after Lambda returns |
| `CUSTOM_TLS_REVIEW` | Custom truststore or TLS config |
| `INSECURE_TLS_CRITICAL` | Trust-all TLS — security risk |
| `MTLS_REVIEW` | Mutual TLS detected |
| `LAMBDA_EXTENSION_CONFIG` | Config fetched from Lambda extension sidecar (localhost:2772) |
| `APPCONFIG_DYNAMIC_CONFIG` | AWS AppConfig used for feature flags |
| `BUNDLED_CONFIG_FILE` | Config baked into JAR — may contain env-specific values |
| `FILESYSTEM_CONFIG` | Config read from /tmp — ephemeral, does not persist |
| `SENSITIVE_ENV_VAR:<name>` | Env var name suggests it holds a secret |
| `POSSIBLE_HARDCODED_SECRET:<file>:<line>` | Possible credential in code |
| `IAM_SDK_USAGE` | Lambda manipulates IAM — high risk |
| `STATIC_STATE_REVIEW` | Non-final mutable static fields — shared across invocations |
| `SWALLOWED_EXCEPTION:<file>:<line>` | Exception caught and silently ignored |
| `TIMEOUT_AWARE_LOGIC` | Code checks remaining execution time |
| `SNAPSTART_DETECTED` | CRaC hooks found — no Azure equivalent |
| `HUMAN_REVIEW_REQUIRED` | Accompanies SNAPSTART_DETECTED |
| `VPC_LIKELY` | Code or pom.xml signals suggest VPC placement |
| `THREAD_POOL_REVIEW` | Thread pool or executor in handler |
| `PARALLEL_STREAM_REVIEW` | parallelStream() in handler |
| `NATIVE_IMAGE_REVIEW` | GraalVM native image build — no standard Azure equivalent |
| `MULTI_MODULE_PROJECT` | Multi-module Maven project |
| `POSSIBLE_SHARED_LAYER` | Sibling module may be a Lambda Layer |
| `RDS_DATA_API_REVIEW` | RDS Data API (SQL over HTTP) — needs Azure mapping assessment |
