# 11: Quick Revision Notes & High-Yield Cheatsheet

> **Document ID**: `11_QUICK_REVISION_NOTES`  
> **System Name**: FreelancerHub  
> **Format**: High-Yield, Last-Minute Technical Interview Cheatsheet  

---

## 1. System in 3 Bullet Points

1. **What is it?**: A distributed freelancing marketplace and escrow platform engineered to eliminate trust deficits, fake profiles, fee circumvention, and payment disputes.
2. **How is it architected?**: 4 decoupled services—Custom Node.js Edge Reverse Proxy (:4000), Python FastAPI ML Microservice (:5001), React TypeScript Client App (:8080), and React Admin Portal (:8081)—interfacing with a Node.js Core Backend (:5000) and MongoDB (:27017).
3. **What makes it unique?**: The proprietary **TLAM (Trust-Layer Adaptive Matching)** multiplicative engine ($C^{0.349} \cdot R^{0.148} \cdot A^{0.034}$), pre-funded Razorpay escrow, cryptographic contract hashing, automated milestone SLA early-delivery bonuses/penalties, and end-to-end encrypted WebSocket chat.

---

## 2. Architecture & Port Reference

```
[Browser Client :8080] ──> [Custom Load Balancer :4000] ──> [Core Backend :5000]
[Admin Portal    :8081] ──> [Core Backend :5000] ──> [FastAPI ML API :5001]
                                                 └──> [MongoDB :27017]
```

- **Port 4000**: Node.js Reverse Proxy (`loadBalancer.js`). Least-connections, AES-256-CBC encrypted sticky cookies, 100 req/min rate limit.
- **Port 5000**: Core Backend. REST API (`/api/vi`), WebSocket server (`/chat/:id`), Razorpay verification.
- **Port 5001**: ML Microservice (`api_gateway_integration.py`). FastAPI, Uvicorn, TLAM matching, Random Forest Success Predictor, Gradient Boost Fake Profile Detector.
- **Port 8080 / 8081**: React Vite SPAs. Client marketplace and Admin governance portal.
- **Port 27017**: MongoDB document database.

---

## 3. Technology Stack at a Glance

| Layer | Primary Technologies | Key Libraries |
|---|---|---|
| **Frontend** | React 18, Vite 6, TypeScript 5.8 | Tailwind CSS, Radix UI, TanStack Query v5, Redux Toolkit, Redux Persist |
| **Cryptography** | Web Crypto API (SubtleCrypto) | AES-GCM 256-bit (Chat), AES-256-CBC (Sticky Cookies), SHA-256 (Contracts) |
| **Edge Proxy** | Node.js 18, Express 4 | `http-proxy`, Helmet, Cookie-Parser, CORS |
| **Machine Learning** | Python 3.10+, FastAPI, Uvicorn | Scikit-learn 1.3, SBERT (`all-MiniLM-L6-v2`), Pandas 2.2, NumPy 1.24, Joblib |
| **Database** | MongoDB | Mongoose ODM, Multi-Document ACID Transactions |
| **External APIs** | Razorpay (Escrow Payments), GitHub REST API (Skill Codebase Profiling) |

---

## 4. Top 10 Critical APIs to Memorize

1. `POST /api/vi/{role}/login`: User authentication with XOR wire obfuscation.
2. `POST /api/vi/security/checkAuth/permission/{role}`: Server-side RBAC session authorization.
3. `POST /api/vi/agreement/create`: Generates digital contract with SHA-256 content hash.
4. `POST /api/vi/agreement/{id}/sign/{role}`: Digital signature recording timestamp, IP, and User-Agent.
5. `POST /api/vi/payments/create-order`: Generates Razorpay Order ID for escrow pre-funding.
6. `POST /api/vi/payments/verify-payment`: Validates HMAC-SHA256 signature to transition Escrow to `funded`.
7. `POST /api/vi/milestone/{id}/confirm`: Approves deliverable, computes early bonus or late penalty, releases escrow.
8. `POST /api/vi/dispute/file`: Files formal dispute; generates Razorpay arbitration fee payment link.
9. `POST /predict/success` (:5001): Predicts project completion probability (0–100%) via 8 scaled features.
10. `POST /api/ml/match` (:5001): Executes TLAM multiplicative scoring across candidate pool.

---

## 5. Core Formulas & Mathematical Models

### 1. TLAM Multiplicative Matching Formula
$$\text{Score} = \text{Capability}^{0.349} \times \text{Reliability}^{0.148} \times \text{Alignment}^{0.034}$$
- **Capability ($C$)**: $0.45 \times \text{skill} + 0.35 \times \text{semantic} + 0.20 \times \text{context}$
- **Reliability ($R$)**: $0.65 \times \text{maturity} + 0.35 \times \text{behavior}$
- **Alignment ($A$)**: Gaussian budget proximity
- **Key Interview Takeaway**: Multiplicative scoring acts as a strict logical AND. If a candidate lacks core skills ($C \approx 0$), the entire score drops to 0, completely preventing unqualified spam bidders from ranking.

### 2. Milestone SLA Bonus / Penalty Formula
- Early Delivery: $\text{Bonus} = \text{Amount} \times (\text{bonusPercent} / 100) \times \text{Days Early}$
- Late Delivery: $\text{Penalty} = \text{Amount} \times (\text{penaltyPercent} / 100) \times \text{Days Late}$
- Net Payout: $\text{Final Amount} = \max(0, \text{Amount} \pm \text{Adjustment})$

---

## 6. The 3 Critical Codebase Bugs to Discuss Confidently

1. **Load Balancer Connection Leak**:
   - *Where*: [load-balancer/loadBalancer.js](file:///e:/DEVOPS/load-balancer/loadBalancer.js#L140)
   - *Issue*: `assignedServer.connections++` increments on each request but is only decremented inside the error handler of `proxy.web()`. Connections accumulate indefinitely.
   - *Fix*: Attach `res.on('finish', () => { assignedServer.connections--; })`.
2. **Static IV in AES-256-CBC Cookie**:
   - *Where*: [load-balancer/loadBalancer.js](file:///e:/DEVOPS/load-balancer/loadBalancer.js#L47)
   - *Issue*: `const IV = crypto.randomBytes(16);` is evaluated once at module boot. Reusing static IV in CBC mode violates semantic security.
   - *Fix*: Generate a unique random 16-byte IV per encryption and prepend it to the ciphertext.
3. **Hardcoded Fallback Key in Redux Store**:
   - *Where*: [frontend/src/redux/store.ts](file:///e:/DEVOPS/frontend/src/redux/store.ts#L8)
   - *Issue*: `const ENCRYPTION_KEY = "fallback-key";` is hardcoded instead of reading `import.meta.env.VITE_ENCRYPTION_KEY`.

---

## 7. 5 Key Trade-Offs & Why

1. **Multiplicative TLAM vs Linear Sum**: Multiplicative eliminates spam bidders with 0 skills; linear gives false high scores based on ratings alone.
2. **MongoDB vs PostgreSQL**: MongoDB stores polymorphic contracts, milestone deliverables, and dispute dossiers as single documents without multi-table joins.
3. **FastAPI Python Microservice vs In-Process Node.js**: Isolates CPU-heavy linear algebra from the event-driven Node.js I/O loop.
4. **WebSockets vs HTTP Polling**: Sub-30ms bidirectional chat with minimal packet header overhead vs repeated HTTP connection handshakes.
5. **Arbitration Fee Gate vs Free Disputes**: Requiring a non-refundable Razorpay fee before dispute activation stops frivolous claim spam.

---

## 8. Top 5 Mental Models & Catchphrases for the Interview

1. *"We don't use AI as a black box; our TLAM engine provides full explainability by exposing sub-feature weights and mathematical formulas directly in the UI."*
2. *"Financial integrity is enforced through pre-funded Escrow and cryptographic state machines, not verbal promises."*
3. *"We enforce a two-tier route defense: client-side Redux for zero-flicker routing, backed by server-side RBAC session validation."*
4. *"In technical talent matching, core skills are non-negotiable constraints. That is why our algorithm uses a multiplicative Cobb-Douglas formula rather than a compensatory linear sum."*
5. *"We practice defense-in-depth: while we have client-side regex filters to prevent disintermediation, all policy infractions are logged on the server with strike increments and automated wallet freezing."*
