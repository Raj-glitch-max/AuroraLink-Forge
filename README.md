# AuroraLink Forge

AuroraLink Forge is a production-style URL shortener that feels like a real product rather than a lab exercise. It gives marketing, growth, or platform teams the ability to mint branded links, track clicks, set expiration rules, and automatically retire stale entries—all powered by AWS-native services so it scales with demand.

## About the Project
The brief was to recreate a URL shortener from the ground up without lifting old code and to do it in a DevOps-first way. Infrastructure is defined using AWS SAM, every Lambda uses structured JSON logging, validation prevents bad data from ever reaching DynamoDB, and configs are driven by `.env` so each environment can tune TTLs, alias rules, or logging levels without touching code.

## Architecture Diagram
```
+------------+      HTTPS      +-------------------+      Lambda Invocation     +------------------------+
|  Clients   +---------------> | Amazon API Gateway +--------------------------> + Lambda functions       |
| (CLI/web)  |                 |  /links endpoints  |                            | create / resolve /     |
+-----+------+                 +---------+---------+                            | stats / cleanup        |
      |                                 |                                    +-----------+------------+
      |                                 v                                    | DynamoDB single table |
      |                       +--------------------+                         | PK=LINK#code items     |
      |                       | CloudWatch Logs &  |                         +-----------+------------+
      |                       | metrics (JSON)     |                                     |
      |                       +--------------------+                                     v
      |                                                                    EventBridge (rate 15m) 
      |                                                                    triggers cleanup Lambda
      +-------------------------------------------------------------------------------------------+
```

## How the pieces work together
1. **Create flow (`POST /links`)** – API Gateway calls `create_link`, which loads config from the environment, validates the payload, generates or accepts a short code, writes a strongly consistent record to DynamoDB, and returns the formatted short URL.
2. **Redirect flow (`GET /{code}`)** – `resolve_link` grabs the record, ensures it hasn’t expired, increments the click counter atomically, logs the click, and responds with an HTTP 302.
3. **Analytics flow (`GET /links/{code}/stats`)** – `link_stats` returns the destination, click counts, creation timestamp, and TTL info for dashboards or ops tooling.
4. **Cleanup loop** – EventBridge fires the `cleanup_expired` Lambda every 15 minutes, which scans a limited batch of expired items and deletes them so the table stays tidy even before DynamoDB TTL eventually kicks in.
5. **Observability** – All handlers share a JSON-formatted logger, so CloudWatch Insights or metric filters can slice and dice events (alias collisions, error codes, cleanup counts, etc.).

## Feature Highlights
1. Config-driven behavior via `.env` (domain, TTL defaults, alias toggles, cleanup batch size).
2. Custom alias support with DynamoDB conditional writes to prevent collisions.
3. Optional TTL per link with safe upper bounds.
4. Click analytics endpoint with created/expiry timestamps for reporting.
5. Structured JSON logging suitable for Insights queries and alarm filters.
6. Seed script to quickly populate demonstration links.
7. pytest coverage for each Lambda entry point.

## Tech Stack
- AWS SAM
- AWS Lambda (Python 3.11)
- Amazon API Gateway (REST)
- Amazon DynamoDB (single-table design)
- Amazon EventBridge + CloudWatch Logs
- pytest

## Folder Overview
```
auroralink-forge/
├── sam-template.yaml       # Infrastructure as code (Lambda, API, DynamoDB, IAM, EventBridge)
├── README.md               # Documentation you’re reading now
├── .env.example            # Configuration template
├── src/
│   ├── handlers/           # Lambda entrypoints
│   ├── models/             # DynamoDB repository helper
│   └── utils/              # Config loader, validators, responders, shortener helpers
├── tests/                  # pytest suites
└── scripts/seed_data.py    # Utility to preload sample links
```

## Environment Variables
| Variable               | Description                                                          |
|------------------------|----------------------------------------------------------------------|
| `LINKS_TABLE_NAME`     | DynamoDB table name storing link metadata                            |
| `SHORT_DOMAIN`         | Base domain for composed short URLs (e.g., `https://auroralink.io`)   |
| `DEFAULT_TTL_SECONDS`  | TTL applied when clients omit a value                                 |
| `MAX_TTL_SECONDS`      | Guardrail to prevent “forever” links                                 |
| `ALLOW_CUSTOM_ALIAS`   | Enable/disable user-provided aliases                                 |
| `MAX_ALIAS_LENGTH`     | Alias length ceiling                                                  |
| `MAX_URL_LENGTH`       | Destination length limit                                             |
| `LOG_LEVEL`            | Logging verbosity for the structured logger                          |
| `CLEANUP_BATCH_SIZE`   | Number of expired items purged per cleanup invocation                |

Copy `.env.example` to `.env`, update values, and export them before local runs.

## DynamoDB Design
- `PK = LINK#<code>` and `SK = METADATA`
- Attributes tracked: `destination`, `owner`, `createdAt`, `expiresAt`, `clicks`
- TTL attribute: `expiresAt` (works in tandem with the cleanup Lambda)
- Counter row: `PK = COUNTER#GLOBAL`, `SK = STATE`, attribute `counter`

## IAM Summary
SAM grants least-privilege per function:
- Create function → `dynamodb:PutItem`, `UpdateItem`, `GetItem`
- Resolve function → `dynamodb:GetItem`, `UpdateItem`
- Stats function → `dynamodb:GetItem`
- Cleanup function → `dynamodb:Scan`, `DeleteItem`
Logging permissions are inherited from SAM’s defaults.

## Setup Checklist
1. Install Python 3.11, AWS CLI v2, and AWS SAM CLI.
2. Configure AWS credentials with rights to create Lambda/API/DynamoDB/EventBridge resources.
3. (Optional) Create a virtual environment and install `pytest` for tests.
4. Copy `.env.example` → `.env` and supply values for your AWS account.

## Running Locally
1. Build:
   ```bash
   sam build
   ```
2. Invoke functions (requires Docker):
   ```bash
   sam local invoke CreateLinkFunction --event events/create_link.json
   sam local invoke ResolveLinkFunction --event events/resolve_link.json
   sam local invoke LinkStatsFunction --event events/link_stats.json
   ```
3. Tests:
   ```bash
   pip install pytest
   pytest tests
   ```

## Deployment (AWS SAM)
1. Build artifacts:
   ```bash
   sam build
   ```
2. First deployment (guided):
   ```bash
   sam deploy --guided
   ```
3. Repeat deployments:
   ```bash
   sam deploy
   ```

## API Endpoints
| Method | Path                  | Purpose                                   |
|--------|-----------------------|-------------------------------------------|
| POST   | `/links`              | Create a short link                       |
| GET    | `/{code}`             | Resolve + redirect to the destination     |
| GET    | `/links/{code}/stats` | Return analytics (clicks, TTL, timestamps) |

### Request/Response Examples
**Create**
```http
POST /links
{
  "destination": "https://example.com/docs",
  "alias": "launch",
  "ttlSeconds": 86400,
  "owner": "growth"
}
```
Response:
```json
{
  "code": "launch",
  "shortUrl": "https://auroralink.io/launch",
  "destination": "https://example.com/docs",
  "expiresAt": 1700000000
}
```

**Stats**
```json
{
  "code": "launch",
  "destination": "https://example.com/docs",
  "clicks": 42,
  "createdAt": "2025-11-18T10:00:00Z",
  "expiresAt": 1700000000
}
```

## Unique Touches
1. Structured JSON logging for CloudWatch Insights out of the gate.
2. Config-driven behavior so dev/stage/prod can each tune TTLs and alias rules.
3. Dual-layer expiry enforcement: DynamoDB TTL plus EventBridge cleanup for determinism.
4. Seed script + analytics endpoint to demonstrate the platform quickly during demos.

## Future Enhancements
- Add API keys or Cognito authorizers for `POST /links`.
- Stream click events to Kinesis Firehose for real-time analytics.
- Provide bulk import/export workflows.
- Create a simple web console for non-technical users.


