how TLS integrates with service accounts:


# 🔐 TLS with Service Accounts in OpenShift

## How Pod-to-API Server Communication Works

When a pod (using a service account) needs to interact with the API server (e.g., list pods), the following flow occurs:

```
Pod (service account)
  │
  ├── 1. TLS connection → API Server presents its cert
  ├── 2. Pod validates cert using CA cert ✅
  ├── 3. Pod sends Bearer token (JWT)
  ├── 4. API Server validates token ✅ (Authentication)
  ├── 5. API Server checks RBAC ✅ (Authorization)
  └── 6. API Server returns result over TLS 🔒
```

---

## Step-by-Step Breakdown

### 1️⃣ TLS Connection
- The pod initiates an **HTTPS** connection to the API server.
- The API server presents its **TLS server certificate**.

### 2️⃣ CA Certificate Validation
- The pod validates the API server's cert using the **CA certificate** auto-mounted at:
  ```
  /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  ```

### 3️⃣ Bearer Token (JWT)
- The pod sends its **service account token** in the request header:
  ```
  Authorization: Bearer <token>
  ```

### 4️⃣ Authentication
- The API server **validates the JWT token** (checks signature, expiry, audience).
- If valid → identity is confirmed.

### 5️⃣ RBAC Authorization
- The API server checks **RBAC policies**:
  - Does this service account have a `Role` / `ClusterRole` bound via `RoleBinding` / `ClusterRoleBinding`?
  - Is the action (`list pods`) permitted?
- ✅ **Allowed** → executes the request.
- ❌ **Denied** → returns `403 Forbidden`.

### 6️⃣ Response
- The result is sent back over the **same TLS-encrypted connection**.

---

## Summary

| Layer | Purpose | Mechanism |
|-------|---------|-----------|
| **TLS** | Secures the channel (encryption + server identity) | CA cert validation |
| **Authentication** | Verifies *who are you?* | Service account JWT token |
| **Authorization** | Verifies *what are you allowed to do?* | RBAC (Roles + RoleBindings) |

> **Key Takeaway:** TLS secures the channel, the service account token handles authentication, and RBAC handles authorization. All three work together to restrict pod-to-API communication.
