# 07: Codebase Deep Dive & Core File Walkthrough

> **Document ID**: `07_CODEBASE_DEEP_DIVE`  
> **System Name**: FreelancerHub  
> **Scope**: In-Depth Source Code Breakdown of Critical Files, Algorithms, and Patterns  

---

## 1. Edge Layer: `load-balancer/loadBalancer.js`

- **File Path**: [load-balancer/loadBalancer.js](file:///e:/DEVOPS/load-balancer/loadBalancer.js)
- **Role**: Software-defined reverse proxy and edge traffic balancer.
- **Lines of Code**: ~180 lines.
- **Dependencies**: `express`, `http-proxy`, `crypto`, `helmet`, `cookie-parser`, `cors`.

### Critical Functions & Business Logic:

#### 1. Cryptographic Sticky Session Engine (`encrypt` / `decrypt`)
```javascript
const COOKIE_NAME = "Server_Id";
const SECRET_KEY = crypto.createHash("sha256").update(process.env.ENCRYPTION_KEY).digest();
const ALGORITHM = "aes-256-cbc";
const IV = crypto.randomBytes(16); // Static IV generated at boot

const encrypt = (text) => {
  const cipher = crypto.createCipheriv(ALGORITHM, Buffer.from(SECRET_KEY), IV);
  let encrypted = cipher.update(text, "utf8", "hex");
  encrypted += cipher.final("hex");
  return encrypted;
};

const decrypt = (text) => {
  try {
    const decipher = crypto.createDecipheriv(ALGORITHM, Buffer.from(SECRET_KEY), IV);
    let decrypted = decipher.update(text, "hex", "utf8");
    decrypted += decipher.final("utf8");
    return decrypted;
  } catch (err) {
    logger.error("[DECRYPT ERROR]: Invalid cookie data");
    return null;
  }
};
```
- **How it Works**: When a client without a `Server_Id` cookie connects, the load balancer selects the least loaded server, encrypts the server URL using AES-256-CBC, and attaches an `HttpOnly`, `Secure`, `SameSite=None` cookie. On subsequent requests, the cookie is decrypted to route the client back to the same backend node.
- **Architectural Vulnerability**: `IV` is generated once at module initialization! In CBC mode, reusing the same initialization vector across multiple plaintexts allows an eavesdropper to perform differential cryptoanalysis or replay attacks.

#### 2. Least-Connections Calculation (`getLeastLoadedServer`)
```javascript
const getLeastLoadedServer = () => {
  const activeServers = servers.filter((server) => server.active);
  if (activeServers.length === 0) return null;
  return activeServers.reduce((prev, curr) =>
    prev.connections < curr.connections ? prev : curr
  );
};
```
- **Architectural Defect Discovered**: In `loadBalancer.js`, `assignedServer.connections++` increments on each request. However, `assignedServer.connections--` is ONLY present inside the proxy error callback! Successful requests never decrement the counter, causing active connection numbers to grow monotonically until server restart.

---

## 2. Machine Learning Gateway: `freelancer-ml-project/api_gateway_integration.py`

- **File Path**: [freelancer-ml-project/api_gateway_integration.py](file:///e:/DEVOPS/freelancer-ml-project/api_gateway_integration.py)
- **Role**: FastAPI microservice serving predictions and TLAM matching.
- **Port**: 5001.
- **Dependencies**: `fastapi`, `uvicorn`, `pydantic`, `scikit-learn`, `numpy`, `joblib`.

### Critical Algorithms & Logic:

#### 1. TLAM Multiplicative Scoring Engine (`compute_tlam`)
```python
TLAM_WEIGHTS = {
    "skill_score": 0.349,
    "semantic_score": 0.244,
    "context_score": 0.150,
    "maturity_score": 0.148,
    "behavior_score": 0.075,
    "budget_score": 0.034,
}

def compute_tlam(features: Dict[str, float]) -> float:
    # Capability combines direct skills, semantic text fit, and domain context
    capability = (
        features["skill_score"] * 0.45 +
        features["semantic_score"] * 0.35 +
        features["context_score"] * 0.20
    )
    # Reliability combines experience maturity and historical behavior
    reliability = (
        features["maturity_score"] * 0.65 +
        features["behavior_score"] * 0.35
    )
    # Alignment reflects budget harmony
    alignment = features["budget_score"]

    # Strict Cobb-Douglas style multiplicative formula:
    score = (
        (capability ** TLAM_WEIGHTS["skill_score"]) *
        (reliability ** TLAM_WEIGHTS["maturity_score"]) *
        (alignment ** TLAM_WEIGHTS["budget_score"])
    )
    return max(0.0, min(1.0, float(score)))
```
- **Algorithm Analysis**:
  - If a candidate has **no matching skills** (`skill_score == 0`), and low semantic similarity, `capability` approaches 0.
  - Raising a near-zero base to the 0.349 power yields 0, completely disqualifying the candidate regardless of high reliability or budget alignment.
  - This solves the classic freelance marketplace flaw where spam bidders with high historical ratings dominate categories they have zero technical competence in.

#### 2. Dual-Engine Reranking (`/api/ml/match`)
The endpoint iterates through the candidate pool, calculates both `compute_tlam()` and `compute_weighted_score()`, attaches individual feature breakdowns with learned weights, and returns a sorted payload with full model transparency metadata.

---

## 3. ML Model Training: `freelancer-ml-project/train_success_model.py`

- **File Path**: [freelancer-ml-project/train_success_model.py](file:///e:/DEVOPS/freelancer-ml-project/train_success_model.py)
- **Role**: Automated model selection and training pipeline for project success prediction.
- **Dependencies**: `pandas`, `scikit-learn`, `joblib`.

### Key Class: `FreelancerSuccessPredictor`
- **Feature Scaling**: Uses `StandardScaler()` to normalize disparate scales (`experience_years`: 0–15 vs `completion_rate`: 0.0–1.0).
- **Automated Algorithm Tournament**:
```python
models = {
    'Logistic Regression': LogisticRegression(random_state=42),
    'Random Forest': RandomForestClassifier(n_estimators=100, random_state=42),
    'Gradient Boosting': GradientBoostingClassifier(n_estimators=100, random_state=42)
}
# Trains each model on 80% split, evaluates AUC-ROC on 20% test split,
# and persists the highest-scoring model to success_predictor.pkl
```
- **Serialization**: Persists a dictionary `{ model, scaler, feature_names }` via Joblib, guaranteeing that inference code in the API gateway applies identical feature transformations.

---

## 4. API Client & Gateway Abstraction: `frontend/src/lib/api.ts`

- **File Path**: [frontend/src/lib/api.ts](file:///e:/DEVOPS/frontend/src/lib/api.ts)
- **Role**: Centralized Axios client managing authentication, error toasts, and domain API calls.
- **Lines of Code**: 800+ lines.

### Critical Mechanics:
1. **Axios Request Interceptor**:
```typescript
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem("auth_token");
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});
```
2. **Idempotency Key Generator**:
```typescript
const generateIdempotencyKey = (): string => {
  return `${Date.now()}-${Math.random().toString(36).substring(2, 18)}`;
};
```
Guarantees that retry attempts on network timeouts for financial transactions (Escrow funding, payout withdrawals) do not trigger duplicate ledger deductions on the backend.

---

## 5. Contract State Machine: `frontend/src/pages/Agreements/AgreementDetails.tsx`

- **File Path**: [frontend/src/pages/Agreements/AgreementDetails.tsx](file:///e:/DEVOPS/frontend/src/pages/Agreements/AgreementDetails.tsx)
- **Role**: Complete digital agreement lifecycle manager.
- **Lines of Code**: 1,486 lines.

### Key Logic:
1. **Multi-State Status Badge Mapping**: Maps contract status across `draft`, `pending_freelancer`, `pending_client`, `active`, `completed`, `cancelled`, `disputed`, `amended`.
2. **Amendment & Re-Signing Protocol**:
   - When terms change, `POST /agreement/:id/amend` resets signatures, increments `version`, and stores `amendmentReason`.
   - The contract cannot transition to `active` until both `clientSignature.signed` and `freelancerSignature.signed` evaluate to `true`.
3. **Cryptographic Integrity Display**: Renders the `contentHash` prominently, allowing both parties to verify that contract clauses have not been tampered with post-signing.

---

## 6. Financial Gateway Integration: `frontend/src/pages/Payment/PaymentCheckout.tsx`

- **File Path**: [frontend/src/pages/Payment/PaymentCheckout.tsx](file:///e:/DEVOPS/frontend/src/pages/Payment/PaymentCheckout.tsx)
- **Role**: Client checkout page coordinating with Razorpay and Escrow.
- **Lines of Code**: 464 lines.

### Step-by-Step Payment Execution:
1. Fetches project details and derives platform commission from `useSiteSettings()`.
2. Calls `api.createPaymentOrder(projectId, totalAmount)` to receive a Razorpay Order ID.
3. Spawns Razorpay Checkout modal via client-side script injection:
```typescript
const options = {
  key: import.meta.env.VITE_RAZORPAY_KEY_ID,
  amount: order.amount,
  currency: "INR",
  name: siteName,
  description: `Escrow Pre-Funding for ${project.title}`,
  order_id: order.id,
  handler: async function (response: any) {
    // Verifies payment signature on backend
    const verifyRes = await api.verifyPayment({
      orderId: order.id,
      paymentId: response.razorpay_payment_id,
      signature: response.razorpay_signature,
    });
    if (verifyRes.status === "success") {
      setPaymentSuccess(true);
      toast.success("Payment verified! Escrow funded.");
    }
  }
};
const rzp = new window.Razorpay(options);
rzp.open();
```

---

## 7. Milestone SLA & Deliverable Manager: `frontend/src/pages/Milestones/MilestoneManager.tsx`

- **File Path**: [frontend/src/pages/Milestones/MilestoneManager.tsx](file:///e:/DEVOPS/frontend/src/pages/Milestones/MilestoneManager.tsx)
- **Role**: Milestone creation, deliverable submission, and automated SLA bonus/penalty tracking.
- **Lines of Code**: 689 lines.

### Critical Calculations:
- Visualizes status with dynamic badges: `pending`, `in_progress`, `submitted`, `confirmed`, `revision`, `released`, `disputed`.
- **Revision Limits**: Enforces a strict maximum of 3 revisions (`revisionCount <= maxRevisions`) before prompting the user to escalate to a formal dispute, preventing indefinite unpaid rework.

---

## 8. Real-Time Encrypted Chat: `frontend/src/components/ChatModal1.tsx`

- **File Path**: [frontend/src/components/ChatModal1.tsx](file:///e:/DEVOPS/frontend/src/components/ChatModal1.tsx)
- **Role**: End-to-end encrypted WebSocket messaging modal with policy filters.
- **Lines of Code**: ~370 lines.

### Key Logic:
1. **Anti-Circumvention Regex Filter**:
```typescript
const containsSensitiveInfo = (message: string): boolean => {
  const phoneRegex = /\b\d{10,15}\b/;
  const emailRegex = /\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/;
  return phoneRegex.test(message) || emailRegex.test(message);
};
```
Prevents sending phone numbers or emails in chat to protect platform escrow commissions.
2. **SubtleCrypto AES-GCM Decryption**:
```typescript
async function decryptMessage(encryptedMessage: string, hexKey: string) {
  const [ivHex, encryptedText, authTagHex] = encryptedMessage.split(":");
  const iv = hexToUint8Array(ivHex);
  const encrypted = hexToUint8Array(encryptedText);
  const authTag = hexToUint8Array(authTagHex);
  const encryptedWithAuthTag = new Uint8Array([...encrypted, ...authTag]);

  const decrypted = await crypto.subtle.decrypt(
    { name: "AES-GCM", iv },
    key,
    encryptedWithAuthTag
  );
  return new TextDecoder().decode(decrypted);
}
```

---

## 9. Two-Tier Route Guards: `PorperCheck.tsx` & `SecurityLayer.tsx`

- **File Paths**: 
  - [PorperCheck.tsx](file:///e:/DEVOPS/frontend/src/components/Security/PorperCheck.tsx) (Typo in original filename for `ProperCheck.tsx`)
  - [SecurityLayer.tsx](file:///e:/DEVOPS/frontend/src/components/Security/SecurityLayer.tsx)

### Core Protection:
- `PorperCheck.tsx` uses custom hook `useCheckPermission(role)` which calls `/security/checkAuth/permission/:role`. If rejected, it immediately redirects to `/forbidden`. If permitted, it wraps children in `ProfileSetupGuard`, ensuring incomplete profiles cannot browse or bid.
- `SecurityLayer.tsx` disables right-click, intercepts DevTools shortcut keys (F12, Ctrl+Shift+I), and checks if `window.outerWidth - window.innerWidth > 200` to wipe the DOM with an access denied screen.

---

## 10. Admin Governance: `DisputesView.tsx` & `ViolationsView.tsx`

- **File Paths**:
  - [DisputesView.tsx](file:///e:/DEVOPS/Freelancer-Admin/src/components/admin/DisputesView.tsx)
  - [ViolationsView.tsx](file:///e:/DEVOPS/Freelancer-Admin/src/components/admin/ViolationsView.tsx)
- **Role**: Judicial arbitration and automated fraud moderation.

### Core Capabilities:
- **Dispute Resolution**: Allows platform operators to inspect uploaded evidence (screenshots, contracts), review filed reasons, execute clawbacks (`POST /admin/projects/:id/clawback`), or distribute split refunds.
- **Violation Enforcement**: Displays flagged chat messages captured by anti-circumvention filters, assigns strike penalties, freezes wallets (`POST /admin/users/:userId/wallet/freeze`), or bans repeat offenders.
