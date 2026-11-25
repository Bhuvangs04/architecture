# 04: Complete API Reference & Specification

> **Document ID**: `04_API_DOCUMENTATION`  
> **System Name**: FreelancerHub  
> **Scope**: Complete Specification of Discovered REST Endpoints across Client, Admin, and ML Services  
> **Standard Base URLs**:
> - Client Backend: `http://localhost:5000/api/vi`
> - Admin Control Plane: `http://localhost:5000`
> - ML Microservice: `http://localhost:5001`
> - Load Balancer Ingress: `http://localhost:4000`

---

## 1. Authentication & Session APIs

### 1.1 User Login (Role-Specific with XOR Wire Obfuscation)
- **Method**: `POST`
- **URL**: `/api/vi/{role}/login` (where `{role}` is `freelancer`, `client`, or `Manager`)
- **Purpose**: Authenticates credentials for the specified portal role.
- **Headers**: `Content-Type: application/json`
- **Request Body**:
```json
{
  "email": "EncodedBase64XORString==",
  "password": "EncodedBase64XORString==",
  "secretCode": "OPTIONAL_FOR_MANAGER_ROLE"
}
```
- **Response Structure (200 OK)**:
```json
{
  "status": "success",
  "token": "jwt_token_string",
  "username": "johndoe",
  "email": "john@example.com",
  "role": "freelancer",
  "requiresPasswordChange": false,
  "isBanned": false,
  "profileComplete": true
}
```
- **Error Responses**:
  - `401 Unauthorized`: Invalid credentials.
  - `403 Forbidden`: Account suspended or banned (`SuspendSection` triggered).

---

### 1.2 Check Server-Side RBAC Permission
- **Method**: `POST`
- **URL**: `/api/vi/security/checkAuth/permission/{role}`
- **Purpose**: Server-side RBAC validation verifying whether the session cookie belongs to `{role}`.
- **Headers**: `withCredentials: true` (HttpOnly session cookie)
- **Response Structure (200 OK)**:
```json
{
  "status": "success",
  "message": "Authorized access for role freelancer",
  "role": "freelancer",
  "userId": "usr_6789abc"
}
```
- **Error Response (403 Forbidden)**:
```json
{
  "status": "error",
  "message": "Forbidden: User does not hold required role permission"
}
```

---

### 1.3 Check Profile Setup Status
- **Method**: `GET`
- **URL**: `/api/vi/security/checkAuth/profile-status`
- **Purpose**: Checks whether user has satisfied mandatory onboarding (skills, bio, rate).
- **Response Structure (200 OK)**:
```json
{
  "profileComplete": true,
  "missingFields": []
}
```

---

### 1.4 User Logout
- **Method**: `GET`
- **URL**: `/api/vi/logout`
- **Purpose**: Clears HttpOnly session cookies on the server and terminates the session.
- **Response Structure (200 OK)**:
```json
{
  "status": "success",
  "message": "Logged out successfully"
}
```

---

## 2. Platform Settings APIs

### 2.1 Get Public Platform Settings
- **Method**: `GET`
- **URL**: `/api/vi/settings/public`
- **Purpose**: Fetches platform fee percentages, maintenance status, and budget limits.
- **Response Structure (200 OK)**:
```json
{
  "platformCommissionPercent": 10,
  "siteName": "FreelancerHub",
  "logoUrl": "https://freelancerhub.com/logo.png",
  "supportEmail": "support@freelancerhub.com",
  "maintenanceMode": false,
  "maintenanceMessage": "",
  "minimumProjectBudget": 500,
  "maximumProjectBudget": 50000
}
```

---

## 3. Project & Work Submission APIs

### 3.1 Fetch Ongoing Projects
- **Method**: `GET`
- **URL**: `/api/vi/worksubmission/ongoing/projects/V1`
- **Purpose**: Returns all ongoing projects for the authenticated freelancer or client.
- **Response Structure (200 OK)**:
```json
[
  {
    "_id": "prj_98234ab",
    "title": "FinTech Mobile App UI/UX",
    "description": "Design 25 screens for a mobile banking platform.",
    "clientId": "usr_client123",
    "clientName": "Acme Corp",
    "freelancerId": "usr_free456",
    "status": "in-progress",
    "budget": 25000,
    "tasks": [
      { "_id": "tsk_01", "title": "Wireframes", "completed": true, "createdAt": "2026-06-01T10:00:00Z" }
    ],
    "files": [],
    "createdAt": "2026-06-01T09:00:00Z"
  }
]
```

---

### 3.2 Add Task to Project
- **Method**: `POST`
- **URL**: `/api/vi/worksubmission/tasks`
- **Request Body**:
```json
{
  "projectId": "prj_98234ab",
  "title": "Design Settings Screen"
}
```
- **Response Structure (200 OK)**: Returns updated Project document.

---

### 3.3 Update Task Completion State
- **Method**: `PATCH`
- **URL**: `/api/vi/worksubmission/tasks/{taskId}`
- **Request Body**:
```json
{
  "projectId": "prj_98234ab",
  "completed": true
}
```

---

### 3.4 Upload Work Submission File Artifact
- **Method**: `POST`
- **URL**: `/api/vi/worksubmission/upload-file`
- **Headers**: `Content-Type: multipart/form-data`
- **Payload**: Form data with `file` binary and `projectId`
- **Response Structure (200 OK)**:
```json
{
  "message": "File uploaded successfully",
  "fileUrl": "https://storage.freelancerhub.com/artifacts/mockup_v1.pdf"
}
```

---

## 4. Legal Agreement APIs

### 4.1 Create Draft Agreement
- **Method**: `POST`
- **URL**: `/api/vi/agreement/create`
- **Request Body**:
```json
{
  "projectId": "prj_98234ab",
  "bidId": "bid_7718aa",
  "agreedAmount": 25000,
  "platformFee": 2500,
  "totalAmount": 27500,
  "deadline": "2026-07-15T00:00:00Z",
  "deliverables": "25 Figma screens, design tokens, export assets",
  "terms": "Work must adhere to WCAG 2.1 AA accessibility guidelines."
}
```
- **Response Structure (200 OK)**:
```json
{
  "status": "success",
  "data": {
    "_id": "agr_110293",
    "agreementNumber": "AGR-2026-0042",
    "status": "draft",
    "version": 1,
    "contentHash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "clientSignature": { "signed": false },
    "freelancerSignature": { "signed": false }
  }
}
```

---

### 4.2 Sign Agreement (Client)
- **Method**: `POST`
- **URL**: `/api/vi/agreement/{id}/sign/client`
- **Purpose**: Attaches client digital signature with IP and User-Agent capture.
- **Response Structure (200 OK)**:
```json
{
  "status": "success",
  "data": {
    "_id": "agr_110293",
    "status": "pending_freelancer",
    "clientSignature": {
      "signed": true,
      "signedAt": "2026-06-02T14:22:10Z",
      "ipAddress": "192.168.1.1",
      "userAgent": "Mozilla/5.0 Chrome/125.0"
    }
  }
}
```

---

### 4.3 Sign Agreement (Freelancer)
- **Method**: `POST`
- **URL**: `/api/vi/agreement/{id}/sign/freelancer`
- **Purpose**: Countersigns contract; transitions agreement state to `active`.

---

### 4.4 Amend Agreement
- **Method**: `POST`
- **URL**: `/api/vi/agreement/{id}/amend`
- **Request Body**:
```json
{
  "newAmount": 30000,
  "newDeadline": "2026-07-30T00:00:00Z",
  "amendmentReason": "Client requested 5 additional dashboard screens"
}
```
- **Response Structure (200 OK)**: Increments `version` to 2; sets `status` to `amended`; clears previous signatures to enforce mutual re-signing.

---

## 5. Milestone Management APIs

### 5.1 Create Milestone
- **Method**: `POST`
- **URL**: `/api/vi/milestone/create`
- **Request Body**:
```json
{
  "agreementId": "agr_110293",
  "projectId": "prj_98234ab",
  "title": "Phase 1: High-Fidelity Prototypes",
  "description": "Figma interactive prototype for user testing",
  "amount": 10000,
  "dueDate": "2026-06-20T00:00:00Z",
  "slaDeadline": "2026-06-18T00:00:00Z",
  "bonusPercent": 2,
  "penaltyPercent": 3
}
```

---

### 5.2 Submit Milestone Deliverables
- **Method**: `POST`
- **URL**: `/api/vi/milestone/{id}/submit`
- **Request Body**:
```json
{
  "deliverables": [
    { "name": "prototype.fig", "url": "https://storage.freelancerhub.com/prototype.fig" }
  ]
}
```

---

### 5.3 Confirm & Release Milestone
- **Method**: `POST`
- **URL**: `/api/vi/milestone/{id}/confirm`
- **Purpose**: Approves work, computes early delivery bonus or late penalty, releases escrow.
- **Response Structure (200 OK)**:
```json
{
  "status": "success",
  "milestone": {
    "_id": "mls_4410",
    "status": "confirmed",
    "daysEarly": 2,
    "bonusAmount": 400,
    "penaltyAmount": 0,
    "finalAmount": 10400
  }
}
```

---

## 6. Payment & Escrow APIs

### 6.1 Create Razorpay Order
- **Method**: `POST`
- **URL**: `/api/vi/payments/create-order`
- **Request Body**:
```json
{
  "projectId": "prj_98234ab",
  "amount": 27500
}
```
- **Response Structure (200 OK)**:
```json
{
  "id": "order_OD918237123",
  "entity": "order",
  "amount": 2750000,
  "currency": "INR",
  "receipt": "rcpt_prj_98234ab",
  "status": "created",
  "notes": {
    "project_id": "prj_98234ab",
    "client_id": "usr_client123"
  }
}
```

---

### 6.2 Verify Razorpay Signature & Fund Escrow
- **Method**: `POST`
- **URL**: `/api/vi/payments/verify-payment`
- **Request Body**:
```json
{
  "orderId": "order_OD918237123",
  "paymentId": "pay_P982371234",
  "signature": "hmac_sha256_hex_digest..."
}
```

---

## 7. Dispute & Arbitration APIs

### 7.1 File Dispute
- **Method**: `POST`
- **URL**: `/api/vi/dispute/file`
- **Request Body**:
```json
{
  "projectId": "prj_98234ab",
  "milestoneId": "mls_4410",
  "category": "quality",
  "reason": "Deliverable failed basic responsiveness requirements.",
  "evidence": [
    {
      "type": "screenshot",
      "title": "Broken layout on mobile view",
      "url": "https://storage.freelancerhub.com/evidence_01.png"
    }
  ]
}
```
- **Response Structure (200 OK)**:
```json
{
  "disputeId": "dsp_77019",
  "status": "pending_payment",
  "arbitrationFee": 500,
  "paymentUrl": "https://api.razorpay.com/v1/plinks/plink_88921"
}
```

---

### 7.2 Confirm Arbitration Fee Payment
- **Method**: `POST`
- **URL**: `/api/vi/dispute/{id}/confirm-payment`
- **Request Body**:
```json
{
  "paymentId": "pay_ArbFee99182"
}
```
- **Response Structure (200 OK)**: Transitions dispute from `pending_payment` $\to$ `open`.

---

## 8. Skill Verification APIs

### 8.1 Verify via GitHub Analysis
- **Method**: `POST`
- **URL**: `/api/vi/skills/verify/github`
- **Request Body**: `{ "username": "octocat" }`
- **Response Structure (200 OK)**:
```json
{
  "username": "octocat",
  "verifiedSkills": [
    { "skillName": "TypeScript", "proficiency": "expert", "score": 88 },
    { "skillName": "Docker", "proficiency": "intermediate", "score": 65 }
  ],
  "profile": {
    "totalCommits": 1420,
    "topLanguages": [{ "name": "TypeScript", "percentage": 74.2 }]
  }
}
```

---

### 8.2 Submit Skill Challenge Answers
- **Method**: `POST`
- **URL**: `/api/vi/skills/challenges/{skillName}/submit`
- **Request Body**:
```json
{
  "answers": [
    { "questionId": "q_01", "selectedOptionId": "opt_c" },
    { "questionId": "q_02", "selectedOptionId": "opt_a" }
  ]
}
```

---

## 9. Machine Learning Service APIs (:5001)

### 9.1 Health Check
- **Method**: `GET`
- **URL**: `http://localhost:5001/health`
- **Response Structure (200 OK)**:
```json
{
  "status": "healthy",
  "models_loaded": true,
  "message": "ML API is running!",
  "available_models": ["success_prediction", "fake_profile_detection"]
}
```

---

### 9.2 Predict Freelancer Project Success
- **Method**: `POST`
- **URL**: `http://localhost:5001/predict/success`
- **Request Body**:
```json
{
  "experience_years": 5.0,
  "total_projects": 45,
  "avg_rating": 4.7,
  "completion_rate": 95,
  "on_time_delivery_rate": 92,
  "skill_match_score": 85,
  "profile_completeness": 90,
  "budget_ratio": 95
}
```
- **Response Structure (200 OK)**:
```json
{
  "success": true,
  "model": "success_prediction",
  "prediction": {
    "success_probability": 89.4,
    "recommendation": "High chance of success",
    "risk_level": "Low"
  }
}
```

---

### 9.3 Detect Fake Profile Anomaly
- **Method**: `POST`
- **URL**: `http://localhost:5001/predict/fake`
- **Request Body**:
```json
{
  "profile_completeness": 20,
  "total_skills": 35,
  "avg_rating": 5.0,
  "total_reviews": 1,
  "total_projects": 0,
  "account_age_days": 15,
  "portfolio_items": 0,
  "budget_ratio": 25,
  "has_certifications": 0
}
```
- **Response Structure (200 OK)**:
```json
{
  "success": true,
  "model": "fake_profile_detection",
  "prediction": {
    "is_fake": true,
    "fake_probability": 87.2,
    "risk_level": "High",
    "red_flags": [
      "Incomplete profile",
      "Unrealistic number of skills",
      "No portfolio items",
      "High rating with very few reviews",
      "Very new account",
      "Suspiciously low pricing"
    ]
  }
}
```

---

### 9.4 TLAM Match Freelancers (Multiplicative vs Weighted Sum)
- **Method**: `POST`
- **URL**: `http://localhost:5001/api/ml/match`
- **Request Body**:
```json
{
  "project": {
    "id": "prj_100",
    "description": "Build high-throughput event streaming backend using Kafka and Go",
    "required_skills": ["Go", "Kafka", "Docker", "PostgreSQL"],
    "budget": 3000
  },
  "freelancers": [
    {
      "id": "free_99",
      "profile_text": "Senior Go engineer specializing in microservices and distributed queues",
      "skills": ["Go", "Kafka", "Docker"],
      "experience": 7,
      "success_rate": 0.98,
      "interaction": 92,
      "rate": 2800
    }
  ],
  "scoring_method": "tlam",
  "limit": 10
}
```
- **Response Structure (200 OK)**:
```json
{
  "success": true,
  "scoring_method": "tlam",
  "project_id": "prj_100",
  "matches": [
    {
      "freelancer_id": "free_99",
      "rank": 1,
      "primary_score": 0.8412,
      "tlam_score": 0.8412,
      "weighted_score": 0.8750,
      "features": {
        "skill_score": 0.75,
        "semantic_score": 0.88,
        "context_score": 0.85,
        "maturity_score": 0.78,
        "behavior_score": 0.90,
        "budget_score": 0.96
      }
    }
  ],
  "model_info": {
    "name": "SBERT + Logistic Regression + TLAM",
    "description": "Trust-Layer Adaptive Matching multiplicative formula"
  }
}
```
