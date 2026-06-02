# Tend — To‑Do App: AWS Build Specification

A complete, build‑ready specification for the **Tend** to‑do app: the UI requirements, the DynamoDB data model, the AWS Lambda backend API, authentication, and a pay‑per‑use AWS deployment you can build in VS Code (with Claude Code) and ship with AWS SAM.

> **Design principle: pay only for what you use.** Every service below is serverless / on‑demand and scales to (near) zero cost when idle. The document also calls out the few traps that quietly bill you 24/7 so you can avoid them.

---

## 1. Scope and suggested phases

**Core (v1):**
- Email/password (and optional Google) **login**.
- After login, the user can **see, add, edit, complete, and delete** their to‑dos.
- Each to‑do supports: a **title**, an optional **note/description**, a **due date**, a **recurrence** rule, a **priority**, **tags**, **subtasks** (nesting), and a **status** (Plan / In Progress / Done).
- **List** and **Board** views, time filters, a Someday bucket, tags, light/dark theme.
- Data is private per user and persists in the cloud.

**Later (v2):**
- Scheduled **email digest** of due tasks.
- **Print** the digest to a printer.

The two phases share the same data model; v2 only adds two Lambdas and two services (SES + EventBridge), so building v1 first costs you nothing later.

---

## 2. AWS services (all pay‑per‑use)

| Layer | Service | Role | Cost behavior |
|---|---|---|---|
| **UI hosting** | **Amazon S3** | Stores the static web build (HTML/CSS/JS) | Pennies for storage + requests |
| | **Amazon CloudFront** | CDN / HTTPS / custom domain in front of S3 | Pay per request + transfer |
| **Auth** | **Amazon Cognito** | User sign‑up / sign‑in, issues JWTs | Free tier of monthly active users; pay per MAU after |
| **API** | **Amazon API Gateway (HTTP API)** | HTTPS endpoints the frontend calls | Pay per request (cheaper than REST API) |
| **Compute** | **AWS Lambda** | Runs all backend logic | Pay per request + GB‑seconds |
| **Database** | **Amazon DynamoDB (on‑demand)** | Stores tasks, tags, settings | Pay per read/write; ~0 idle |
| **Scheduler (v2)** | **Amazon EventBridge Scheduler** | Cron tick for the daily digest | Pay per invocation |
| **Email (v2)** | **Amazon SES** | Sends the digest email / email‑to‑print | ~fraction of a cent per email |
| **Queue (v2, optional)** | **Amazon SQS** | Reliable retries between scheduler and senders | Pay per request |
| **Secrets** | **AWS Systems Manager Parameter Store** | Stores API keys (e.g. PrintNode) | Standard tier is free |
| **IaC / deploy** | **AWS SAM** (+ **AWS CLI**, **AWS Toolkit for VS Code**) | Define + deploy everything from VS Code | Free (you pay only for deployed resources) |
| **Printing (v2, external)** | **PrintNode** (not AWS) | Sends print jobs to a physical printer | Its own usage pricing |

**Minimum to run v1:** S3 + CloudFront, Cognito, API Gateway + Lambda, DynamoDB. Everything else is additive.

### Cost guardrails (read before deploying)
- **Never put the Lambdas in a VPC** unless you must. A VPC for internet access requires a **NAT Gateway (~$32/month fixed)** — the single biggest hidden cost. DynamoDB, SES, and Cognito are all reached over public AWS endpoints, so no VPC is needed.
- **DynamoDB in on‑demand mode**, not provisioned — idle tables cost only storage.
- **Use Parameter Store (standard, free)** for secrets, not Secrets Manager (~$0.40/secret/month).
- **Set CloudWatch Logs retention** (7–14 days) on every Lambda log group, or logs accrue forever.
- **Prefer HTTP API or Lambda Function URLs** over REST API.
- Turn on a **billing alarm / AWS Budget** (e.g. alert at $5/month) as a safety net.

---

## 3. Architecture

```
                       ┌──────────────────────────┐
   Browser / Mobile ──►│ CloudFront  →  S3 (static)│   (the Tend web app)
        │              └──────────────────────────┘
        │ JWT (Cognito access token)
        ▼
   ┌─────────────────────┐     ┌──────────────────────────────┐
   │ API Gateway HTTP API │────►│ Lambda (one per route group) │
   │  + Cognito authorizer│     │  tasksFn / tagsFn / settingsFn│
   └─────────────────────┘     └───────────────┬──────────────┘
                                                │
                                                ▼
                                   ┌────────────────────────┐
                                   │ DynamoDB (single table) │
                                   └────────────────────────┘

   v2 scheduling:
   EventBridge Scheduler ──(daily)──► digestFn (Lambda)
        digestFn → query due tasks (DynamoDB GSI) → SES (email)
                                                   └► PrintNode (print)
   Secrets (PrintNode key) ← Parameter Store
```

---

## 4. UI requirements (detailed)

The web app is a single‑page app. It must be **responsive and mobile‑first** (touch targets ≥ 40px, bottom‑sheet dialogs on mobile, columns that swipe). A working reference implementation already exists (`todo-app.html`); these are the functional requirements it must satisfy against the live backend.

### 4.1 Authentication screens
- **Sign up** (email + password; optional name). Email verification via Cognito.
- **Sign in** (email + password). Optional “Sign in with Google” (Cognito federated identity).
- **Forgot password** flow (Cognito).
- **Sign out** control in the header/settings.
- All app routes are gated: an unauthenticated user only sees the auth screens. The JWT is stored in memory/secure storage and attached as `Authorization: Bearer <token>` to every API call.

### 4.2 Core task model (what the UI exposes)
Each task supports:
- **Title** (required, single line).
- **Note / description** (optional, multi‑line free text).
- **Due date** (optional, `YYYY‑MM‑DD`).
- **Recurrence** (optional): one of *none, daily, weekly, monthly, specific day of month, every 3 months, every 6 months, yearly*; when “specific day of month”, a day `1–31` is stored.
- **Priority**: None / Low / Medium / High.
- **Tags**: zero or more, chosen from a user‑managed catalog (see 4.7).
- **Status**: Plan / In Progress / Done.
- **Subtasks**: a task may contain child tasks, nested to any depth.
- **Order**: manual sort position among siblings.

### 4.3 Views
- **View switcher: List ↔ Board**, remembered per user.
- **List view:** tasks shown as rows; subtasks nested and indented under their parent with a connecting line; expand/collapse per parent; each row shows status checkbox, title, and badges (priority, due date with color state, recurrence, in‑progress, subtask progress `x/y`).
- **Board view (Jira‑style):** three columns — **Plan, In Progress, Done**. Each top‑level task is a card placed in the column matching its (derived) status. A parent card lists all its subtasks underneath, each with a tappable status dot (Plan → In Progress → Done) and a progress bar. Cards drag between columns on desktop; on mobile, status is changed via the per‑subtask dots / card controls and columns swipe horizontally.

### 4.4 Status rules
- A **leaf task** (no subtasks) has a directly settable status.
- A **parent task derives its status from its subtasks**: all subtasks done → parent **Done**; some started → **In Progress**; none started → **Plan**. *Completing all subtasks automatically completes the parent.*
- Toggling a parent’s checkbox cascades to complete/reopen the whole subtree.

### 4.5 Due dates & quick scheduling
- Due dates render as smart badges: **Overdue** (red), **Today** (amber), **Tomorrow / weekday / date** (neutral/green).
- The editor offers **date quick‑chips**: *No date · Today · Tomorrow · Weekend · Next week*.

### 4.6 Recurrence behavior
- Completing a recurring task does **not** delete it; it **rolls forward** to the next occurrence and returns to Plan. (Mirrored server‑side — see 7.)

### 4.7 Tags (categories)
- Tags are created/deleted in **Settings** (a managed catalog, e.g. `personal`, `work`). Each tag has a consistent color.
- In the task editor, the user **picks** tags from the catalog (optional).
- A **Category dropdown** filters the whole app (list, board, counts) to a selected tag.
- Settings has independent toggles: **tags on main tasks** and **tags on subtasks**.

### 4.8 Filters & buckets
- Filter chips with live counts: **All · Today · This week · This month · Someday · Overdue · Done**.
- **Someday** = tasks with **no due date** (the backlog). Capturing = add a task with no date; promoting = give it a date.

### 4.9 Other UI requirements
- **Light/dark theme** toggle, remembered; respects system preference on first run.
- **Progress ring** showing completion % (scoped to the active category).
- **Autosave** — every change persists immediately (optimistic UI, then API call).
- **Empty states** for each filter.
- Toast confirmations for create/save/delete/reschedule.

---

## 5. Database schema (DynamoDB, single‑table)

A single on‑demand table named **`Tend`** holds all entities, partitioned by user. At personal scale, the app can fetch all of a user’s items in one query and do grouping/filtering client‑side (as the reference UI does).

**Table key schema**
- Partition key: `PK` (String)
- Sort key: `SK` (String)

**Global Secondary Index (for the v2 scheduler)**
- **GSI1** — `GSI1PK` (String), `GSI1SK` (String). Only items that need scheduling carry these keys.

### 5.1 Item types

**Task**
```json
{
  "PK": "USER#<cognitoSub>",
  "SK": "TASK#<taskId>",
  "type": "Task",
  "taskId": "01HF...",
  "title": "Pay electricity bill",
  "note": "Use the autopay portal; account #12345",
  "status": "plan",                 // plan | doing | done
  "due": "2026-06-15",              // YYYY-MM-DD or null
  "recurrence": "monthly",          // none|daily|weekly|monthly|dayOfMonth|q3|q6|yearly
  "dayOfMonth": null,               // 1..31 when recurrence = dayOfMonth
  "priority": 2,                    // 0 none, 1 low, 2 medium, 3 high
  "tags": ["personal"],            // subset of the user's tag catalog
  "parentId": null,                 // taskId of parent, or null for top-level
  "order": 3,                       // manual sort among siblings
  "collapsed": false,
  "createdAt": "2026-06-02T10:00:00Z",
  "updatedAt": "2026-06-02T10:00:00Z",
  "nextRunAt": "2026-06-15T07:00:00Z",   // v2: when to include in a digest; null if none
  "GSI1PK": "DUE",                  // present only if nextRunAt set
  "GSI1SK": "2026-06-15T07:00:00Z"  // = nextRunAt, for range queries
}
```

**Tag catalog** (one item per user)
```json
{
  "PK": "USER#<cognitoSub>",
  "SK": "TAGS",
  "type": "TagCatalog",
  "tags": ["personal", "work", "errands"]
}
```

**Settings** (one item per user)
```json
{
  "PK": "USER#<cognitoSub>",
  "SK": "SETTINGS",
  "type": "Settings",
  "theme": "light",        // light | dark
  "view": "list",          // list | board
  "tagsParent": true,
  "tagsSub": true,
  "digestHourUTC": 7,      // v2: preferred daily digest time
  "printEnabled": false    // v2
}
```

### 5.2 Access patterns → queries

| # | Need | Operation |
|---|---|---|
| 1 | All of a user’s tasks | `Query PK = USER#<sub> AND begins_with(SK, "TASK#")` |
| 2 | One task | `GetItem PK=USER#<sub>, SK=TASK#<id>` |
| 3 | Create / update a task | `PutItem` / `UpdateItem` on that key |
| 4 | Delete a task + subtree | Query children by `parentId` (or fetch all + filter), then `BatchWriteItem` deletes |
| 5 | Tag catalog | `GetItem PK=USER#<sub>, SK=TAGS` |
| 6 | Settings | `GetItem PK=USER#<sub>, SK=SETTINGS` |
| 7 | App bootstrap (load everything) | single `Query PK=USER#<sub>` returns tasks + tags + settings |
| 8 | (v2) Tasks due now, all users | `Query GSI1 where GSI1PK="DUE" AND GSI1SK <= now` |

> For higher scale you would shard `GSI1PK` (e.g. `DUE#0..9`) and fan out, but a single partition is fine for personal use.

---

## 6. Backend API (AWS Lambda)

**Auth:** API Gateway **HTTP API** with a **Cognito JWT authorizer**. Every request must carry `Authorization: Bearer <access_token>`. The Lambda reads the user id from the verified `sub` claim — clients never pass a user id.

**Lambda organization:** group routes into a few functions (cheaper cold‑start surface, simpler IAM). Suggested: `tasksFn`, `tagsFn`, `settingsFn`, plus `digestFn` (v2). Each Lambda has an IAM role scoped to the `Tend` table (and SES/Parameter Store for `digestFn`).

### 6.1 Endpoints

| Method | Path | Purpose | Body / notes |
|---|---|---|---|
| `GET` | `/bootstrap` | Load tasks + tags + settings in one call (app startup) | — |
| `GET` | `/tasks` | List the user’s tasks | — |
| `POST` | `/tasks` | Create a task (or subtask) | `{title, note?, due?, recurrence?, dayOfMonth?, priority?, tags?, parentId?}` |
| `PATCH` | `/tasks/{id}` | Update any task fields | partial: `{title?, note?, status?, due?, recurrence?, dayOfMonth?, priority?, tags?, order?, collapsed?}` |
| `DELETE` | `/tasks/{id}` | Delete a task and all its subtasks | server deletes the subtree |
| `GET` | `/tags` | Get the tag catalog | — |
| `POST` | `/tags` | Add a tag to the catalog | `{name}` |
| `DELETE` | `/tags/{name}` | Delete a tag (and remove it from all tasks) | — |
| `GET` | `/settings` | Get user settings | — |
| `PUT` | `/settings` | Replace user settings | `{theme, view, tagsParent, tagsSub, digestHourUTC?, printEnabled?}` |

### 6.2 Server‑side rules the Lambdas enforce
- **Ownership:** every read/write is constrained to `PK = USER#<sub>` from the token. A user can never touch another user’s items.
- **Status derivation:** on any status change (`PATCH`), recompute affected parents’ status from their children (all done → done, etc.).
- **Recurrence roll‑forward:** when a `PATCH` sets a **recurring** task to `done`, the Lambda computes the next due date, resets status to `plan`, updates `due`/`nextRunAt`, and returns the rescheduled task (instead of marking it permanently done).
- **Cascade delete:** `DELETE /tasks/{id}` removes the task and every descendant.
- **Validation:** title required; `recurrence` and `priority` from the allowed enums; `dayOfMonth` 1–31 only when `recurrence = dayOfMonth`.
- **Timestamps:** set `createdAt` on create, `updatedAt` on every write.

### 6.3 Example request/response

`POST /tasks`
```json
// request
{ "title": "Renew passport", "note": "Bring photo", "due": "2026-07-01", "priority": 3, "tags": ["personal"] }

// response 201
{ "taskId": "01HF8...", "title": "Renew passport", "note": "Bring photo",
  "status": "plan", "due": "2026-07-01", "recurrence": "none", "priority": 3,
  "tags": ["personal"], "parentId": null, "order": 0,
  "createdAt": "2026-06-02T10:00:00Z", "updatedAt": "2026-06-02T10:00:00Z" }
```

---

## 7. Scheduling, email & print (v2)

1. **EventBridge Scheduler** fires `digestFn` on a schedule (e.g. hourly; each run handles users whose `digestHourUTC` matches, or once daily for a single‑user app).
2. `digestFn` queries due/overdue tasks (GSI1 or per‑user query), groups them by category/status, and renders a digest (and a PDF if printing is on).
3. It sends the email via **SES** (`from` a verified address) and, if `printEnabled`, pushes the PDF to **PrintNode** using the key from **Parameter Store**.
4. **SQS** (optional) sits between “find due tasks” and “send” for retries and rate‑limiting.

> Recurring tasks advance when the user completes them (rule in 6.2), so the digest only **reports** what’s due — it doesn’t mutate schedules.
> **SES note:** new accounts start in a sandbox; request production access and verify your sender domain/address before sending to arbitrary recipients.

---

## 8. Authentication (Cognito) flow

1. **Cognito User Pool** with an **app client** (no client secret for a browser SPA). Enable email sign‑up + (optional) Google as a federated IdP.
2. Frontend uses **AWS Amplify Auth** (or Cognito Hosted UI) for sign‑up/sign‑in/forgot‑password and to obtain JWTs.
3. The **access token** is sent on every API call; **API Gateway’s Cognito JWT authorizer** validates it and passes claims to the Lambda.
4. Token refresh is handled by Amplify; on sign‑out, tokens are cleared.

---

## 9. Project structure & deployment (VS Code + SAM)

**Install once:** AWS CLI, AWS SAM CLI, Node.js, and the **AWS Toolkit for VS Code** extension. Run `aws configure` with an IAM user/role that can deploy.

**Suggested repo layout**
```
tend/
├─ template.yaml              # SAM: API, Lambdas, DynamoDB table, Cognito, (v2) scheduler/SES
├─ src/
│  ├─ tasks/                  # tasksFn handler
│  ├─ tags/                   # tagsFn handler
│  ├─ settings/               # settingsFn handler
│  ├─ digest/                 # digestFn handler (v2)
│  └─ lib/                    # shared: dynamo client, auth helpers, recurrence logic
├─ web/                       # the frontend (your todo-app, built to static files)
│  └─ dist/                   # build output synced to S3
└─ README.md
```

**`template.yaml` should define:** the DynamoDB table (`PAY_PER_REQUEST`, GSI1), the Lambda functions (Node.js, **not in a VPC**), the HTTP API with the Cognito authorizer, the Cognito User Pool + client, log‑group retention, and (v2) the EventBridge schedule + SES permissions. Store the PrintNode key in Parameter Store and grant `digestFn` read access.

**Deploy backend**
```bash
sam build
sam deploy --guided      # first time; creates the stack and saves config
```

**Deploy frontend**
```bash
# build the web app, then:
aws s3 sync web/dist s3://<your-bucket> --delete
aws cloudfront create-invalidation --distribution-id <id> --paths "/*"
```
*(Or use AWS Amplify Hosting for git‑push deploys instead of the S3/CloudFront commands.)*

After `sam deploy`, wire the frontend to the outputs: the **API base URL**, the **Cognito User Pool ID**, and the **App Client ID**.

---

## 10. Security & operational notes
- Private per user via the `sub`‑scoped partition key; IAM roles least‑privilege to the `Tend` table.
- HTTPS everywhere (CloudFront + API Gateway).
- CORS on the HTTP API limited to your CloudFront domain.
- Enable point‑in‑time recovery on DynamoDB (cheap insurance) and a billing alarm.

---

## 11. Open questions (please confirm)
1. **“Note”** — I’ve modeled it as a **description field on each task**. Did you instead want notes as a **separate entity** from to‑dos? If separate, I’ll add a `Note` item type and its own endpoints.
2. **Scope of v1** — build **login + full task CRUD first**, and add the **email digest + printing** as v2? (Recommended.)
3. **Reminders** — a **single daily digest**, or **per‑task time‑of‑day reminders**? (Per‑task reminders add a bit of scheduler complexity.)
4. **Users** — just **you**, or will **other people** sign up too? (Affects Cognito setup and SES production access.)
5. **Sign‑in methods** — email/password only, or also **Google**?
6. **Custom domain** — do you want one (e.g. `tend.yourdomain.com`) via Route 53 + ACM, or is the default CloudFront URL fine to start?
7. **Region** — which AWS region (closest to you, e.g. `ca-central-1` for Calgary)?

---

### Suggested next step
If you confirm the answers above (especially #1–#3), I can generate the **`template.yaml` (SAM)** plus starter **Lambda handler code** and the **shared recurrence/status logic**, ready to `sam deploy` from VS Code.
