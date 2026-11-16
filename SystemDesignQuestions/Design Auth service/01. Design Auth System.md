# Cortex Auth Service — Architecture & Design Notes

## 1. Introduction

Cortex is a distributed SaaS platform consisting of multiple microservices:

- Instance Service
- Backup Service
- Metrics Service
- Account Service
- User Service
- Organization Service

Different types of callers access these services:

| Caller | Example | Authentication |
|---|---|---|
| Customer | User accessing Cortex UI | OIDC |
| SRE / Platform Engineer | Internal platform operations | OIDC |
| Automation | Jenkins, CI/CD, scripts | Service Account |
| Internal Service | Temporal worker calling Instance Service | mTLS |

The challenge is to provide a consistent and secure authentication and authorization mechanism across all these services.

**Cortex Auth is responsible for authenticating external callers, resolving their permissions and issuing short-lived internal JWTs.**

Internal service-to-service calls use mTLS and do not depend on Cortex Auth.

---

## 2. Authentication vs Authorization

Before understanding the architecture, we need to distinguish two concepts.

### Authentication — Who are you?

Authentication verifies the identity of the caller.

Example:

Alice logs into Cortex using:

```text
Username: alice@example.com
Password: ********
```

The Identity Provider validates her credentials and confirms that the user is Alice.

### Authorization — What are you allowed to do?

Authorization determines what operations Alice can perform.

For example:

```text
Alice:
  instances.read       ALLOWED
  instances.create     ALLOWED
  instances.delete     DENIED
  backups.create      DENIED
```

A successfully authenticated user does not automatically have permission to perform every operation.

**Remember: Authentication establishes identity; authorization determines access.**

---

## 3. High-Level Architecture

```text
                  Customer / SRE
                        |
                        |
                 Cortex Web UI
                        |
                  OIDC Login
                        |
                        v
                Identity Provider
                (Keycloak / Okta)
                        |
                   OIDC JWT
                        |
                        v
                Envoy Gateway
                        |
                   ext_authz
                        |
                        v
                 Cortex Auth
                   /   |   \
                  /    |    \
             Valkey   Cortex   OIDC
                      Account  Provider
                  |
          Resolve Identity
          Resolve Permissions
          Generate Internal JWT
                  |
                  v
             Envoy Gateway
                  |
               mTLS
                  |
        +---------+----------+
        |         |          |
        v         v          v
     Instance  Backup     Metrics
     Service   Service    Service
```

Supporting components:

| Component | Responsibility |
|---|---|
| Envoy Gateway | Routes requests and invokes external authorization |
| Cortex Auth | Validates external credentials and resolves permissions |
| Identity Provider | Authenticates human users |
| Cortex Account | Source of truth for accounts, team members, roles and service accounts |
| Valkey | Shared cache for authentication and authorization data |
| Vault | Provisions signing keys and internal certificates |
| Backend Services | Enforce RBAC and resource ownership |

---

# Part A — External Authentication Flow

## 4. What Happens When a User Logs In?

Let's follow Alice's request step by step.

### Step 1: User enters credentials

Alice opens the Cortex UI and enters her username and password.

Cortex delegates authentication to an external Identity Provider (IdP).

Examples:

- Keycloak
- Okta
- Microsoft Entra ID

The IdP validates her credentials.

### Step 2: Identity Provider issues an OIDC token

After successful login, the IdP issues a signed OIDC token.

Conceptually:

```json
{
  "sub": "user_123",
  "email": "alice@example.com",
  "groups": ["dbaas-admins"],
  "iss": "https://identity.example.com",
  "exp": 1719503600
}
```

Important claims:

| Claim | Meaning |
|---|---|
| `sub` | Unique user identity |
| `email` | User email |
| `groups` | Identity Provider group membership |
| `iss` | Token issuer |
| `exp` | Token expiry |

The browser receives the token and uses it when calling Cortex APIs.

Example:

```http
GET /instances
Authorization: Bearer <OIDC_TOKEN>
```

**Important distinction:**

The OIDC token proves the user's identity. It does not automatically contain all the Cortex-specific permissions required by backend services.

### Step 3: Request reaches Envoy Gateway

The browser sends its API request to the public Envoy Gateway.

Before forwarding the request to the backend, Envoy invokes Cortex Auth through its `ext_authz` mechanism.

`ext_authz` means external authorization.

Envoy effectively asks:

> Is this request from a valid caller, and should I allow it to proceed?

### Step 4: Cortex Auth validates the OIDC token

Cortex Auth performs the following checks:

1. Extracts the Bearer token.
2. Retrieves the Identity Provider's public keys from JWKS.
3. Validates the token signature.
4. Validates issuer and expiry.
5. Checks the configured maximum JWT age.
6. Extracts the user identity, email, groups and account context.

The design uses:

```text
MAX_JWT_AGE = 15 minutes
```

This is separate from the OIDC token's `exp` claim.

Why?

Suppose an SRE is removed from the `dbaas-sres` group.

The old OIDC token may still contain that group because group membership is embedded in the token when it is issued.

The maximum-age check bounds how long Cortex accepts those potentially stale group claims.

### Step 5: Cortex Auth resolves permissions

Now Cortex Auth needs to understand what Alice can do inside Cortex.

It uses Cortex Account as the source of truth.

For example:

```text
User:
  user_123

Account:
  account_456

Assigned roles:
  admin

Permissions:
  instances.read
  instances.create
  instances.delete
  backups.read
  backups.create
  ...
```

Cortex Auth resolves:

- Customer roles and permissions.
- Platform roles and permissions from OIDC groups.
- Account context.
- Optional impersonation context.

It also checks the GDPR block key:

```text
gdpr_block:{account_id}
```

If the key exists, the request is denied.

---

## 5. Why Do We Need an Internal JWT?

This is the most important part of the architecture.

After Cortex Auth validates Alice's OIDC token and resolves her permissions, it needs to communicate that information to backend microservices.

One option would be to forward the original OIDC token to every backend.

But that creates unnecessary complexity:

- Every backend would need to understand the external Identity Provider.
- Every backend would need to validate external OIDC tokens.
- Backend services would need to understand how Cortex permissions are resolved.
- External identity-provider dependencies would spread across the platform.

Instead, Cortex Auth creates a new internal JWT.

### External OIDC JWT vs Internal JWT

Think of these as two different credentials with two different purposes.

| OIDC JWT | Internal JWT |
|---|---|
| Issued by external Identity Provider | Issued by Cortex Auth |
| Proves user identity | Carries Cortex identity and authorization context |
| Contains external identity claims | Contains Cortex account and permission claims |
| Validated by Cortex Auth | Validated by backend services |
| Has the IdP-defined lifetime | Expires after 10 seconds |

In simple English:

```text
OIDC JWT:

"The Identity Provider confirms this is Alice."

Internal JWT:

"Cortex confirms this is Alice.
 She belongs to account_456.
 These are her permissions."
```

### Example Internal JWT

Cortex Auth generates a short-lived token:

```json
{
  "iss": "cortex-auth.iad.dbaas.rax.io",
  "sub": "user_123",
  "iat": 1719500000,
  "exp": 1719500010,
  "act": "account_456",
  "tid": 42,
  "perm": "v1:000000007FFFFFFF:000000000000034F",
  "auth": "oidc"
}
```

Meaning:

| Claim | Meaning |
|---|---|
| `iss` | Cortex Auth issuer, including region |
| `sub` | User identity |
| `iat` | Issued-at timestamp |
| `exp` | Expiry timestamp |
| `act` | Cortex account ID |
| `tid` | Tenant ID for VictoriaMetrics/VictoriaLogs |
| `perm` | Encoded permission bitmask |
| `auth` | Authentication method |

Optional claims:

| Claim | Meaning |
|---|---|
| `scope` | Allowed instance UUIDs for a scoped service account |
| `imp` | Original caller identity during impersonation |

### Why only 10 seconds?

The internal JWT is intended for one request hop.

It is not a long-lived user session token.

A short lifetime:

- Limits the impact of a leaked internal token.
- Limits how long a previously issued authorization decision remains usable.
- Reduces the need for token revocation infrastructure.

The token is signed using Ed25519.

---

## 6. What Happens After the Internal JWT Is Created?

Cortex Auth returns the internal JWT to Envoy in:

```http
X-Cortex-JWT: <INTERNAL_JWT>
```

Envoy forwards the request to the appropriate backend service.

Example:

```http
GET /instances
X-Cortex-JWT: <INTERNAL_JWT>
```

The Instance Service receives the request.

It does not need to contact Cortex Auth again.

Instead, it:

1. Validates the JWT signature using its locally cached public keys.
2. Extracts the identity, account and permission claims.
3. Checks whether the caller has the required permission.
4. Checks resource ownership and scope.
5. Executes the handler if all checks pass.

Example:

```text
Request:
  DELETE /instances/instance_789

JWT:
  sub  = user_123
  act  = account_456
  perm = Alice's permissions

Checks:

1. Is JWT signature valid?
   YES

2. Does Alice have instances.delete?
   YES

3. Does instance_789 belong to account_456?
   YES

Result:
   ALLOW
```

If Alice has `instances.read` but not `instances.delete`, the request is denied.

**A valid JWT does not mean that every API operation is allowed.**

---

# Part B — Other Authentication Mechanisms

## 7. Three Authentication Paths

Cortex supports three authentication mechanisms.

```text
                   Incoming Request
                          |
              +-----------+-----------+
              |           |           |
             OIDC       Service       mTLS
             JWT        Account
              |           |           |
              v           v           v
          Human User   Automation   Internal
          / SRE        / CI/CD      Services
              |           |           |
              v           v           v
          Cortex Auth   Cortex Auth  PeerAuth
              |           |           |
              v           v           v
        Internal JWT  Internal JWT  Local RBAC
```

### 7.1 OIDC — Human Users

Used by:

- Customers
- SREs
- Platform engineers

Flow:

```text
User
 |
 | Login
 v
Identity Provider
 |
 | OIDC JWT
 v
Envoy Gateway
 |
 | ext_authz
 v
Cortex Auth
 |
 | Validate OIDC JWT
 | Resolve account and permissions
 | Generate internal JWT
 v
Backend Service
```

Platform roles are mapped from OIDC groups.

Example:

| AD Group | Platform Role |
|---|---|
| `dbaas-viewers` | viewer |
| `dbaas-dbas` | dba |
| `dbaas-cdes` | cde |
| `dbaas-sres` | sre |
| `dbaas-admins` | admin |

### 7.2 Service Accounts — Automation

Service accounts are used by:

- Jenkins
- CI/CD pipelines
- Scripts
- Applications
- Automated jobs

Example:

```http
Authorization: Bearer crtx_<TOKEN>
```

The `crtx_` prefix allows Cortex Auth to identify service-account tokens.

Flow:

```text
Automation
    |
    | crtx_<TOKEN>
    v
Envoy Gateway
    |
    v
Cortex Auth
    |
    | SHA-256(token)
    |
    | Lookup sa:{hash}
    v
Valkey
    |
    | HIT or MISS
    |
    | On MISS, call Cortex Account
    v
Resolve Service Account
    |
    | Validate expiry and revocation
    | Resolve roles and permissions
    v
Generate Internal JWT
    |
    v
Backend Service
```

Important design decisions:

- Token format starts with `crtx_`.
- SHA-256 hash is stored, not the raw token.
- Raw token is returned only once during creation.
- Mandatory expiry.
- Maximum expiry of one year.
- Tokens can be revoked.
- Service accounts receive customer permissions only.
- They cannot hold platform permissions.

### Instance Scope

A service account can optionally be restricted to specific instances.

Example:

```json
{
  "service_account": "sa_123",
  "account_id": "account_456",
  "instance_ids": [
    "instance_1",
    "instance_2"
  ]
}
```

Cortex Auth passes this scope through the internal JWT.

The backend enforces it.

For example:

```text
Requested instance:
  instance_3

Allowed instances:
  instance_1
  instance_2

Result:
  DENY
```

The backend also checks account ownership.

If the instance does not belong to the caller's account, or is outside the service account's scope, the backend returns `NOT_FOUND`.

This avoids revealing whether an inaccessible instance actually exists.

### 7.3 mTLS — Internal Service-to-Service Calls

Used when one internal service calls another.

Example:

```text
Temporal Worker
      |
      | mTLS connection
      v
Instance Service
```

Here, Cortex Auth is not involved.

Why?

The caller already has a trusted identity through its Vault-issued client certificate.

The receiving service:

1. Verifies the client certificate.
2. Extracts the peer certificate SAN.
3. Looks up permissions from local configuration.
4. Injects identity and permissions into the request context.
5. Runs the RBAC interceptor.

Example configuration:

```yaml
tls:
  peer_sans:
    temporal-worker-psmdb:
      permissions:
        - instances.read
        - instances.update
```

The certificate SAN identifies the calling service.

**Important: Internal mTLS calls do not require an internal JWT.**

This avoids making Cortex Auth a dependency for internal service communication.

---

# Part C — RBAC and Authorization

## 8. What Is RBAC?

RBAC stands for Role-Based Access Control.

Instead of assigning permissions individually to every user, permissions are grouped into roles.

Example:

```text
Role: viewer

Permissions:
  instances.read
  databases.read
  backups.read
  metrics.read
```

A user assigned this role receives those permissions.

### Built-in Customer Roles

| Role | Permissions | Purpose |
|---|---|---|
| viewer | `*.read` | Read-only access |
| operator | Read permissions + operational permissions | Day-to-day operations |
| admin | `*` | Full account control |

Customers can also create custom roles.

Example:

```text
Role: backup-manager

Permissions:
  backups.read
  backups.create
  backups.restore
```

This allows customers to implement fine-grained access control.

## 9. Platform RBAC vs Customer RBAC

Cortex has two authorization levels.

| Level | Purpose |
|---|---|
| Platform RBAC | Can this caller access the platform operation? |
| Customer RBAC | Can this caller perform this operation within the customer account? |

Examples:

```text
Customer Admin:
  Can manage resources in their own account.

SRE:
  Can perform authorized platform operations.
```

These are different privileges.

A customer administrator should not automatically become a platform administrator.

The design stores customer and platform permissions in separate words of the permission bitmask:

```text
Word 0 = Customer permissions
Word 1 = Platform permissions
```

The detailed platform permission registry is maintained in the separate Platform RBAC design.

## 10. Permission Bitmask

Why use a bitmask instead of passing a large list of permission strings in every JWT?

Consider:

```text
instances.read
instances.create
instances.delete
backups.read
backups.create
```

We can assign each permission a fixed bit position.

Example:

| Bit | Permission |
|---|---|
| 0 | instances.read |
| 1 | instances.create |
| 2 | instances.delete |
| 3 | instances.update |
| 4 | instances.actions |
| 5 | databases.read |
| 6 | databases.write |

A permission set is represented using `[]uint64`.

Each `uint64` holds 64 permission bits.

Example:

```go
type PermissionSet []uint64

func (ps PermissionSet) Has(bit uint16) bool {
    word := bit / 64
    pos := bit % 64

    if int(word) >= len(ps) {
        return false
    }

    return ps[word]&(1<<pos) != 0
}
```

### Combining Roles

Suppose a user has two roles:

```text
Role 1:
  instances.read
  instances.create

Role 2:
  backups.read
  backups.create
```

The effective permissions are the union of both roles.

Bitwise OR combines the permission bitmasks.

```text
Effective Permissions = Role 1 OR Role 2
```

### Internal JWT Representation

The permission claim uses:

```text
v1:<word0_hex>:<word1_hex>
```

Example:

```text
v1:000000007FFFFFFF:000000000000034F
```

- `v1` represents the bitmask format version.
- `word0` contains customer permissions.
- `word1` contains platform permissions.

Human-readable permission strings are the source of truth. The bitmask is derived.

Bit positions are append-only and must never be reused.

Unknown permissions must default to deny.

---

## 11. Backend Interceptor Chain

Every protected backend RPC has a required permission annotation.

Example:

```protobuf
rpc DeleteInstance(DeleteInstanceRequest)
    returns (DeleteInstanceResponse) {
  option (required_permission) = "instances.delete";
}
```

The interceptor checks whether the caller has the required permission.

External request:

```text
Envoy Gateway
     |
     v
Cortex Auth
     |
     | Internal JWT
     v
Backend Service
     |
     v
JWT Validation
     |
     v
RBAC Interceptor
     |
     v
Permission Check
     |
     v
Ownership Check
     |
     v
Handler
```

Internal request:

```text
Calling Service
     |
     | mTLS
     v
Receiving Service
     |
     v
PeerAuthMiddleware
     |
     v
RBAC Interceptor
     |
     v
Permission Check
     |
     v
Handler
```

Both paths share the same permission-checking mechanism.

The difference is how the caller's identity and permissions are established:

- External path: Internal JWT.
- Internal path: mTLS peer certificate SAN.

---

# Part D — Caching and Data Layer

## 12. Why Does Cortex Auth Need Caching?

Cortex Auth needs information from Cortex Account:

- Service account details.
- User-to-role assignments.
- Role definitions.
- Platform role mappings.
- Platform role definitions.

Fetching this information from Cortex Account on every request would create unnecessary network traffic and load.

Therefore, Cortex Auth uses a tiered cache.

```text
                Request
                   |
                   v
             L1 In-Process
                Cache
                   |
             Cache HIT?
              /       \
            YES       NO
             |         |
             v         v
           Return    Valkey
                       |
                  Cache HIT?
                   /     \
                 YES     NO
                  |       |
                  v       v
                Return  Singleflight
                            |
                            v
                      Cortex Account
                            |
                            v
                       Update Cache
                            |
                            v
                          Return
```

### Three Layers

| Layer | Component | Purpose |
|---|---|---|
| L1 | In-process cache | Fast local lookup |
| L2 | Valkey | Shared distributed cache |
| L3 | Cortex Account | Source of truth |

### Cache TTLs

| Data | L1 TTL | Valkey TTL |
|---|---:|---:|
| Service account | 10 seconds | 60 seconds |
| User role assignment | 10 seconds | 60 seconds |
| Role definition | 10 seconds | 60 seconds |
| OIDC JWKS | 1 hour | 300 seconds |

L1 cache:

- Uses `hashicorp/golang-lru/v2/expirable`.
- Maximum 10,000 entries.
- Exists separately inside each Cortex Auth replica.

Impersonation session keys are excluded from L1 because they must always be checked fresh.

## 13. Singleflight Pattern

Consider 100 concurrent requests arriving with the same service account token.

The token is not present in L1 or Valkey.

Without protection:

```text
Request 1   ---> Cortex Account
Request 2   ---> Cortex Account
Request 3   ---> Cortex Account
...
Request 100 ---> Cortex Account
```

This creates 100 identical RPC calls.

With Go's `singleflight`:

```text
Request 1   \
Request 2    \
Request 3     ---> Singleflight ---> One RPC
...          /                    to Cortex Account
Request 100 /
```

Only one request performs the actual lookup.

Other callers wait for the same result.

Implementation:

```go
group := singleflight.Group{}

result, err, shared := group.Do(key, func() (interface{}, error) {
    return accountClient.LookupServiceAccount(ctx, hash)
})
```

The design uses `DoChan` and a detached context with a five-second timeout so cancellation of the first caller does not cancel the shared lookup for everyone.

## 14. Concurrency Limiter

Singleflight handles duplicate requests for the same cache key.

But what if thousands of requests miss the cache for thousands of different keys?

Cortex Auth uses a weighted semaphore to limit concurrent calls to Cortex Account.

Default:

```text
Maximum concurrent calls = 15
```

If the semaphore is full, Cortex Auth returns `UNAVAILABLE` instead of creating an unbounded queue.

This protects Cortex Account from overload.

## 15. Cache Invalidation and Revocation

Suppose a service account is revoked.

What happens?

Cortex Account publishes a Valkey `DEL` for the affected cache key.

```text
Service Account Revoked
          |
          v
     Cortex Account
          |
          v
      Valkey DEL
          |
          v
    Distributed Cache
      entry removed
```

However, every Cortex Auth replica has its own L1 cache.

Valkey deletion does not directly clear those L1 entries.

Therefore:

| Scenario | Expected stale-data window |
|---|---|
| Valkey DEL succeeds | L1 may serve stale data for up to 10 seconds |
| Valkey DEL fails | Valkey TTL is 60 seconds, with up to 10 additional seconds from L1 |

This is an important trade-off between cache performance and immediate revocation.

Impersonation sessions are excluded from L1 so their revocation can be checked immediately against Valkey.

## 16. What If Valkey Is Down?

Cortex Auth falls back to Cortex Account.

```text
Request
   |
   v
L1 Cache
   |
   | MISS
   v
Valkey
   |
   | UNAVAILABLE
   v
Singleflight
   |
   v
Cortex Account
```

L1 absorbs most traffic.

Singleflight collapses duplicate cache misses.

The semaphore limits concurrent calls.

The goal is to avoid a cascading failure where Valkey downtime overloads Cortex Account.

### Degradation Contract

| Valkey | L1 | Behaviour |
|---|---|---|
| Healthy | Warm | Normal cache operation |
| Healthy | Cold | Valkey warms L1 |
| Down | Warm | L1 absorbs most requests |
| Down | Cold | Singleflight and semaphore protect Cortex Account |

Cortex Auth has no PostgreSQL database of its own.

Cortex Account owns the persistent data.

---

# Part E — Key Management and Security

## 17. JWT Signing and Verification

Cortex Auth signs internal JWTs using an Ed25519 private key.

The key is provisioned by Vault Transit.

```text
                  Vault
                    |
                    | Provision signing key
                    v
               Cortex Auth
                    |
                    | Load private key at startup
                    |
                    | Sign internal JWT locally
                    v
               Internal JWT
                    |
                    v
               Backend Services
                    |
                    | Verify signature
                    v
                Public Key
```

Important design decisions:

- One signing key per region.
- Private key is loaded at startup.
- JWT signing happens locally.
- Vault is not called for every request.
- Cortex Auth exposes `/.well-known/jwks.json`.
- Backends fetch public keys every 30 seconds.
- Initial JWKS fetch is a readiness requirement.

### Why Asymmetric Signing?

With Ed25519:

- Cortex Auth owns the private signing key.
- Backend services receive public keys.
- Backends can validate tokens.
- Backends cannot create valid tokens themselves.

This is a good fit for a distributed microservice architecture.

## 18. Key Rotation

Recommended rotation interval:

```text
90 days
```

Rotation process:

1. Vault provisions a new key version.
2. Cortex Auth loads the latest key.
3. JWKS publishes old and new public keys.
4. Cortex Auth signs new JWTs using the new private key.
5. Backends refresh JWKS within 30 seconds.
6. Old tokens remain verifiable during the overlap.
7. Once the 10-second token lifetime has passed, old-key tokens are no longer expected to be in flight.

This supports zero-downtime key rotation.

## 19. Important Security Controls

| Control | Purpose |
|---|---|
| mTLS | Secure communication between services |
| Ed25519 signing | Protect internal JWT integrity |
| 10-second JWT expiry | Limit exposure of leaked tokens |
| SHA-256 service account hashing | Avoid storing raw tokens |
| Valkey TLS | Protect cached identities and permissions |
| CiliumNetworkPolicy | Restrict Valkey access |
| GDPR block key | Block requests for restricted accounts |
| Default-deny RBAC | Unknown permissions are denied |
| Instance ownership checks | Prevent cross-account access |
| Inbound JWT header stripping | Prevent client-supplied internal JWT injection |

**Important header security rule:**

Envoy must strip any incoming client-supplied `X-Cortex-JWT` header before invoking `ext_authz`.

Backends must reject the internal JWT on connections that are not protected by mTLS.

This prevents clients from attempting to inject their own internal authorization context.

---

# Part F — Additional Design Considerations

## 20. Impersonation

Impersonation allows an authorized platform user to act in the context of a customer account.

Example:

An SRE needs to troubleshoot an issue in account `account_456`.

The request includes:

```http
X-Cortex-Impersonate: <SESSION_ID>
```

Cortex Auth:

1. Looks up `impersonate:{session_id}` in Valkey.
2. Verifies the caller has `platform.impersonate`.
3. Checks that the session has not expired.
4. Sets the target account as the active account context.
5. Adds the original caller identity to the `imp` claim.

Example:

```json
{
  "sub": "user_sre123",
  "act": "account_456",
  "imp": "user_sre123"
}
```

The original caller identity is retained for auditing.

## 21. Rate Limiting

Rate limiting protects Cortex from excessive or abusive traffic.

### External Traffic

| Scope | Default Limit |
|---|---:|
| Per account | 100 requests/sec |
| Per service account | 50 requests/sec |
| Public endpoints per source IP | 10 requests/sec |

### Internal Traffic

| Scope | Default Limit |
|---|---:|
| Per calling service certificate SAN | 200 requests/sec |
| Per procedure | Configurable |

Example:

A Temporal worker repeatedly retries a failed operation in a tight loop.

Per-caller rate limiting prevents that worker from consuming all backend capacity.

Circuit breakers in Envoy provide another layer of protection.

## 22. Public Endpoints

The Auth Service exposes a small set of public procedures that do not require authentication:

```text
/v1/auth/oidc-config
/v1/health
/v1/version
```

Other protected external requests go through the authentication flow.

The design uses Bearer token authentication rather than cookies, so browser cookies are not used for API authentication or session management.

---

# Part G — Implementation Structure

## 23. Proposed Go Package Structure

The design identifies the following main files:

```text
proto/
  cortex/
    auth/
      v1/
        auth.proto

internal/
  service/
    auth.go

  interceptor/
    auth.go
    rbac.go

  signing/
    keyset.go
    issuer.go
    jwks.go
    jwksclient.go

  permissions/
    registry.go
    bitmask.go

  cache/
    l1.go
    valkey.go

  identity/
    oidc.go
    service_account.go
```

The `identity`, `permissions` and `cache` directories above are suggested organizational subdivisions; the design explicitly lists the proto, service, interceptor and signing paths.

### Responsibilities

| Package | Responsibility |
|---|---|
| `service/auth.go` | Public AuthService RPC implementations |
| `interceptor/auth.go` | Authentication and credential processing |
| `interceptor/rbac.go` | Permission enforcement |
| `signing/keyset.go` | Load and manage signing keys |
| `signing/issuer.go` | Create and sign internal JWTs |
| `signing/jwks.go` | Expose public keys |
| `signing/jwksclient.go` | Fetch and cache public keys in backends |
| `permissions` | Permission registry and bitmask operations |
| `cache` | L1 and Valkey caching |

---

# Part H — Important Takeaways

## 24. Things to Remember

1. OIDC authenticates human users through an external Identity Provider.
2. Cortex Auth validates OIDC tokens and resolves Cortex-specific permissions.
3. The internal JWT is a short-lived authorization credential for backend services.
4. Backend services validate the internal JWT locally; they do not call Cortex Auth for every request.
5. Service accounts are used for automation and receive customer permissions only.
6. Internal service-to-service communication uses mTLS, not Cortex Auth or internal JWTs.
7. RBAC is enforced inside each backend through interceptors.
8. Permission sets use variable-width `[]uint64` bitmasks.
9. Customer and platform permissions occupy separate words.
10. Cortex Account is the source of truth; Cortex Auth is stateless.
11. L1 + Valkey + Singleflight reduce load on Cortex Account.
12. Vault provisions signing keys; Cortex Auth signs JWTs locally.
13. Backends verify signatures using cached public keys.
14. Resource ownership and service-account instance scope are enforced by the backend.

## 25. One-Minute Architecture Summary

```text
EXTERNAL REQUEST:

User
 |
 | OIDC JWT
 v
Envoy Gateway
 |
 | ext_authz
 v
Cortex Auth
 |
 | Validate identity
 | Resolve account and roles
 | Build permission bitmask
 | Sign internal JWT (10 sec)
 v
Envoy Gateway
 |
 | mTLS + X-Cortex-JWT
 v
Backend Service
 |
 | Verify signature
 | Check RBAC
 | Check ownership/scope
 v
Execute API


INTERNAL REQUEST:

Temporal Worker
 |
 | mTLS client certificate
 v
Backend Service
 |
 | Verify certificate
 | Extract SAN
 | Resolve local permissions
 | Check RBAC
 v
Execute API
```

**The central design principle:**

Cortex Auth centralizes external identity validation and permission resolution, while backend services remain responsible for enforcing authorization on their own APIs and resources. Internal service communication relies on mTLS and local permission configuration rather than a central authentication dependency.
