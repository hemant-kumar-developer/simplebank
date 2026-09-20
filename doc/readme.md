# Simple Bank Project - Complete Beginner Guide (Hinglish)

> Ye document `simplebank` repository ko first-time developer ke nazariye se explain karta hai. Isme architecture, folders, important files, request flows, database, authentication, worker, frontend, tests, Docker aur Kubernetes sab cover kiye gaye hain.
>
> **Important scope note:** `pb/`, `db/sqlc/`, `db/mock/`, `doc/statik/` aur kuch frontend/generated assets machine-generated hain. Unki har generated line ko manually explain karna useful nahi hota, isliye yahan unka source file, generation command, generated contract aur use explain kiya gaya hai. Business logic wali handwritten files ko symbol-by-symbol aur flow-by-flow samjhaya gaya hai.

## 1. Project ka purpose

Simple Bank ek learning-oriented banking backend hai. Iska main kaam:

1. User banana aur user profile update karna.
2. User ko login karwana.
3. Access token aur refresh token dena.
4. Bank accounts banana aur account balance maintain karna.
5. Balance ke har change ka `entries` record banana.
6. Do accounts ke beech atomic money transfer karna.
7. Email verification ke liye background job queue karna.
8. Same service ko native gRPC aur HTTP JSON gateway dono ke through expose karna.
9. Local Docker aur production Kubernetes deployment support karna.

Ye repository course project se grow hua hai. Isliye isme do API generations milti hain:

- `api/`: purana Gin REST API. Iske tests hain, lekin current `main.go` ise start nahi karta.
- `gapi/` + `proto/` + `pb/`: current gRPC API. HTTP clients ke liye grpc-gateway isi API ko JSON endpoints me expose karta hai.

Current runtime me sabse important path hai:

```text
Client (frontend / gRPC client)
        |
        | HTTP JSON: :8080        | native gRPC: :9090
        v                         v
grpc-gateway ---------------> gapi.Server
                                  |
                    +-------------+-------------+
                    |                           |
                 PostgreSQL                 Redis/Asynq
                    |                           |
                sqlc Store                 Email worker
```

## 2. Application start hone par kya hota hai

Entry point `main.go` hai. Program ka startup sequence ye hai:

1. `util.LoadConfig(".")` current directory se `app.env` padhta hai.
2. Development environment me zerolog ko human-readable console mode milta hai.
3. OS shutdown signals ke liye context banaya jata hai.
4. `pgxpool.New` PostgreSQL connection pool create karta hai.
5. `runDBMigration` pending migrations automatically apply karta hai.
6. `db.NewStore` sqlc queries ko application-facing store me wrap karta hai.
7. Redis address se Asynq task distributor create hota hai.
8. `errgroup` ke andar teen concurrent services start hoti hain:
   - background task processor,
   - HTTP grpc-gateway,
   - native gRPC server.
9. Context cancel hone par har service graceful shutdown karti hai.
10. Koi goroutine error return kare to `waitGroup.Wait()` us error ko observe karta hai.

### `main.go` ke important functions

- `main`: poora dependency graph banata hai aur servers start karta hai.
- `runDBMigration`: `golang-migrate` se database schema ko latest version tak le jata hai.
- `runTaskProcessor`: Gmail sender aur Redis/Asynq processor start karta hai; shutdown par `Shutdown()` call karta hai.
- `runGrpcServer`: gRPC listener, interceptor, service registration aur graceful stop configure karta hai.
- `runGatewayServer`: protobuf service ko HTTP JSON gateway par register karta hai, CORS/logging/Swagger/static frontend configure karta hai.
- `runGinServer`: legacy Gin server ka helper hai; current `main` me call nahi hota.

`errgroup.WithContext` ka matlab hai ki multiple long-running services ko ek shared cancellation signal milta hai. `SIGINT` ya `SIGTERM` par context cancel hota hai aur services resources release karke band hoti hain.

## 3. Repository map

### Top-level files

| Path | Kya karta hai |
|---|---|
| `main.go` | Application bootstrap, migration, worker, gRPC aur gateway startup |
| `app.env` | Local configuration values |
| `go.mod` | Go module name, Go version aur dependencies |
| `go.sum` | Dependency checksums; exact downloaded versions verify karta hai |
| `Makefile` | Repeated Docker, migration, generation aur test commands |
| `Dockerfile` | Multi-stage Go container image build |
| `docker-compose.yaml` | Local PostgreSQL, Redis aur API services |
| `start.sh` | Container entrypoint; passed command execute karta hai |
| `wait-for.sh` | Dependency TCP/HTTP ready hone ka wait helper |
| `sqlc.yaml` | SQL se Go code generation ki configuration |
| `README.md` | Course overview, setup aur lecture links |
| `LICENSE` | MIT license |
| `simplebank` | Repository me present compiled binary; source of truth nahi |

### Main folders

| Folder | Responsibility |
|---|---|
| `api/` | Legacy Gin HTTP API aur uske tests |
| `db/migration/` | Ordered PostgreSQL schema migrations |
| `db/query/` | Handwritten SQL jo sqlc input hai |
| `db/sqlc/` | sqlc-generated models, queries, store aur transactions |
| `db/mock/` | GoMock-generated database mock |
| `gapi/` | Active gRPC service handlers, auth, logging aur converters |
| `pb/` | Protobuf compiler se generated Go/gRPC/gateway code |
| `proto/` | Handwritten protobuf contracts aur annotation dependencies |
| `token/` | JWT/PASETO token abstraction aur implementations |
| `util/` | Config, password, random, currency, role helpers |
| `val/` | Input validation rules |
| `worker/` | Redis/Asynq distributor, processor aur email task |
| `mail/` | Gmail SMTP email sender |
| `frontend/` | Vue 3 + Vite + TypeScript frontend |
| `doc/` | DBML, schema dump, Swagger aur this guide |
| `eks/` | Kubernetes/EKS deployment manifests |
| `.github/workflows/` | CI tests aur release deployment automation |

## 4. Configuration: `app.env` aur `util/config.go`

`app.env` me ek line `NAME=value` format me hoti hai. `util.Config` har value ko typed Go field me map karta hai.

| Variable | Meaning | Example |
|---|---|---|
| `ENVIRONMENT` | Logging/runtime mode | `development` |
| `ALLOWED_ORIGINS` | CORS origins ki comma-separated list | `http://localhost:3000` |
| `DB_SOURCE` | PostgreSQL connection string | `postgresql://...` |
| `MIGRATION_URL` | Migration files ka source | `file://db/migration` |
| `HTTP_SERVER_ADDRESS` | Gateway bind address | `0.0.0.0:8080` |
| `GRPC_SERVER_ADDRESS` | gRPC bind address | `0.0.0.0:9090` |
| `REDIS_ADDRESS` | Redis host/port | `localhost:6379` |
| `TOKEN_SYMMETRIC_KEY` | PASETO symmetric key; 32 bytes/chars expected | development key |
| `ACCESS_TOKEN_DURATION` | Access token lifetime | `1m` |
| `REFRESH_TOKEN_DURATION` | Refresh token lifetime | `24h` |
| `EMAIL_SENDER_NAME` | Email display name | `Simple Bank` |
| `EMAIL_SENDER_ADDRESS` | SMTP sender account | Gmail address |
| `EMAIL_SENDER_PASSWORD` | SMTP credential/app password | secret |

### `util/config.go` line-by-line concept

- `package util`: file utility package ka part hai.
- `import time`: duration fields ke liye `time.Duration` chahiye.
- `import viper`: file aur environment configuration read karne ke liye.
- `type Config struct`: application ki saari settings ek object me rakhta hai.
- `mapstructure:"..."`: Viper ko batata hai kaun sa env/config key kis field me jana hai.
- `AllowedOrigins []string`: multiple CORS origins represent karta hai.
- Duration fields `1m`/`24h` jaise strings ko typed duration me parse karte hain.
- `LoadConfig(path string)`: Viper ko config directory, filename `app`, aur type `env` deta hai.
- `AutomaticEnv()`: process environment ko config values par override karne deta hai.
- `ReadInConfig()`: actual `app.env` read karta hai.
- `Unmarshal(&config)`: raw settings ko typed struct me convert karta hai.

**Security:** repository me `app.env` ke andar real-looking email password aur token/database defaults dikh rahe hain. Shared repository me in secrets ko rotate karke environment/secret manager se load karna chahiye. Guide me actual secret repeat nahi kiya gaya hai.

## 5. Database design

### 5.1 `accounts`

- `id`: auto-incrementing `bigserial` primary key.
- `owner`: username; users table se foreign key.
- `balance`: integer minor-unit style amount. Code floating point money use nahi karta.
- `currency`: supported currency string.
- `created_at`: PostgreSQL `now()` default.

`owner + currency` unique constraint ka matlab ek user ki same currency me duplicate account nahi ban sakta.

### 5.2 `entries`

- Har balance change ka ledger-style record.
- `amount` positive ho sakta hai deposit/credit ke liye.
- `amount` negative ho sakta hai withdrawal/debit ke liye.
- `account_id` account ko refer karta hai.

### 5.3 `transfers`

- Ek transfer ka business record.
- `from_account_id` source.
- `to_account_id` destination.
- `amount` positive hona chahiye; negative direction entries me represent hoti hai.
- Foreign keys ensure karti hain ki dono accounts exist karte hain.

### 5.4 `users`

- `username`: primary key.
- `hashed_password`: bcrypt hash; plaintext password kabhi store nahi hota.
- `full_name`: display name.
- `email`: unique.
- `password_changed_at`: password update ke time security metadata.
- `created_at`: registration time.
- `is_email_verified`: verification status.
- `role`: `depositor` ya `banker`; default `depositor`.

### 5.5 `sessions`

Refresh token ko database session se bind karta hai:

- `id`: refresh token payload ka UUID.
- `username`: session owner.
- `refresh_token`: stored token.
- `user_agent` aur `client_ip`: login metadata.
- `is_blocked`: session revoke karne ka flag.
- `expires_at`: refresh session expiry.
- `created_at`: session creation time.

### 5.6 `verify_emails`

- `id`: email verification request id.
- `username`, `email`: recipient/user relation.
- `secret_code`: random verification secret.
- `is_used`: replay prevention.
- `created_at`: creation time.
- `expired_at`: default creation se 15 minutes baad.

### Migration files

`db/migration/` me har schema change pair me hota hai:

- `NNNNNN_name.up.sql`: change apply.
- `NNNNNN_name.down.sql`: change rollback.

Current order:

1. `000001_init_schema`: accounts, entries, transfers, foreign keys aur indexes.
2. `000002_add_users`: users table, account owner relation, owner/currency uniqueness.
3. `000003_add_sessions`: refresh token sessions.
4. `000004_add_verify_emails`: verification records aur user verification flag.
5. `000005_add_role_to_users`: role column with `depositor` default.

Migration filenames ka numeric order important hai. Naya schema change hamesha new migration se add karein; purani applied migration ko edit na karein.

## 6. SQL aur sqlc layer

### `db/query/`

Ye handwritten SQL source of truth hai:

- `account.sql`: create/get/list account, row locking, balance update.
- `entry.sql`: entry insert/get/list.
- `transfer.sql`: transfer insert/get/list.
- `user.sql`: create/get/update user; optional fields nullable parameters se update.
- `session.sql`: create/get session.
- `verify_email.sql`: code create aur valid unused code consume.

sqlc SQL comments/annotations ko padhkar typed Go methods generate karta hai. Isse SQL strongly typed Go API me convert hota hai aur manually repetitive scan/args code likhne ki zaroorat nahi padti.

### `sqlc.yaml`

- `schema`: sqlc ko table definitions kahan milengi.
- `queries`: handwritten queries ka folder.
- `engine: postgresql`: PostgreSQL syntax/type rules.
- `package: db`: generated package name.
- `out: db/sqlc`: generated output.
- `sql_package: pgx/v5`: generated database code pgx v5 use kare.
- `emit_json_tags`: structs me JSON tags.
- `emit_interface`: `Querier` interface generate.
- `emit_empty_slices`: no-row list result ko nil ke badle empty slice behavior.
- `overrides`: PostgreSQL `timestamptz` ko `time.Time` aur UUID ko `uuid.UUID` map.

### `db/sqlc/`

Generated files ko manually edit nahi karna chahiye. SQL badalne ke baad `make sqlc` run karein.

- `models.go`: table row structs.
- `querier.go`: all generated query methods ka interface.
- `db.go`: `DBTX`, `Queries`, connection/transaction binding.
- `account.sql.go`, `entry.sql.go`, `transfer.sql.go`, `user.sql.go`, `session.sql.go`, `verify_email.sql.go`: SQL statements ke typed implementations.
- `error.go`: PostgreSQL error code classification.
- `store.go`: application-facing `Store` interface aur `SQLStore`.
- `exec_tx.go`: transaction begin/callback/commit/rollback helper.
- `tx_transfer.go`: transfer ko ek atomic transaction me execute karta hai.
- `tx_create_user.go`: user insert aur callback ke through after-create action.
- `tx_verify_email.go`: verification code consume aur user verification update atomically.

### Transfer transaction ka exact logic

`TransferTx` ke callback me:

1. `transfers` row insert hoti hai.
2. Source account ke liye negative `entries` row insert hoti hai.
3. Destination account ke liye positive `entries` row insert hoti hai.
4. Dono balances update hote hain.
5. Koi bhi step fail ho to callback error deta hai aur transaction rollback hoti hai.
6. Deadlock risk kam karne ke liye account IDs ke ascending order me balance rows lock/update hoti hain.

Isliye transfer ka invariant hai: ya to transfer, entries aur balances sab update honge, ya kuch bhi nahi hoga.

## 7. Current gRPC + HTTP API

### `proto/` source contracts

`proto/service_simple_bank.proto` service ka public contract define karta hai. Har RPC ke saath `google.api.http` annotation HTTP route banati hai:

| RPC | HTTP method/path | Purpose |
|---|---|---|
| `CreateUser` | `POST /v1/create_user` | New user |
| `UpdateUser` | `PATCH /v1/update_user` | Partial user update |
| `LoginUser` | `POST /v1/login_user` | Tokens/session |
| `VerifyEmail` | `GET /v1/verify_email` | Email code verify |

`openapiv2_operation` descriptions Swagger generation ke liye hain. `option go_package` generated Go imports ka package path set karta hai.

Input/output messages alag files me hain:

- `rpc_create_user.proto`
- `rpc_update_user.proto`
- `rpc_login_user.proto`
- `rpc_verify_email.proto`
- `user.proto`

`proto/google/api/` aur `proto/protoc-gen-openapiv2/` compiler annotations ke vendored definitions hain. Ye business logic nahi hain.

### `pb/` generated output

`make proto` old generated files remove karke protobuf compiler chalata hai. Output:

- `*.pb.go`: request/response message structs.
- `service_simple_bank_grpc.pb.go`: gRPC client/server interfaces.
- `service_simple_bank.pb.gw.go`: HTTP gateway handlers.
- Swagger JSON: HTTP API documentation.

Generated output ko edit karne ke bajay `.proto` source edit karke `make proto` run karein.

### `gapi/server.go`

`Server` struct dependencies hold karta hai:

- config,
- database `store`,
- token maker,
- task distributor.

`NewServer` symmetric key se PASETO maker banata hai aur invalid key/config par constructor error deta hai. Is layer ka kaam transport-level RPC request ko business/database operations se connect karna hai.

### `gapi/rpc_create_user.go`

1. `validateCreateUserRequest` username, password, full name aur email validate karta hai.
2. Invalid fields ko structured `BadRequest.FieldViolation` response me convert kiya jata hai.
3. Password bcrypt se hash hota hai.
4. `CreateUserTxParams` me user insert data aur `AfterCreate` callback banta hai.
5. Callback Redis queue me verification email task schedule karta hai.
6. Database unique violation ko gRPC `AlreadyExists` banaya jata hai.
7. Success par database user ko protobuf `User` me convert karke response hota hai.

Yahan `AfterCreate` database transaction callback ke andar external Redis call karta hai. PostgreSQL aur Redis same transaction nahi hain; production design me outbox/reliable event pattern consider kiya ja sakta hai.

### `gapi/rpc_login_user.go`

1. Username/password validate.
2. Username se user load.
3. Missing user ko `NotFound`.
4. bcrypt password compare.
5. Access token short duration ke saath create.
6. Refresh token long duration ke saath create.
7. gRPC metadata se user-agent aur client IP extract.
8. Refresh token id aur metadata ke saath `sessions` row save.
9. User, session id, both tokens aur expiry timestamps response me return.

Password failure ko bhi `NotFound` return kiya gaya hai, jisse username/password distinction leak na ho.

### `gapi/rpc_update_user.go`

1. Bearer access token authorize hota hai.
2. Sirf `banker` aur `depositor` roles allowed.
3. Request fields validate hote hain.
4. Depositor sirf apna username update kar sakta hai.
5. Banker kisi user ko update kar sakta hai.
6. `pgtype.Text` aur `pgtype.Timestamptz` optional protobuf fields ko SQL NULL/valid semantics me convert karte hain.
7. Password diya ho to bcrypt hash aur `password_changed_at` update.
8. DB missing row ko `NotFound`.
9. Updated user response me convert.

### `gapi/rpc_verify_email.go`

1. Email id aur secret code validate.
2. `VerifyEmailTx` code ko atomic tareeqe se check/consume karta hai.
3. User ka `is_email_verified` true return hota hai.
4. Current implementation database verification failure ko generic `Internal` deta hai; invalid/expired/already-used code ke liye more precise status future improvement ho sakta hai.

### `gapi/authorization.go`

- Incoming gRPC metadata se `authorization` value read.
- `Bearer <token>` format parse.
- Token maker se signature/encryption, type aur expiry verify.
- Allowed roles me match check.
- Valid payload context ke liye return.

Authorization ka use abhi `UpdateUser` handler me visible hai. New protected RPC add karte waqt same authorization pattern apply karna hoga.

### `gapi/metadata.go`

- gRPC metadata se `user-agent` read.
- Peer information se client address/IP read.
- Login session ke audit fields fill karta hai.

### `gapi/error.go`

Validation errors ko gRPC status ke saath `BadRequest` details me encode karta hai. Isse client ko sirf generic error nahi, balki kis field me kya problem hai ye milta hai.

### `gapi/converter.go`

Database `db.User` fields ko protobuf `pb.User` fields me map karta hai. Conversion ko alag rakhne se handlers database model directly public response me leak nahi karte.

### `gapi/logger.go`

Unary gRPC interceptor aur HTTP gateway logger structured request information record karte hain. Zerolog key-value logs production search/observability ke liye useful hain.

## 8. Legacy Gin API: `api/`

Ye code current `main()` se active nahi hai, lekin architecture aur tests samajhne ke liye important hai.

### `api/server.go`

- `Server` struct store, config aur token maker rakhta hai.
- PASETO maker initialize hota hai.
- Gin router create hota hai.
- Public routes users/login/token renewal.
- Authenticated routes accounts/transfers.
- Validator registration hoti hai.

Routes:

```text
POST /users
POST /users/login
POST /tokens/renew_access
POST /accounts                 authenticated
GET  /accounts/:id             authenticated
GET  /accounts                 authenticated
POST /transfers                authenticated
```

### `api/account.go`

- Create request bind aur validate.
- Current authenticated username se owner set.
- Account create.
- Single account get me ownership check.
- List endpoint pagination query se accounts deta hai.

### `api/transfer.go`

- Amount positive hai ya nahi check.
- Source/destination account exist check.
- Currency matching check.
- Source account current user ka hai ya nahi check.
- `TransferTx` call.
- Success me transfer plus account/entry result return.

### `api/user.go`

- User create request receive.
- Password hash.
- User insert.
- Login me password verify.
- Access/refresh token generate.
- Refresh session persist.

### `api/token.go`

Refresh token ko sirf cryptographic token maan kar trust nahi karta. Matching database session load karke:

- session blocked nahi,
- username same,
- stored refresh token same,
- expiry valid,

check karta hai; phir new access token issue karta hai.

### `api/middleware.go`

`Authorization: Bearer <access-token>` read karta hai, token verify karta hai aur authenticated user payload context me set karta hai. Missing, malformed, unsupported ya invalid token tests me cover hain.

### `api/validator.go`

Gin/validator package me custom currency validation register karta hai. Supported values `USD`, `EUR`, `CAD` hain.

## 9. Authentication aur tokens

### Token interface

`token/maker.go` common `Maker` interface define karta hai. Is abstraction ki wajah se server JWT ya PASETO implementation ke saath kaam kar sakta hai.

### Payload

`token/payload.go` me token claims ka common structure hai:

- UUID token id.
- username.
- role.
- token type: access ya refresh.
- issued time.
- expiry time.

Payload validation expiry aur token type verify karti hai. Refresh token ko access endpoint par accept nahi karna chahiye.

### PASETO

`token/paseto_maker.go` v2 local symmetric encryption use karta hai. Server current runtime me isi maker ko use karta hai. Symmetric key ko secret rakhna zaroori hai; key leak hone par token trust boundary toot jati hai.

### JWT

`token/jwt_maker.go` HS256 JWT implementation hai. Iske tests hain, lekin current `gapi`/Gin runtime PASETO configure karta hai. JWT implementation compatibility/testing/reference ke liye repository me hai.

### Login flow

```text
Client -> LoginUser(username,password)
       -> GetUser(PostgreSQL)
       -> bcrypt compare
       -> Create access token
       -> Create refresh token
       -> Save refresh session
       <- tokens + expiry + user
```

### Protected request flow

```text
Authorization: Bearer <access-token>
       -> parse metadata
       -> verify token
       -> verify type and expiry
       -> verify allowed role
       -> handler business operation
```

### Refresh security

Refresh token session database me stored hai, isliye server session ko block karke refresh token revoke kar sakta hai. Current frontend refresh endpoint use nahi karta; legacy Gin route renewal provide karta hai.

## 10. Background worker, Redis aur email

### `worker/distributor.go`

`TaskDistributor` interface queueing contract define karta hai. `RedisTaskDistributor` Asynq client ke through task Redis me enqueue karta hai. Interface mock hone ki wajah se gRPC tests real Redis ke bina run kar sakte hain.

### `worker/processor.go`

- Redis/Asynq server options configure.
- Queue priority configure.
- Task type handler register.
- `Start()` worker goroutines start.
- Retry/error logger attach.
- `Shutdown()` graceful stop.

### `worker/task_send_verify_email.go`

`TaskSendVerifyEmail` task type string hai. Payload me username hota hai.

Distributor side:

1. payload JSON marshal.
2. Asynq task create.
3. queue me enqueue.
4. retry/queue metadata log.

Processor side:

1. JSON unmarshal; malformed payload retry skip.
2. username se user read.
3. 32-character random code create.
4. `verify_emails` row insert.
5. verification URL build.
6. HTML email Gmail sender ko pass.
7. success log.

Current URL hard-coded `http://localhost:8080` hai; deployed frontend/domain ke liye environment config banana hoga.

### Email verification end-to-end flow

```text
CreateUser RPC
  -> PostgreSQL user commit
  -> Redis task delayed by 10 seconds
  -> worker user read
  -> verification code DB me save
  -> Gmail email send
  -> user link click
  -> GET /v1/verify_email
  -> id/code/unused/expiry checks
  -> users.is_email_verified = true
```

External email failure ke baad verification row reh sakti hai aur retry naya row/code bana sakta hai. Ye behavior future cleanup/idempotency design ka candidate hai.

### `mail/sender.go`

Gmail SMTP connection, authentication, subject/body/recipient setup aur send operation encapsulate karta hai. Password source code me hardcode nahi hona chahiye; config/secret manager se aana chahiye.

### `worker/mock/`

Generated distributor mock gRPC unit tests me expected task enqueue verify karne ke liye use hota hai.

## 11. Utility packages

| File | Explanation |
|---|---|
| `util/password.go` | bcrypt hash aur compare; plaintext password store nahi |
| `util/password_test.go` | correct/wrong password behavior |
| `util/random.go` | test data aur random strings; `math/rand` security secrets ke liye suitable nahi |
| `util/currency.go` | supported currencies aur validator |
| `util/role.go` | `depositor` aur `banker` constants |
| `util/config.go` | Viper-backed typed configuration |
| `val/validator.go` | username/name/password/email/id/code rules |

Validation ka goal invalid data ko database ya business logic tak pahunchne se pehle reject karna hai. Handler validation ka result structured field violations me return karta hai.

## 12. Frontend: Vue 3 + Vite

### Frontend startup

- `frontend/index.html`: browser HTML shell.
- `frontend/src/main.ts`: Vue app create, router install, PrimeVue install, toast service install, mount.
- `frontend/src/App.vue`: current route render.
- `frontend/src/router/index.ts`: abhi root `/` route `HomeView` par map.
- `frontend/vite.config.ts`: Vue plugin, `@` alias aur dev server port `3000`.

### `src/store.ts`

`reactive<AuthState>` in-memory auth state rakhta hai:

- `user`
- `accessToken`
- `refreshToken`

`setUser` login response ko state me rakhta hai. `clearUser` logout par teenon values null karta hai. `readonly(state)` components ko direct mutation se bachata hai.

State localStorage/cookie me persist nahi hoti, isliye page reload par login disappear ho jata hai.

### `src/views/HomeView.vue`

- Toast component render.
- Agar `store.state.user` hai to `UserInfo`.
- Nahi hai to `LoginUser`.
- Logout event par toast aur `store.clearUser()`.

Ye frontend ka main conditional screen hai.

### `src/components/LoginUser.vue`

- `username`, `password`, `errorMessage` Vue refs.
- PrimeVue input components form render karte hain.
- `handleLogin` axios se `POST http://localhost:8080/v1/login_user` karta hai.
- Response ke user/tokens ko store me save karta hai.
- Success/error toast show karta hai.
- Current request me unnecessary `Authorization: none` header bheja jata hai; public login endpoint ke liye ise remove kiya ja sakta hai.

### `src/components/UserInfo.vue`

Logged-in user ki profile display karta hai aur logout event parent ko emit karta hai.

### Types and styling

- `src/types/user.ts`: user response shape.
- `src/types/auth_state.ts`: auth store shape.
- `src/assets/main.css`: global layout, PrimeVue theme/icon styling.
- `public/favicon.ico`: browser icon.
- `package.json`: `dev`, `build`, type-check, test, lint aur format scripts.
- `package-lock.json`: npm dependency lock.
- `tsconfig*.json`: TypeScript/Vue/Vitest compiler settings.
- `vitest.config.ts`: Vitest setup.
- `env.d.ts`: Vite type declarations.

Frontend me abhi registration, account list, account creation, transfer, refresh-token, email verification aur persistent auth UI nahi hai. Vitest script present hai, lekin source tree me frontend test files nahi dikhte.

## 13. Documentation assets: `doc/`

- `doc/db.dbml`: human-readable DBML data model; database docs generate karne ka source.
- `doc/schema.sql`: DBML se generated PostgreSQL schema snapshot.
- `doc/swagger/`: protobuf annotations se generated OpenAPI/Swagger JSON.
- `doc/statik/`: Swagger/static files ko Go binary me embed karne ke generated assets.
- `doc/PROJECT_DETAILED_GUIDE_HI.md`: ye beginner guide.

`db.dbml` aur migrations me consistency maintain karni chahiye. Schema change ke baad DBML/schema/Swagger regeneration check karein.

## 14. Docker local setup

### `docker-compose.yaml`

Services:

- `postgres`: PostgreSQL 14 Alpine, host port `5432`, persistent `data-volume`.
- `redis`: Redis 7 Alpine, internal service address `redis:6379`.
- `api`: Dockerfile se build; ports `8080` aur `9090` expose.

Container ke andar `DB_SOURCE` me `localhost` nahi, service name `postgres` use hota hai. Isi tarah Redis ke liye `redis:6379` use hota hai. Docker network DNS service names resolve karta hai.

`depends_on` startup order deta hai, readiness guarantee nahi. Isliye API entrypoint `wait-for.sh postgres:5432 -- /app/start.sh` use karta hai.

### `Dockerfile`

Build stage:

1. Go Alpine image.
2. `/app` workdir.
3. Source copy.
4. `go build -o main main.go`.

Run stage:

1. Small Alpine runtime image.
2. Compiled binary copy.
3. `app.env`, scripts aur migration files copy.
4. Ports metadata expose.
5. Entrypoint script command execute.

Multi-stage build final image me compiler/source dependencies nahi rakhta, image size kam karta hai.

### Shell helpers

- `start.sh`: command ko `exec` karta hai, jisse signals actual Go process tak pahunchte hain.
- `wait-for.sh`: dependency available hone tak polling karta hai, phir `--` ke baad command launch karta hai.

## 15. Kubernetes/EKS deployment

`eks/` production deployment manifests hain:

- `deployment.yaml`: API replicas, image, ports aur pod settings.
- `service.yaml`: internal ClusterIP service; HTTP/gRPC ports expose.
- `ingress-http.yaml`: HTTPS HTTP gateway hostname routing.
- `ingress-grpc.yaml`: gRPC hostname routing aur GRPC upstream protocol.
- `ingress-nginx.yaml`: nginx ingress class configuration.
- `issuer.yaml`: cert-manager Let’s Encrypt issuer.
- `aws-auth.yaml`: AWS identity ko Kubernetes access; `system:masters` bahut powerful permission hai.
- `install.sh`: ingress-nginx aur cert-manager installation helpers.

Kubernetes deployment ko DB/Redis network access, image registry, DNS, TLS, secrets aur AWS permissions chahiye. Manifests me production env values hardcode nahi honi chahiye; CI workflow AWS Secrets Manager se `app.env` generate karta hai.

## 16. CI/CD workflows

### `.github/workflows/test.yml`

Push/pull request on `master` par:

1. Go checkout/setup.
2. PostgreSQL service start.
3. Migration tool install.
4. Database migrations run.
5. `make test` run.

### `.github/workflows/deploy.yml`

`release` push par broadly:

1. Production secrets AWS Secrets Manager se read.
2. `app.env` generate.
3. Docker image build.
4. ECR me push.
5. kubeconfig update.
6. Kubernetes deployment update/apply.

Go module currently `go 1.24` declare karta hai, jabki Docker/CI configuration me Go 1.22 references ho sakte hain. Build environment versions align karna zaroori hai.

## 17. Tests ko kaise samjhein

### Run command

```bash
go test -v -cover -short ./...
```

`-short` external/integration-heavy tests ko skip karne ki permission deta hai. Database tests ke liye PostgreSQL aur migrations available honi chahiye.

### Test groups

| Test location | Kya verify hota hai |
|---|---|
| `api/account_test.go` | account create/get/list HTTP behavior |
| `api/transfer_test.go` | transfer validation aur transaction result |
| `api/user_test.go` | user create/login |
| `api/middleware_test.go` | bearer token edge cases |
| `db/sqlc/*_test.go` | CRUD, pagination, atomicity, deadlock behavior |
| `gapi/rpc_create_user_test.go` | validation, duplicate, tx callback/task |
| `gapi/rpc_update_user_test.go` | RBAC, validation, partial update |
| `token/*_test.go` | token creation, expiry, type, algorithm |
| `util/password_test.go` | bcrypt behavior |
| `mail/sender_test.go` | SMTP integration; external credential dependency |

Mock database aur mock task distributor unit tests ko deterministic banate hain. Expectations verify karti hain ki handler correct query aur async task call kar raha hai.

## 18. Makefile command reference

```bash
make network       # Docker network create
make postgres      # PostgreSQL container
make createdb      # simple_bank database create
make migrateup     # all migrations up
make migrateup1    # one migration up
make migratedown   # all migrations down
make migratedown1  # one migration down
make new_migration name=add_feature
make sqlc          # SQL -> Go generation
make mock          # GoMock generation
make proto         # protobuf/gRPC/gateway/Swagger generation
make db_docs       # DBML docs publish/build
make db_schema     # DBML -> schema.sql
make redis         # Redis container
make server        # go run main.go
make test          # tests with coverage
make evans         # interactive gRPC client
```

Windows par Make/Docker/Unix shell commands ke liye WSL2 ya compatible shell ki zaroorat ho sakti hai. README me original setup Homebrew-oriented hai; Windows environment me tools ko Docker Desktop/WSL ke through install karna practical hota hai.

## 19. First-time developer ke liye complete local run

### Option A: Docker Compose

Repository root me:

```bash
docker compose up --build
```

API container PostgreSQL readiness ka wait karega, app migrations run karega, Redis worker start karega, HTTP gateway `localhost:8080` aur gRPC `localhost:9090` par listen karega.

Frontend alag terminal me:

```bash
cd frontend
npm install
npm run dev
```

Frontend normally `http://localhost:3000` par chalega. CORS config me ye origin allowed hona chahiye.

### Option B: dependencies manually

```bash
make network
make postgres
make createdb
make migrateup
make redis
make server
```

Is mode me `app.env` ke addresses host machine ke hisaab se hone chahiye. PostgreSQL/Redis pehle running hone chahiye.

## 20. Example request flows

### Create user

```http
POST /v1/create_user
Content-Type: application/json

{
  "username": "alice",
  "password": "strong-password",
  "full_name": "Alice Example",
  "email": "alice@example.com"
}
```

Result: user DB me insert, verification task Redis me schedule, user response return.

### Login

```http
POST /v1/login_user
Content-Type: application/json

{
  "username": "alice",
  "password": "strong-password"
}
```

Result: access token, refresh token, session id, expiry timestamps aur user.

### Update user

```http
PATCH /v1/update_user
Authorization: Bearer <access-token>
Content-Type: application/json

{
  "username": "alice",
  "full_name": "Alice New Name"
}
```

Protobuf optional fields ki wajah se omitted field unchanged rehti hai. Password change par hash aur `password_changed_at` dono update hote hain.

### Verify email

```http
GET /v1/verify_email?email_id=1&secret_code=<code>
```

Valid, unused aur non-expired code user ko verified mark karta hai.

## 21. Common beginner concepts

### Package
Go package related `.go` files ka namespace hai. `package main` executable banata hai; baaki packages reusable code provide karte hain.

### Context
`context.Context` cancellation, deadline aur request-scoped metadata carry karta hai. Shutdown context cancel karne par workers/servers stop hote hain.

### Interface
Interface behavior contract hai. `db.Store` aur `worker.TaskDistributor` ke against handler program karta hai, concrete implementation ke against nahi. Isi se mocks possible hote hain.

### Pointer
`*pb.Request` ya `*Server` pointer original object ko refer karta hai; copies aur nil handling ka dhyan rakhein.

### Transaction
Transaction statements ka all-or-nothing group hai. Transfer me partial balance update unacceptable hai, isliye transaction mandatory hai.

### Goroutine
`errgroup.Go` long-running server/worker task ko concurrent run karta hai. Shared context lifecycle control karta hai.

### gRPC status
`codes.NotFound`, `codes.InvalidArgument`, `codes.PermissionDenied` jaise statuses machine-readable client behavior enable karte hain.

### Protobuf optional field
Update request me `nil` ka matlab field bheji hi nahi gayi. Empty string ka matlab field bheji gayi aur empty value validate/update karni hai. Is distinction ke liye `pgtype` nullable wrappers use hote hain.

## 22. Important current limitations and risks

1. `app.env` me secrets/development credentials hain; rotate and externalize them.
2. Verification URL localhost hard-coded hai.
3. Create-user transaction me Redis enqueue external side effect hai; DB rollback aur queue success diverge kar sakte hain.
4. Verify-email failures generic `Internal` status me map hote hain.
5. Frontend auth memory-only hai; reload par session lost.
6. Frontend refresh-token renewal implement nahi.
7. Frontend account/transfer/registration screens nahi.
8. Current `main` legacy Gin server start nahi karta.
9. Docker/CI Go versions aur `go.mod` version align karne chahiye.
10. `math/rand` se generated values security-sensitive secrets ke liye use nahi karne chahiye.
11. `aws-auth.yaml` me cluster-admin level `system:masters` access high risk hai.
12. SMTP integration test external Gmail setup par depend kar sakta hai.
13. `depends_on` alone service readiness guarantee nahi karta; wait helper important hai.
14. Balance integer hai; amount unit clearly define karke client/API contract me document karna chahiye.

## 23. Change karne ka correct workflow

### Database change

1. New migration create karein.
2. `.up.sql` aur `.down.sql` likhein.
3. `db/query` SQL update karein.
4. `make sqlc` run karein.
5. Store/transaction tests add/update karein.
6. DBML/schema docs regenerate karein.

### New API endpoint

1. `.proto` request/response/RPC define karein.
2. HTTP annotation add karein agar JSON route chahiye.
3. `make proto` run karein.
4. `gapi` me handwritten handler implement karein.
5. Validation aur authorization add karein.
6. Mock-based tests likhein.
7. Swagger output review karein.

### New background task

1. Unique task type constant.
2. JSON payload struct.
3. Distributor method/interface.
4. Processor handler registration.
5. Retry policy aur error behavior.
6. Unit tests with mock distributor/store.
7. External side effect idempotency review.

### Frontend feature

1. Backend contract confirm.
2. TypeScript response/request types.
3. API service call.
4. Reactive state update.
5. Loading/error/success states.
6. Route/component.
7. CORS/auth behavior check.

## 24. Debugging checklist

- Config error? `app.env` location, variable name aur duration format check karein.
- DB connection error? PostgreSQL running, port, database name aur credentials check karein.
- Migration error? Current migration version aur SQL syntax check karein.
- Redis error? `REDIS_ADDRESS` host-vs-container address check karein.
- CORS error? Frontend origin `ALLOWED_ORIGINS` me add karein.
- Login 404? User exists, password correct, gateway running aur request path check karein.
- Protected API unauthenticated? `Authorization: Bearer ...` exact format check karein.
- Email nahi aayi? Worker running, Redis task, Gmail credential aur spam folder check karein.
- Swagger missing? `doc/statik` regeneration aur embedded import check karein.
- Generated compile error? Source `.proto`/SQL change ke baad `make proto`/`make sqlc` run karein.
- Transfer inconsistency? Transaction result, account currency, amount sign aur DB rollback logs inspect karein.

## 25. One-page mental model

```text
Configuration
  -> main bootstrap
  -> PostgreSQL pool + automatic migrations
  -> sqlc Store
  -> gRPC server (:9090)
  -> grpc-gateway HTTP server (:8080)
  -> Redis/Asynq worker

HTTP request
  -> generated gateway
  -> gapi handler
  -> validation
  -> authorization (protected endpoints)
  -> db.Store / transaction
  -> protobuf response

Create user
  -> bcrypt password
  -> users insert
  -> async verification task
  -> verify_emails row + SMTP email

Login
  -> bcrypt compare
  -> PASETO access/refresh tokens
  -> sessions row

Transfer (legacy Gin/business DB layer)
  -> validate ownership/currency/amount
  -> one PostgreSQL transaction
  -> transfer row + two entries + two balance updates
  -> commit or rollback
```

## 26. Final takeaway

Is project ko samajhne ka best order hai:

1. `main.go` se startup samjho.
2. `app.env` aur `util/config.go` se configuration samjho.
3. `db/migration` se data model samjho.
4. `db/query` aur `db/sqlc` se persistence samjho.
5. `proto` aur `gapi` se active API samjho.
6. `token` aur `authorization` se security samjho.
7. `worker`/`mail` se asynchronous flow samjho.
8. `frontend` se client usage samjho.
9. Tests se expected behavior verify karo.
10. Docker/Kubernetes/CI se deployment lifecycle samjho.

Sabse important engineering rule: handwritten source (`proto`, `db/query`, `gapi`, `worker`, `api`, `frontend`) edit karein; generated output (`pb`, `db/sqlc`, mocks, statik assets) generation command se update karein. Har change ke baad focused test, full test aur relevant generation/build check run karein.
