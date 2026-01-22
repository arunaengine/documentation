# Data Transformation Workflows – Automated File Transformation with Webhooks

### Understanding Webhooks: Concepts, Patterns, and Implementation

A comprehensive guide to webhook architecture and implementation, illustrated with the ABCD2BioSchema Transformation Service.

---

## Table of Contents

1. [Introduction to Webhooks](#1-introduction-to-webhooks)
2. [Webhook Anatomy](#2-webhook-anatomy)
3. [Webhook Lifecycle](#3-webhook-lifecycle)
4. [Design Patterns](#4-design-patterns)
5. [Implementation Guide](#5-implementation-guide)
6. [Error Handling Strategies](#6-error-handling-strategies)
7. [Security Considerations](#7-security-considerations)
8. [Practical Example: ABCD2BioSchema Service](#8-practical-example-abcd2bioschema-service)
9. [Testing and Debugging](#9-testing-and-debugging)
10. [Checklist for Production](#10-checklist-for-production)

---

## 1. Introduction to Webhooks

### What Is a Webhook?

A webhook is an HTTP-based callback mechanism that enables real-time communication between systems. 
Unlike traditional APIs where a client repeatedly polls a server for updates, webhooks invert this relationship: 
the server proactively notifies the client when relevant events occur.

Think of webhooks as "reverse APIs" or "HTTP push notifications." When a specified event happens in a source system, 
that system makes an HTTP request to a pre-configured URL, delivering event data to the receiving service.

### Webhooks vs. Polling

| Aspect             | Polling                             | Webhooks                        |
|--------------------|-------------------------------------|---------------------------------|
| **Direction**      | Client pulls from server            | Server pushes to client         |
| **Latency**        | Depends on poll interval            | Near real-time                  |
| **Resource Usage** | Continuous requests, even when idle | Requests only when events occur |
| **Complexity**     | Simpler initial setup               | Requires endpoint management    |
| **Reliability**    | Client controls retry logic         | Requires callback confirmation  |

### Common Use Cases

Webhooks excel in scenarios requiring real-time reactions to events:

- **Data Transformation**: Converting files between formats when uploaded
- **Notification Systems**: Sending alerts when specific conditions are met
- **Integration Pipelines**: Connecting disparate systems without tight coupling
- **Automation Workflows**: Triggering actions based on state changes
- **Audit and Logging**: Recording events as they happen

### Illustrative Example: ABCD2BioSchema

The ABCD2BioSchema Transformation Service demonstrates a practical webhook implementation. When a user uploads an ABCD 
(Access to Biological Collection Data) XML file to the Aruna data platform, a webhook triggers the service to 
automatically transform the metadata into BioSchema-compliant JSON format. This happens without any manual 
intervention, the moment the upload completes.

---

## 2. Webhook Anatomy

A webhook system consists of three primary components working in concert.

### 2.1 Trigger Events

Triggers define **when** a webhook fires. They are conditions evaluated by the source system.

**Common trigger types include:**

- **Resource lifecycle events**: Created, updated, deleted, archived
- **State transitions**: Status changes, workflow progression
- **Content events**: Upload completed, modification detected
- **Conditional triggers**: Label added, threshold exceeded

**Example from Aruna (full list can be found under [Getting Started > 13 | Hooks](../get_started/basic_usage/13_How-To-Hooks.md)):**

| Trigger Type                    | Description                             |
|---------------------------------|-----------------------------------------|
| `TRIGGER_TYPE_OBJECT_FINISHED`  | Fires when an object upload completes   |
| `TRIGGER_TYPE_LABEL_ADDED`      | Fires when a specific label is attached |
| `TRIGGER_TYPE_RESOURCE_CREATED` | Fires when any resource is created      |

Triggers can be combined with **filters** to narrow scope:

```
Trigger: OBJECT_FINISHED
Filter: Label key matches "^ABCD$"
Result: Only fires for objects with the ABCD label
```

### 2.2 Payload Structure

The payload is the data package sent when a webhook fires. Well-designed payloads contain everything the receiving 
service needs to process the event.

**Essential payload components:**

| Component            | Purpose                                       |
|----------------------|-----------------------------------------------|
| **Event identifier** | Unique ID for tracking and idempotency        |
| **Event type**       | What happened (created, updated, etc.)        |
| **Resource data**    | The affected entity's current state           |
| **Credentials**      | Temporary tokens for authenticated operations |
| **Metadata**         | Timestamps, versions, context                 |

**ABCD2BioSchema payload structure:**

```
Hook Payload
├── hook_id          → Unique webhook invocation identifier
├── secret           → Verification token for authenticity
├── object           → Complete resource metadata
│   ├── id           → Resource identifier
│   ├── name         → Filename
│   ├── content_len  → File size
│   ├── key_values   → Labels and metadata
│   ├── relations    → Links to other resources
│   └── ...
├── download         → Presigned URL to retrieve file content
├── access_key       → Temporary S3 access credential
├── secret_key       → Temporary S3 secret credential
├── token            → API bearer token for callbacks
└── pubkey_serial    → Key identifier for verification
```

### 2.3 Callback Mechanism

Callbacks are responses sent from the webhook receiver back to the source system. They serve two purposes: 
acknowledging receipt and reporting processing results.

**Callback types:**

1. **Immediate acknowledgment**: HTTP response confirming receipt (synchronous)
2. **Status callback**: Separate request reporting final processing status (asynchronous)

**Why callbacks matter:**

- The source system knows the event was received
- Processing results can update the original resource
- Errors can be logged and surfaced to users
- Retry logic can be triggered on failures

---

## 3. Webhook Lifecycle

Understanding the complete lifecycle of a webhook invocation is essential for building robust services.

### 3.1 Registration Phase

Before webhooks can fire, they must be registered with the source system.

**Registration specifies:**
- **Endpoint URL**: Where to send webhook requests
- **HTTP method**: Usually POST
- **Trigger conditions**: Events and filters
- **Scope**: Which resources to monitor
- **Credentials**: Authentication for the callback

**Example registration (Aruna):**

```bash
curl -X POST 'https://api.aruna-engine.org/v2/hooks' \
  -H "Authorization: Bearer ${TOKEN}" \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "ABCD2BioSchema",
    "trigger": {
      "triggerType": "TRIGGER_TYPE_OBJECT_FINISHED",
      "filters": [{
        "keyValue": {
          "key": "^ABCD$",
          "value": ".*",
          "variant": "KEY_VALUE_VARIANT_LABEL"
        }
      }]
    },
    "hook": {
      "externalHook": {
        "url": "https://transform-service.example.com/transform/url",
        "method": "METHOD_POST"
      }
    },
    "projectIds": ["project-123"],
    "timeout": "1735689600000"
  }'
```

### 3.2 Event Detection

When a user action or system process satisfies the trigger conditions, the source system initiates the webhook.

**Detection flow:**

```
User uploads file with ABCD label
               │
               ▼
┌─────────────────────────────┐
│  Source System (Aruna)      │
│  ┌───────────────────────┐  │
│  │  Event: Object        │  │
│  │  finished uploading   │  │
│  └───────────┬───────────┘  │
│              ▼              │
│  ┌───────────────────────┐  │
│  │  Check registered     │  │
│  │  webhooks             │  │
│  └───────────┬───────────┘  │
│              ▼              │
│  ┌───────────────────────┐  │
│  │  Evaluate filters:    │  │
│  │  Label "ABCD" exists? │  │
│  └───────────┬───────────┘  │
│              ▼              │
│         Filter passes       │
└──────────────┬──────────────┘
               │
               ▼
       Webhook triggered
```

### 3.3 Payload Delivery

The source system constructs and sends the webhook payload.

**Delivery characteristics:**

- **Method**: HTTP POST (standard)
- **Content-Type**: application/json (typical)
- **Timeout**: Source system waits for initial acknowledgment
- **Retries**: May retry on connection failures

**ABCD2BioSchema receives:**

```
POST /transform/url HTTP/1.1
Host: transform-service.example.com
Content-Type: application/json

{
  "hook_id": "hook-abc-123",
  "object": {
    "object": {
      "id": "obj-xyz-789",
      "name": "specimen_data.xml",
      "content_len": 2048576,
      "key_values": [
        {"key": "ABCD", "value": "true", "variant": 1}
      ],
      ...
    }
  },
  "download": "https://s3.aruna.example.com/bucket/obj-xyz-789?...",
  "secret": "webhook-secret-token",
  "token": "temporary-api-token",
  ...
}
```

### 3.4 Processing

The webhook service receives the payload and executes its business logic.

**Processing stages in ABCD2BioSchema:**

```
┌─────────────────────────────────────────────────────────────┐
│                      Webhook Service                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. RECEIVE & VALIDATE                                      │
│     ├── Parse JSON payload                                  │
│     ├── Verify webhook secret                               │
│     └── Validate required fields present                    │
│                      │                                      │
│                      ▼                                      │
│  2. DOWNLOAD SOURCE                                         │
│     ├── Use presigned URL from payload                      │
│     └── Retrieve ABCD XML file                              │
│                      │                                      │
│                      ▼                                      │
│  3. TRANSFORM                                               │
│     ├── Send XML to GFBio transformation API                │
│     └── Receive BioSchema JSON result                       │
│                      │                                      │
│                      ▼                                      │
│  4. UPLOAD RESULT                                           │
│     ├── Use temporary S3 credentials from payload           │
│     ├── Upload JSON to storage                              │
│     └── Create new object with metadata                     │
│                      │                                      │
│                      ▼                                      │
│  5. ESTABLISH RELATIONSHIPS                                 │
│     ├── Link new object to source (ORIGIN relation)         │
│     └── Preserve parent hierarchy                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.5 Status Callback

After processing completes (success or failure), the service notifies the source system.

**Callback request structure:**

```
gRPC: HookCallbackRequest
├── hook_id        → Matches original webhook
├── secret         → Proves authenticity
├── object_id      → Affected resource
├── pubkey_serial  → Key verification
└── status         → Processing result
    └── Finished
        ├── add_key_values    → Labels to add (e.g., "BioSchema: true")
        └── remove_key_values → Labels to remove
```

**Success callback (ABCD2BioSchema):**

```rust
Status::Finished(Finished {
    add_key_values: vec![
        KeyValue {
            key: "TRANSFORMED_BY_GFBIO".to_string(),
            value: "success".to_string(),
            variant: KeyValueVariant::Label as i32,
        },
    ],
    remove_key_values: vec![],
})
```

**Error callback:**

```rust
Status::Finished(Finished {
    add_key_values: vec![
        KeyValue {
            key: "Error".to_string(),
            value: "Transformation failed: invalid XML format".to_string(),
            variant: KeyValueVariant::Label as i32,
        },
    ],
    remove_key_values: vec![],
})
```

### 3.6 Complete Lifecycle Diagram

```
┌──────────────┐         ┌──────────────────┐         ┌──────────────┐
│              │         │                  │         │              │
│    User      │         │  Source System   │         │   Webhook    │
│              │         │    (Aruna)       │         │   Service    │
│              │         │                  │         │              │
└──────┬───────┘         └────────┬─────────┘         └──────┬───────┘
       │                          │                          │
       │  1. Upload file          │                          │
       │  with ABCD label         │                          │
       │─────────────────────────►│                          │
       │                          │                          │
       │                          │  2. Event detected       │
       │                          │  Webhook triggered       │
       │                          │                          │
       │                          │  3. POST /transform/url  │
       │                          │─────────────────────────►│
       │                          │                          │
       │                          │  4. HTTP 200 (accepted)  │
       │                          │◄─────────────────────────│
       │                          │                          │
       │                          │                          │ 5. Download
       │                          │◄ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │    source
       │                          │                          │
       │                          │                          │ 6. Transform
       │                          │                          │    data
       │                          │         ┌────────────────┤
       │                          │         │ GFBio API      │
       │                          │         └────────────────┤
       │                          │                          │
       │                          │                          │ 7. Upload
       │                          │◄ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │    result
       │                          │                          │
       │                          │  8. gRPC: HookCallback   │
       │                          │◄─────────────────────────│
       │                          │                          │
       │  9. Labels updated       │                          │
       │  (TRANSFORMED_BY_GFBIO)  │                          │
       │◄─────────────────────────│                          │
       │                          │                          │
       ▼                          ▼                          ▼
```

---

## 4. Design Patterns

Effective webhook services follow established patterns that ensure reliability, maintainability, and scalability.

### 4.1 Asynchronous Processing

**Pattern**: Acknowledge receipt immediately, process in the background.

**Why it matters**: Source systems typically have short timeouts for webhook delivery. Long processing times 
can cause timeout failures, leading to retries and duplicate processing.

**Implementation:**

```rust
pub async fn webhook_handler(hook: Hook) -> Result<Json<Response>> {
    // Validate quickly
    validate_hook(&hook)?;
    
    // Acknowledge immediately
    let job_id = generate_job_id();
    
    // Spawn background task
    tokio::spawn(async move {
        if let Err(e) = process_webhook(hook.clone()).await {
            let _ = send_error_callback(&hook, e.to_string()).await;
        }
    });
    
    // Return accepted status
    Ok(Json(Response {
        status: "accepted",
        job_id,
    }))
}
```

### 4.2 Idempotency

**Pattern**: Ensure the same webhook can be processed multiple times without adverse effects.

**Why it matters**: Network issues, retries, and duplicate deliveries are common. Without idempotency, the same 
event could create duplicate records or trigger duplicate side effects.

**Implementation strategies:**

1. **Track processed webhook IDs:**
```rust
async fn handle_webhook(hook: Hook) -> Result<()> {
    // Check if already processed
    if is_processed(&hook.hook_id).await? {
        info!("Hook {} already processed", hook.hook_id);
        return Ok(());
    }
    
    // Process
    process(hook.clone()).await?;
    
    // Mark complete
    mark_processed(&hook.hook_id).await?;
    Ok(())
}
```

2. **Use unique constraints on output:**
```rust
// Generate deterministic output key based on input
let output_key = format!("{}-bioschema.json", source_object_id);
// If object already exists, skip or update
```

### 4.3 Graceful Degradation

**Pattern**: Handle partial failures without losing the entire operation.

**Why it matters**: Multi-step processes can fail at any point. Losing progress forces complete restarts 
and wastes resources.

**Implementation:**

```rust
pub async fn handle_url_transformation(hook: Hook) -> Result<Json<JobResponse>> {
    // Step 1: Transform (can fail)
    let result = match send_gfbio_request(&download_url).await {
        Ok(response) => response,
        Err(e) => {
            // Report failure but don't crash
            let _ = send_error_callback(&hook, format!("Transform failed: {}", e)).await;
            return Err(transform_error(e));
        }
    };
    
    // Step 2: Upload (can fail)
    let object_id = match fetch_result_and_upload(&hook, &result).await {
        Ok(id) => id,
        Err(e) => {
            // Report upload failure
            let _ = send_error_callback(&hook, format!("Upload failed: {}", e)).await;
            return Err(upload_error(e));
        }
    };
    
    // Step 3: Callback (best effort)
    if let Err(e) = send_success_callback(&hook, object_id.clone()).await {
        // Log but don't fail - the work is done
        error!("Callback failed: {}", e);
    }
    
    Ok(Json(JobResponse { ... }))
}
```

### 4.4 State Externalization

**Pattern**: Store processing state externally so it survives restarts.

**Why it matters**: Long-running processes or service restarts shouldn't lose progress or state.

**Implementation approaches:**

- **Job tracking**: Store job status in a database
- **Checkpointing**: Save progress at each major step
- **Stateless design**: Derive all needed state from the payload

**ABCD2BioSchema job tracking:**

```rust
pub struct Job {
    pub job_id: String,
    pub transformation_id: String,
    pub status: String,              // "pending", "processing", "complete", "failed"
    pub start_time: DateTime<FixedOffset>,
    pub finish_time: Option<DateTime<FixedOffset>>,
    pub result_file: String,
    pub input_file_url: String,
    // ...
}
```

### 4.5 Separation of Concerns

**Pattern**: Isolate webhook handling from business logic.

**Why it matters**: Clean separation enables testing, reuse, and independent evolution of components.

**ABCD2BioSchema structure:**

```
src/
├── main.rs       → Application bootstrap, server setup
├── service.rs    → HTTP endpoint handlers (thin layer)
├── webhook.rs    → Core transformation logic
├── models.rs     → Data structures
└── job.rs        → Job state management
```

**Service layer (thin):**

```rust
// service.rs - Only handles HTTP concerns
pub async fn url_transform(
    State(state): State<Arc<Handler>>,
    Json(request): Json<Hook>,
) -> Result<Json<JobResponse>, (StatusCode, Json<ErrorResponse>)> {
    debug!("/transform/url endpoint called");
    // Delegate to business logic
    state.webhook.handle_url_transformation(request).await
}
```

---

## 5. Implementation Guide

This section provides practical guidance for implementing webhook services.

### 5.1 Receiving Webhooks

**Requirements:**
- HTTP server capable of handling POST requests
- JSON parsing
- Input validation

**Framework setup (Axum example):**

```rust
pub fn create_router(state: Arc<Handler>) -> Router {
    Router::new()
        .route("/health", get(health_check))
        .route("/transform/url", post(url_transform))
        .route("/job/{job_id}", get(get_job_status))
        .layer(CorsLayer::permissive())
        .with_state(state)
}
```

### 5.2 Validating Payloads

**Always validate:**
- Required fields are present
- Secret matches expected value
- Resource type is expected

**Validation example:**

```rust
pub async fn handle_url_transformation(hook: Hook) -> Result<...> {
    // Validate download URL exists
    let download_url = hook.download.clone().ok_or_else(|| {
        let error_msg = "No download URL provided in hook";
        // Send error callback
        let _ = send_error_callback(&hook, error_msg.to_string()).await;
        (StatusCode::BAD_REQUEST, Json(ErrorResponse {
            error: "missing_download_url".to_string(),
            message: error_msg.to_string(),
        }))
    })?;
    
    // Validate object type
    let object_id = match &hook.object {
        Resource::Object(r) => r.id.clone(),
        _ => {
            return Err("Invalid resource type: expected Object");
        }
    };
    
    // Continue processing...
}
```

### 5.3 Processing Data

**Common processing patterns:**

1. **File download:**
```rust
let response = client
    .get(&download_url)
    .header("User-Agent", "WebhookService/1.0")
    .send()
    .await?;

if !response.status().is_success() {
    return Err(format!("Download failed: {}", response.status()).into());
}

let content = response.bytes().await?;
```

2. **External API calls:**
```rust
let transform_url = format!(
    "{}/transform?transformation={}&input_file_url={}",
    api_base_url, transformation_id, encode(&input_url)
);

let result = client.get(&transform_url).send().await?;
let json: Value = result.json().await?;
```

3. **Result upload with credentials from payload:**
```rust
// Extract credentials from hook payload
let credentials = Credentials::new(
    hook.access_key.clone().ok_or("No access key")?,
    hook.secret_key.clone().ok_or("No secret key")?,
    None, None, "WEBHOOK_SERVICE",
);

// Configure S3 client
let s3_client = aws_sdk_s3::Client::from_conf(
    aws_sdk_s3::config::Builder::from(&config)
        .credentials_provider(credentials)
        .endpoint_url(endpoint)
        .build()
);

// Upload
s3_client
    .put_object()
    .bucket(bucket)
    .key(key)
    .body(data.into())
    .send()
    .await?;
```

### 5.4 Sending Callbacks

**Callback implementation:**

```rust
async fn send_hook_callback(
    &self,
    hook: &Hook,
    status: Status,
) -> Result<(), Box<dyn Error + Send + Sync>> {
    let callback_request = HookCallbackRequest {
        secret: hook.secret.clone(),
        hook_id: hook.hook_id.clone(),
        object_id: extract_object_id(&hook.object),
        pubkey_serial: hook.pubkey_serial,
        status: Some(status),
        ..Default::default()
    };

    let interceptor = ClientInterceptor {
        api_token: hook.token.clone(),
    };

    let mut hook_client =
        HooksServiceClient::with_interceptor(self.channel.clone(), interceptor);

    hook_client.hook_callback(tonic::Request::new(callback_request)).await?;
    Ok(())
}
```

---

## 6. Error Handling Strategies

Robust error handling is critical for webhook reliability.

### 6.1 Error Categories

| Category              | Examples                                | Handling Strategy                       |
|-----------------------|-----------------------------------------|-----------------------------------------|
| **Validation errors** | Missing fields, invalid format          | Reject immediately, send error callback |
| **Transient errors**  | Network timeout, rate limiting          | Retry with backoff                      |
| **Permanent errors**  | Invalid credentials, resource not found | Fail fast, send error callback          |
| **Partial failures**  | Upload succeeds but callback fails      | Log and continue                        |

### 6.2 Always Send Callbacks

**Critical principle**: The source system should always know the outcome.

```rust
match process_webhook(hook.clone()).await {
    Ok(result) => {
        send_success_callback(&hook, result.object_id).await?;
    }
    Err(e) => {
        // ALWAYS notify on error
        send_error_callback(&hook, e.to_string()).await?;
        return Err(e);
    }
}
```

### 6.3 Error Labeling

Adding errors as labels makes them visible to users:

```rust
async fn send_error_callback(&self, hook: &Hook, error_message: String) {
    let status = Status::Finished(Finished {
        add_key_values: vec![KeyValue {
            key: "Error".to_string(),
            value: error_message,
            variant: KeyValueVariant::Label as i32,
        }],
        remove_key_values: vec![],
    });
    
    self.send_hook_callback(hook, status).await
}
```

### 6.4 Structured Error Responses

Return informative error responses:

```rust
#[derive(Debug, Serialize)]
pub struct ErrorResponse {
    pub error: String,   // Machine-readable code
    pub message: String, // Human-readable description
}

// Usage
Err((
    StatusCode::BAD_GATEWAY,
    Json(ErrorResponse {
        error: "gfbio_api_error".to_string(),
        message: format!("GFBio API request failed: {}", e),
    }),
))
```

---

## 7. Security Considerations

### 7.1 Secret Verification

Verify webhook authenticity using the provided secret:

```rust
fn verify_webhook_secret(hook: &Hook, expected: &str) -> Result<()> {
    if hook.secret != expected {
        return Err("Invalid webhook secret".into());
    }
    Ok(())
}
```

### 7.2 HTTPS Enforcement

Use TLS for all communications:

```rust
let endpoint = if server_url.starts_with("https") {
    Channel::from_shared(server_url)?
        .tls_config(ClientTlsConfig::new())?
} else {
    return Err("Only HTTPS endpoints allowed".into());
};
```

### 7.3 Credential Handling

- Never log credentials
- Use credentials immediately, don't store
- Credentials in payloads are temporary and scoped

### 7.4 Input Sanitization

Validate and sanitize all input:

```rust
// URL decode and re-encode to prevent injection
let decoded_url = urlencoding::decode(input_url)?;
let safe_url = urlencoding::encode(&decoded_url);
```

---

## 8. Practical Example: ABCD2BioSchema Service

This section walks through the complete ABCD2BioSchema implementation.

### 8.1 Service Overview

**Purpose**: Transform ABCD XML metadata into BioSchema JSON format automatically when files are uploaded to Aruna.

**Architecture:**

```
┌─────────────┐            ┌──────────────────┐         ┌──────────────────┐
│   Aruna     │   webhook  │  ABCD2BioSchema  │  API    │      GFBio       │
│   Storage   ├───────────►│     Service      ├────────►│  Transformation  │
│             │◄───────────┤                  │◄────────┤       API        │
└─────────────┘  callback  └──────────────────┘         └──────────────────┘
   ▲                          │
   │                          │
   │     S3 Upload            │
   └──────────────────────────┘
```

### 8.2 Trigger Configuration

The webhook fires when:
1. An object finishes uploading (`TRIGGER_TYPE_OBJECT_FINISHED`)
2. The object has a label with key `ABCD`

```json
{
  "trigger": {
    "triggerType": "TRIGGER_TYPE_OBJECT_FINISHED",
    "filters": [{
      "keyValue": {
        "key": "^ABCD$",
        "value": ".*",
        "variant": "KEY_VALUE_VARIANT_LABEL"
      }
    }]
  }
}
```

### 8.3 Data Flow

**Step 1: Receive webhook**

```rust
pub async fn handle_url_transformation(&self, hook: Hook) -> Result<...> {
    info!("Received hook: {:?}", hook);
    
    let download_url = hook.download.clone()
        .ok_or("No download URL provided")?;
```

**Step 2: Request transformation**

```rust
let gfbio_response = self.send_gfbio_request(&download_url).await?;
// GFBio API processes the XML and returns job info with result location
```

**Step 3: Fetch and upload result**

```rust
let object_id = self.fetch_result_data_and_upload(
    &hook, 
    &job.job_id, 
    &job.result_file
).await?;
```

**Step 4: Create new object with metadata**

```rust
let request = CreateObjectRequest {
    name: new_key.to_string(),
    title: format!("{} BioSchema", original_title),
    description: format!(
        "BioSchema-compliant format derived from ABCD record.\n\n\
         Original description:\n{}",
        original_description
    ),
    key_values: vec![
        KeyValue { key: "BioSchema", value: "true", ... },
        KeyValue { key: "TRANSFORMED_BY_GFBIO", value: "success", ... },
    ],
    relations: vec![
        // ORIGIN relationship to source
        Relation {
            relation: Some(RelationEnum::Internal(InternalRelation {
                resource_id: source_object_id,
                defined_variant: InternalRelationVariant::Origin as i32,
                direction: RelationDirection::Outbound as i32,
                ...
            })),
        }
    ],
    // Inherit licenses and authors from source
    metadata_license_tag: source.metadata_license_tag,
    data_license_tag: source.data_license_tag,
    authors: source.authors,
    ...
};
```

**Step 5: Send success callback**

```rust
self.send_success_callback(&hook, object_id).await?;
```

### 8.4 Output Object Properties

Transformed objects include:

| Property        | Value                                              |
|-----------------|----------------------------------------------------|
| **Name**        | Original filename with `.json` extension           |
| **Title**       | Original title + " BioSchema"                      |
| **Description** | Explanation + original description                 |
| **Labels**      | `BioSchema: true`, `TRANSFORMED_BY_GFBIO: success` |
| **Relations**   | `ORIGIN` link to source ABCD file                  |
| **Licenses**    | Inherited from source                              |
| **Authors**     | Inherited from source                              |

---

## 9. Testing and Debugging

### 9.1 Local Testing

**Mock webhook payload:**

```bash
curl -X POST http://localhost:3000/transform/url \
  -H "Content-Type: application/json" \
  -d '{
    "hook_id": "test-hook-123",
    "secret": "test-secret",
    "object": {
      "object": {
        "id": "obj-456",
        "name": "test.xml",
        "content_len": 1024,
        "key_values": [{"key": "ABCD", "value": "true", "variant": 1}]
      }
    },
    "download": "https://example.com/test.xml",
    "token": "test-token",
    "pubkey_serial": 1
  }'
```

### 9.2 Health Endpoint

Always include a health check:

```rust
pub async fn health_check() -> impl IntoResponse {
    Json(serde_json::json!({
        "status": "ok",
        "message": "ABCD2BioSchema Service is running"
    }))
}
```

### 9.3 Logging

Implement comprehensive logging at appropriate levels:

```rust
// INFO: High-level operations
info!("Webhook received: hook_id={}", hook.hook_id);
info!("Transformation completed: object_id={}", new_object_id);

// DEBUG: Detailed information for troubleshooting
debug!("Download URL: {}", hook.download);
debug!("S3 Upload Target: bucket={}, key={}", bucket, new_key);

// WARN: Recoverable issues
warn!("Callback retry needed: {}", e);

// ERROR: Failures
error!("Failed to upload: {}", e);
```

### 9.4 Debugging Checklist

- [ ] Service is reachable from source system network
- [ ] Webhook endpoint URL is correctly configured
- [ ] Authentication token has required permissions
- [ ] Webhook filters match test data labels
- [ ] gRPC channel configured with correct TLS settings
- [ ] S3 credentials are valid and have write permissions
- [ ] Callbacks are being sent (check both success and error paths)
- [ ] Error logs capture sufficient detail

### 9.5 Common Issues

| Issue                  | Likely Cause          | Solution                                      |
|------------------------|-----------------------|-----------------------------------------------|
| Webhook never triggers | Filter mismatch       | Verify label regex patterns                   |
| Download fails         | Expired presigned URL | Check timing between trigger and download     |
| Upload fails           | Invalid credentials   | Verify access_key and secret_key from payload |
| Callback fails         | Token permissions     | Ensure token has hook callback permission     |
| Timeout errors         | Processing too slow   | Implement async pattern                       |

---

## 10. Checklist for Production

### Pre-Deployment

- [ ] All error paths send callbacks
- [ ] Idempotency handling implemented
- [ ] Health endpoint available
- [ ] Structured logging configured
- [ ] Secrets not logged
- [ ] HTTPS enforced
- [ ] Timeouts configured
- [ ] Resource cleanup implemented

### Deployment

- [ ] Environment variables documented
- [ ] Docker/container ready
- [ ] Network accessible from source system
- [ ] TLS certificates valid
- [ ] Monitoring and alerting configured

### Post-Deployment

- [ ] Webhook registered with source system
- [ ] Test event processed successfully
- [ ] Error scenarios tested
- [ ] Logs accessible
- [ ] Metrics being collected

### Environment Variables (ABCD2BioSchema)

```env
# Required
ARUNA_BASE_URL="https://aruna-engine.org"
ARUNA_SERVER_ADDRESS=http://server:50051
GFBIO_BASE_URL=https://transformation.gfbio.org/api
TEMP_DIR=/tmp/abcd2bioschema

# Optional
SERVER_HOST=0.0.0.0
SERVICE_PORT=3000
TRANSFORMATION_ID=5
```

---

## Summary

Webhooks enable real-time, event-driven integrations between systems. Building reliable webhook services requires 
attention to:

1. **Clear contracts**: Well-defined payloads and callback structures
2. **Robust processing**: Async patterns, idempotency, graceful degradation
3. **Complete feedback**: Always send status callbacks, success or failure
4. **Security**: Secret verification, HTTPS, credential handling
5. **Observability**: Comprehensive logging, health checks, monitoring

The ABCD2BioSchema service demonstrates these principles in a production implementation, transforming biological 
metadata automatically while maintaining full traceability between source and derived data.

---

## Additional Resources

- [ABCD2BioSchema Repository](https://github.com/arunaengine/ABCD2BioSchema)
- [GFBio Transformation API](https://transformation.gfbio.org)
- [BioSchema.org](https://bioschemas.org/)