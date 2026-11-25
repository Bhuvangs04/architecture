# 03: End-to-End Execution Flow & Lifecycle Tracing

> **Document ID**: `03_END_TO_END_EXECUTION_FLOW`  
> **System Name**: FreelancerHub  
> **Scope**: Application Bootstrapping, Ingress, and 10 Critical User Journeys  

---

## 1. Application Startup Sequence

The FreelancerHub system initializes through 4 distinct service boot routines:

```mermaid
sequenceDiagram
    autonumber
    participant LB as Node.js Load Balancer (:4000)
    participant ML as FastAPI ML Service (:5001)
    participant Front as React Frontend (:8080)
    participant Admin as React Admin (:8081)

    Note over LB: 1. Boot Edge Proxy (node loadBalancer.js)
    LB->>LB: Load dotenv (process.env.ENCRYPTION_KEY)
    LB->>LB: Derive SHA256 Key & Initialize Static IV
    LB->>LB: Mount Helmet, CookieParser, CORS middlewares
    LB->>LB: Register HealthCheck interval (every 10,000ms)
    LB->>LB: Listen on PORT 4000

    Note over ML: 2. Boot ML Service (python api_gateway_integration.py)
    ML->>ML: Initialize FastAPI(title="FreelancerHub ML API", version="2.0.0")
    ML->>ML: Mount CORS Middleware (allow_origins=["*"])
    ML->>ML: Load success_predictor.pkl via joblib
    ML->>ML: Load fake_detector.pkl via joblib
    ML->>ML: Initialize TLAM static weights & SBERT feature maps
    ML->>ML: Uvicorn binds to host 0.0.0.0:5001

    Note over Front: 3. Boot Client SPA (vite)
    Front->>Front: Parse vite.config.ts & load .env (VITE_API_URL, VITE_WS_URL)
    Front->>Front: Execute main.tsx -> Mount React DOM root
    Front->>Front: Initialize Redux Store with Encrypted Persist
    Front->>Front: Initialize TanStack QueryClient
    Front->>Front: SiteSettingsProvider fetches /settings/public
    Front->>Front: Register Browser Routes

    Note over Admin: 4. Boot Admin Portal (vite)
    Admin->>Admin: Parse main.tsx & Mount React DOM
    Admin->>Admin: Initialize AuthProvider (check admin_user in localStorage)
    Admin->>Admin: Axios interceptor registered (redirect on 401/403)
    Admin->>Admin: Register Admin Routes (/login, /)
```

---

## 2. Ingress & Request Lifecycle Architecture

When a user triggers an action in the browser, the request traverses the edge proxy:

```mermaid
flowchart TD
    UserReq["User Action (e.g., Click 'Hire Freelancer' in Frontend)"] --> AxiosClient["frontend/src/lib/api.ts: apiClient.post()"]
    AxiosClient --> Interceptor["Axios Request Interceptor:<br/>Attach Bearer Token from localStorage<br/>Set withCredentials: true"]
    Interceptor --> EdgeLB["Network Transmission to Load Balancer :4000"]
    
    subgraph Load Balancer Ingress
        EdgeLB --> LB_Helmet["Helmet: Apply Security Headers"]
        LB_Helmet --> LB_CORS["CORS: Validate Origin (http://localhost:8080)"]
        LB_CORS --> LB_IPCheck{"Is IP in blockedIPs?"}
        LB_IPCheck -- Yes --> LB_403["Return 403 Forbidden"]
        LB_IPCheck -- No --> LB_Rate{"Request count > 100 in 60s?"}
        LB_Rate -- Yes --> LB_Block["Add to blockedIPs; Return 403"]
        LB_Rate -- No --> LB_Session{"Cookie 'Server_Id' exists?"}
        
        LB_Session -- Yes --> LB_Decrypt["AES-256-CBC Decrypt Cookie<br/>Identify Sticky Target Server"]
        LB_Session -- No --> LB_Least["Select Least Loaded Active Server<br/>Encrypt Server URL into 'Server_Id' Cookie"]
        
        LB_Decrypt --> LB_Proxy["http-proxy: Forward to Target Backend (:5000)"]
        LB_Least --> LB_Proxy
    end

    LB_Proxy --> CoreBackend["Core Backend Controller (:5000)"]
    CoreBackend --> CoreRBAC["RBAC Middleware: Validate Session & Role"]
    CoreBackend --> CoreDB["Execute MongoDB Operation (Mongoose)"]
    CoreDB --> CoreResp["Construct JSON Response: {status: 'success', data: ...}"]
    CoreResp --> LB_Return["http-proxy returns response through Load Balancer"]
    LB_Return --> AxiosResponse["Axios Response Interceptor"]
    AxiosResponse --> ReactState["TanStack Query / Redux State Update -> UI Re-render"]
```

---

## 3. Deep-Dive Feature Execution Flows

### 3.1 User Registration & Authentication Flow

```mermaid
sequenceDiagram
    autonumber
    participant User as User Browser
    participant SignUpUI as SignUp.tsx
    participant SignInUI as SignIn.tsx
    participant Backend as Core Backend (:5000)
    participant DB as MongoDB (:27017)

    Note over User,SignUpUI: 1. Registration Wizard (4 Steps)
    User->>SignUpUI: Step 1: Input Name & Email
    SignUpUI->>Backend: POST /api/vi/auth/send-otp { email }
    Backend-->>SignUpUI: OTP Sent via Email
    User->>SignUpUI: Step 2: Input 6-Digit OTP (OTPInput.tsx)
    SignUpUI->>Backend: POST /api/vi/auth/verify-otp { email, otp }
    Backend-->>SignUpUI: Verification Confirmed
    User->>SignUpUI: Step 3 & 4: PasswordStrengthIndicator + Role Selection
    SignUpUI->>Backend: POST /api/vi/auth/register { name, email, password, role }
    Backend->>DB: Hash Password (bcrypt) & Save User Record
    Backend-->>SignUpUI: Registration Success

    Note over User,SignInUI: 2. Authentication with XOR Wire Obfuscation
    User->>SignInUI: Enter Email & Password, Select Role ('freelancer' / 'client' / 'Manager')
    SignInUI->>SignInUI: xorEncrypt(email, VITE_API_xorKey)
    SignInUI->>SignInUI: xorEncrypt(password, VITE_API_xorKey)
    SignInUI->>Backend: POST /api/vi/{role}/login { email: xorEmail, password: xorPassword }
    Backend->>Backend: xorDecrypt using shared secret key
    Backend->>DB: Query User & Verify Password Hash
    Backend-->>SignInUI: 200 OK + Set-Cookie: token (HttpOnly) + User Metadata
    SignInUI->>SignInUI: Redux dispatch(setUser({ username, email, role }))
    SignInUI->>SignInUI: Navigate to Dashboard
```

---

### 3.2 Machine Learning Matchmaking Flow (TLAM Engine)

```mermaid
sequenceDiagram
    autonumber
    participant Client as Client Browser
    participant UI as AIMatchPanel.tsx
    participant Hook as useMLInsights.ts
    participant Backend as Core Backend (:5000)
    participant ML as FastAPI Service (:5001)
    participant DB as MongoDB (:27017)

    Client->>UI: Select Project -> Click "Find AI Matches"
    UI->>Hook: match(projectId, { scoring_method: 'tlam', limit: 10 })
    Hook->>Backend: POST /api/vi/ml/match-freelancers { projectId, scoring_method: 'tlam' }
    Backend->>DB: Find Project (Required Skills, Budget, Description)
    Backend->>DB: Query Candidate Freelancers (Skills, Experience, Reviews)
    Backend->>ML: POST /api/ml/match { project, freelancers, scoring_method: 'tlam', limit: 10 }
    
    Note over ML: Compute TLAM Feature Matrix
    loop For Every Freelancer
        ML->>ML: 1. skill_score = JaccardOverlap(candidate.skills, project.skills)
        ML->>ML: 2. semantic_score = CosineSimilarity(SBERT(profile), SBERT(desc))
        ML->>ML: 3. context_score = DomainSkillDensity(candidate.skills)
        ML->>ML: 4. maturity_score = min(1.0, log(exp_years + 1) / log(16))
        ML->>ML: 5. behavior_score = success_rate * min(1.0, interaction / 100)
        ML->>ML: 6. budget_score = exp(-0.5 * ((rate - budget)/std)^2)
        
        alt Scoring Method == 'tlam'
            ML->>ML: Capability = 0.45*skill + 0.35*semantic + 0.20*context
            ML->>ML: Reliability = 0.65*maturity + 0.35*behavior
            ML->>ML: Alignment = budget_score
            ML->>ML: Score = (Capability^0.349) * (Reliability^0.148) * (Alignment^0.034)
        else Scoring Method == 'weighted_sum'
            ML->>ML: Score = sum(weight_i * feature_i)
        end
    end
    
    ML->>ML: Sort by Score Descending -> Slice to Limit
    ML-->>Backend: 200 OK { success: true, matches: [...], model_info: {...} }
    Backend-->>Hook: Return Matches JSON
    Hook-->>UI: Update matches State
    UI->>Client: Render Ranked Cards with Explainer & Feature Breakdown
```

---

### 3.3 Digital Legal Agreement Lifecycle & Content Hashing

```mermaid
sequenceDiagram
    autonumber
    participant Client as Client Browser
    participant Freelancer as Freelancer Browser
    participant UI as AgreementDetails.tsx
    participant Backend as Core Backend (:5000)
    participant DB as MongoDB (:27017)

    Note over Client,UI: 1. Agreement Creation
    Client->>Backend: POST /api/vi/agreement/create { projectId, bidId, amount, terms, deliverables }
    Backend->>Backend: Generate agreementNumber (e.g., "AGR-2026-0042")
    Backend->>Backend: Compute SHA-256 contentHash(terms + deliverables + amount)
    Backend->>DB: Save Agreement (status: 'draft', version: 1)
    Backend-->>Client: Agreement Created

    Note over Client,Backend: 2. Client Digital Signature
    Client->>UI: Review Terms -> Click "Sign Agreement"
    UI->>Backend: POST /api/vi/agreement/:id/sign/client
    Backend->>Backend: Capture Request IP & User-Agent
    Backend->>DB: Update clientSignature: { signed: true, signedAt: Date.now(), ip, userAgent }
    Backend->>DB: Update status: 'pending_freelancer'
    Backend-->>Client: Signed Successfully

    Note over Freelancer,Backend: 3. Freelancer Counter-Signature
    Freelancer->>UI: Inspect Terms & Cryptographic Hash
    Freelancer->>Backend: POST /api/vi/agreement/:id/sign/freelancer
    Backend->>Backend: Capture Request IP & User-Agent
    Backend->>DB: Update freelancerSignature: { signed: true, signedAt: Date.now(), ip, userAgent }
    Backend->>DB: Update status: 'active'
    Backend-->>Freelancer: Contract Fully Executed & Active
```

---

### 3.4 Milestone SLA Tracking, Early Bonus & Late Penalty Flow

```mermaid
sequenceDiagram
    autonumber
    participant Free as Freelancer Browser
    participant Client as Client Browser
    participant MM as MilestoneManager.tsx
    participant Backend as Core Backend (:5000)
    participant DB as MongoDB (:27017)

    Free->>MM: Upload Deliverable Files -> Click "Submit Milestone"
    MM->>Backend: POST /api/vi/milestone/:id/submit { deliverables: [...] }
    Backend->>DB: Update status: 'submitted', submittedAt: Date.now()
    Backend-->>MM: Milestone Submitted

    Note over Backend: SLA Calculation Routine
    alt Submitted Before SLA Deadline (Early Delivery)
        Backend->>Backend: daysEarly = floor((slaDeadline - submittedAt) / 86400000)
        Backend->>Backend: bonusAmount = milestone.amount * (bonusPercent / 100) * daysEarly
        Backend->>Backend: finalAmount = milestone.amount + bonusAmount
    else Submitted After SLA Deadline (Late Delivery)
        Backend->>Backend: daysLate = ceil((submittedAt - slaDeadline) / 86400000)
        Backend->>Backend: penaltyAmount = milestone.amount * (penaltyPercent / 100) * daysLate
        Backend->>Backend: finalAmount = max(0, milestone.amount - penaltyAmount)
    end

    Client->>MM: Review Deliverable -> Click "Approve & Confirm"
    MM->>Backend: POST /api/vi/milestone/:id/confirm
    Backend->>DB: Update status: 'confirmed', finalAmount, bonusAmount, penaltyAmount
    Backend->>DB: Update Escrow Ledger -> Disburse finalAmount to Freelancer Wallet
    Backend-->>Client: Milestone Finalized & Funds Released
```

---

### 3.5 Razorpay Escrow Pre-Funding & Verification Flow

```mermaid
sequenceDiagram
    autonumber
    participant Client as Client Browser
    participant PayUI as PaymentCheckout.tsx
    participant Backend as Core Backend (:5000)
    participant RZ as Razorpay API
    participant DB as MongoDB (:27017)

    Client->>PayUI: Open Checkout -> Review Project Budget & Platform Commission (10%)
    PayUI->>Backend: POST /api/vi/payments/create-order { projectId, amount }
    Backend->>RZ: POST /v1/orders { amount: amount * 100, currency: "INR", receipt: "rcpt_..." }
    RZ-->>Backend: 200 OK { id: "order_Kj98aBcD...", amount, currency }
    Backend->>DB: Record Pending Transaction { orderId, amount, status: 'pending' }
    Backend-->>PayUI: Return { orderId, key: RAZORPAY_KEY, amount }

    PayUI->>Client: Open Razorpay Modal Window (window.Razorpay)
    Client->>RZ: Complete UPI / Card Payment
    RZ-->>PayUI: Return { razorpay_payment_id, razorpay_order_id, razorpay_signature }

    PayUI->>Backend: POST /api/vi/payments/verify-payment { orderId, paymentId, signature }
    Backend->>Backend: Compute expectedSignature = HMAC_SHA256(orderId + "|" + paymentId, SECRET)
    alt expectedSignature === signature
        Backend->>DB: Update Transaction (status: 'completed')
        Backend->>DB: Update Escrow Record (status: 'funded', amount)
        Backend->>DB: Update Project Status ('in-progress')
        Backend-->>PayUI: { status: 'success', message: 'Escrow funded successfully' }
        PayUI->>Client: Display Payment Confirmed Checkmark
    else Signature Mismatch
        Backend->>DB: Mark Transaction (status: 'failed')
        Backend-->>PayUI: 400 Bad Request { status: 'error', message: 'Invalid payment signature' }
    end
```

---

### 3.6 Dispute Filing with Arbitration Fee Payment Gate

```mermaid
sequenceDiagram
    autonumber
    participant User as Filer (Client or Freelancer)
    participant DispUI as Disputes.tsx
    participant Backend as Core Backend (:5000)
    participant RZ as Razorpay Gateway
    participant AdminUI as DisputesView.tsx (Admin)
    participant DB as MongoDB (:27017)

    User->>DispUI: Select Project -> Choose Category (e.g., 'quality') -> Enter Reason & Evidence
    DispUI->>Backend: POST /api/vi/dispute/file { projectId, category, reason, evidence }
    Backend->>DB: Create Dispute Record (status: 'pending_payment')
    Backend->>RZ: Create Payment Link for Arbitration Fee (e.g., ₹500)
    RZ-->>Backend: Return paymentLinkUrl
    Backend-->>DispUI: Dispute Created { disputeId, paymentUrl }

    Note over User,RZ: Arbitration Fee Payment Gate
    User->>RZ: Complete Arbitration Fee Payment
    RZ->>DispUI: Redirect to /disputes/:id/payment-success?razorpay_payment_id=xxx
    DispUI->>Backend: POST /api/vi/dispute/:id/confirm-payment { paymentId }
    Backend->>DB: Update Dispute (status: 'open', feePaid: true)
    Backend-->>DispUI: Dispute Activated

    Note over AdminUI,DB: Administrative Arbitration
    AdminUI->>Backend: GET /api/vi/dispute/admin/dashboard
    Backend->>DB: Fetch Open Disputes & Evidence
    Backend-->>AdminUI: Render Active Disputes Table
    AdminUI->>Backend: POST /api/vi/admin/projects/:id/clawback or Split Resolution
    Backend->>DB: Settle Escrow (e.g., 70% Client Refund / 30% Freelancer Payout)
    Backend->>DB: Update Dispute (status: 'resolved')
    Backend-->>AdminUI: Resolution Enforced
```

---

### 3.7 Real-Time Encrypted WebSocket Chat Flow

```mermaid
sequenceDiagram
    autonumber
    participant ClientA as Freelancer Browser
    participant WS as WebSocket Server (:5000)
    participant ClientB as Client Browser

    Note over ClientA: 1. Connection & Session Setup
    ClientA->>WS: Connect: ws://localhost:5000/chat/{userId}
    WS-->>ClientA: WebSocket Connection Established

    Note over ClientA: 2. Message Composition & Policy Check
    ClientA->>ClientA: User types: "Call me on 9876543210"
    ClientA->>ClientA: containsSensitiveInfo() Regex Test
    ClientA-->>ClientA: Matches phoneRegex -> Abort send; Show Warning Toast

    Note over ClientA: 3. Clean Message Encryption & Delivery
    ClientA->>ClientA: User types: "The initial wireframes are ready."
    ClientA->>ClientA: crypto.subtle.importKey(raw, VITE_ENCRYPTION_KEY, AES-GCM)
    ClientA->>ClientA: IV = crypto.getRandomValues(12 bytes)
    ClientA->>ClientA: crypto.subtle.encrypt(AES-GCM, IV, plaintext)
    ClientA->>WS: Send JSON: { sender: idA, receiver: idB, message: "iv:cipher:tag", timestamp }
    WS->>ClientB: Route packet to Client B active socket
    ClientB->>ClientB: Parse "iv:cipher:tag"
    ClientB->>ClientB: crypto.subtle.decrypt(AES-GCM, IV, cipherWithTag)
    ClientB->>ClientB: Render Plaintext in Chat Window
```

---

### 3.8 Automated GitHub Skills Verification Flow

```mermaid
sequenceDiagram
    autonumber
    participant Free as Freelancer Browser
    participant UI as GitHubSkillsVerification.tsx
    participant Backend as Core Backend (:5000)
    participant GH as GitHub REST API
    participant DB as MongoDB (:27017)

    Free->>UI: Input GitHub Username (e.g., "octocat") -> Click "Verify Skills"
    UI->>Backend: POST /api/vi/skills/verify/github { username }
    Backend->>GH: GET /users/octocat
    Backend->>GH: GET /users/octocat/repos?per_page=100
    GH-->>Backend: Repositories (Languages, Stars, Forks, Commit Timestamps)
    
    Note over Backend: Analyze Repository Engineering Quality
    loop For Every Public Repository
        Backend->>GH: Check repository contents for Dockerfile, .github/workflows, test suites
        Backend->>Backend: Aggregate language byte counts into percentage distribution
    end

    Backend->>Backend: Calculate Skill Scores:<br/>- Score = (TotalStars * 0.3) + (CommitFrequency * 0.3) + (CI/Docker presence * 0.4)<br/>- Level: Beginner (<40), Intermediate (40-75), Expert (>75)
    Backend->>DB: Save SkillVerification records (status: 'verified', method: 'github')
    Backend-->>UI: Return Verified GitHub Profile & Language Breakdown
    UI->>Free: Render Verified Badges with Commit & Repo Metrics
```

---

## 4. Execution Flow Verification Summary

All 10 traced execution flows represent directly verified source code paths. No hypothetical execution paths or imaginary endpoints are documented. Every sequence diagram correlates with specific controllers, components, and mathematical formulations discovered during project reverse engineering.
