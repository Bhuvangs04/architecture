# AiTwin Project Documentation Index

Welcome to the definitive architecture and reverse engineering documentation for **AiTwin** (Autonomous Digital Twin Social & Matchmaking Platform).

This documentation suite is organized into 14 comprehensive guides designed for software architects, backend engineers, system evaluators, and technical interview candidates.

---

## Documentation Navigation Table

| Document | Title | Purpose & Key Topics |
| :--- | :--- | :--- |
| [00_PROJECT_OVERVIEW.md](./00_PROJECT_OVERVIEW.md) | **Project Overview** | Executive summary, problem statement, business value, core capabilities, and repository structure. |
| [01_PROJECT_ARCHITECTURE.md](./01_PROJECT_ARCHITECTURE.md) | **Project Architecture** | Hybrid architecture pattern, high-level Mermaid diagrams, component responsibilities, state machines. |
| [02_TECH_STACK_DEEP_DIVE.md](./02_TECH_STACK_DEEP_DIVE.md) | **Tech Stack Deep Dive** | Exhaustive analysis of all 22+ libraries, frameworks, tools, trade-offs, and comparison matrix. |
| [03_END_TO_END_EXECUTION_FLOW.md](./03_END_TO_END_EXECUTION_FLOW.md) | **End-to-End Execution Flow** | Application startup sequence, WebSocket handshake, and step-by-step flows for all 4 chat modes. |
| [04_API_DOCUMENTATION.md](./04_API_DOCUMENTATION.md) | **API Documentation** | Complete REST and WebSocket endpoint specifications, request/response JSON payloads, and error codes. |
| [05_DATABASE_DESIGN.md](./05_DATABASE_DESIGN.md) | **Database Design** | MongoDB document schemas, compound indexes, ChromaDB vector topology, and distributed mutex lock. |
| [06_TRADEOFFS_AND_DESIGN_DECISIONS.md](./06_TRADEOFFS_AND_DESIGN_DECISIONS.md) | **Trade-offs & Design Decisions** | In-depth trade-off analyses (MongoDB vs Postgres, vLLM vs OpenAI, Chroma vs Pinecone, Mutex vs Redlock). |
| [07_CODEBASE_DEEP_DIVE.md](./07_CODEBASE_DEEP_DIVE.md) | **Codebase Deep Dive** | File-by-file inspection of all major source modules, critical classes, and complex algorithms. |
| [08_DEPLOYMENT_AND_DEVOPS.md](./08_DEPLOYMENT_AND_DEVOPS.md) | **Deployment & DevOps** | System requirements, environment variables, production Docker Compose, health probes, and CI/CD. |
| [09_SECURITY_PERFORMANCE_AND_SCALABILITY.md](./09_SECURITY_PERFORMANCE_AND_SCALABILITY.md) | **Security & Scalability** | Threat modeling, DoS prevention, AI memory poisoning shield, bottlenecks, and scaling roadmap to 1M users. |
| [10_INTERVIEW_PREPARATION.md](./10_INTERVIEW_PREPARATION.md) | **Interview Preparation Guide** | 30s/1m/3m pitches, 60+ technical questions & answers, cross-question preparation, and debugging stories. |
| [11_QUICK_REVISION_NOTES.md](./11_QUICK_REVISION_NOTES.md) | **Quick Revision Notes** | One-page cheat sheet, architectural state machine summary, mathematical formulas, and top 10 talking points. |
| [12_SETUP_AND_RUN_GUIDE.md](./12_SETUP_AND_RUN_GUIDE.md) | **Setup & Run Guide** | Step-by-step local execution instructions (MongoDB, vLLM, MLflow, FastAPI, testing, troubleshooting). |
| [13_VERIFICATION_AND_EVIDENCE.md](./13_VERIFICATION_AND_EVIDENCE.md) | **Verification & Evidence** | Source file line counts, verified facts vs inferred rationale, unverified components, and action items. |

---

## Suggested Reading Paths by Role

- **For Engineering Managers & System Architects:**
  Start with [00_PROJECT_OVERVIEW.md](./00_PROJECT_OVERVIEW.md) $\rightarrow$ [01_PROJECT_ARCHITECTURE.md](./01_PROJECT_ARCHITECTURE.md) $\rightarrow$ [06_TRADEOFFS_AND_DESIGN_DECISIONS.md](./06_TRADEOFFS_AND_DESIGN_DECISIONS.md) $\rightarrow$ [09_SECURITY_PERFORMANCE_AND_SCALABILITY.md](./09_SECURITY_PERFORMANCE_AND_SCALABILITY.md).
- **For Backend & AI Engineers:**
  Start with [01_PROJECT_ARCHITECTURE.md](./01_PROJECT_ARCHITECTURE.md) $\rightarrow$ [03_END_TO_END_EXECUTION_FLOW.md](./03_END_TO_END_EXECUTION_FLOW.md) $\rightarrow$ [05_DATABASE_DESIGN.md](./05_DATABASE_DESIGN.md) $\rightarrow$ [07_CODEBASE_DEEP_DIVE.md](./07_CODEBASE_DEEP_DIVE.md).
- **For Technical Interview Preparation:**
  Start with [11_QUICK_REVISION_NOTES.md](./11_QUICK_REVISION_NOTES.md) $\rightarrow$ [10_INTERVIEW_PREPARATION.md](./10_INTERVIEW_PREPARATION.md) $\rightarrow$ [06_TRADEOFFS_AND_DESIGN_DECISIONS.md](./06_TRADEOFFS_AND_DESIGN_DECISIONS.md).
- **For DevOps & Infrastructure Engineers:**
  Start with [08_DEPLOYMENT_AND_DEVOPS.md](./08_DEPLOYMENT_AND_DEVOPS.md) $\rightarrow$ [12_SETUP_AND_RUN_GUIDE.md](./12_SETUP_AND_RUN_GUIDE.md) $\rightarrow$ [09_SECURITY_PERFORMANCE_AND_SCALABILITY.md](./09_SECURITY_PERFORMANCE_AND_SCALABILITY.md).

