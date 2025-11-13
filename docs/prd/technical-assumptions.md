# Technical Assumptions

This document captures the technical decisions and assumptions that will guide the architecture and implementation of the Demand Letter Generator. These decisions are based on the project brief, success metrics, and MVP requirements.

## Repository Structure

**Decision:** Monorepo

**Rationale:**
- Simplifies dependency management across frontend and backend
- Enables atomic commits affecting both frontend and backend
- Easier code sharing and refactoring
- Better suited for small to medium team size
- Simplified CI/CD pipeline setup

**Structure:**
```
/packages
  /web-app (React frontend)
  /api (Node.js/Express backend)
  /ai-service (Python service for AI operations)
  /shared (Shared types, utilities, constants)
```

---

## Service Architecture

**Decision:** Microservices within Monorepo using AWS Lambda (Serverless)

**Rationale:**
- **Cost Efficiency:** Serverless aligns with free-tier optimization goals (NFR-040)
- **Scalability:** Auto-scaling handles variable load without manual intervention (NFR-020, NFR-023)
- **Reduced Operational Overhead:** No server management required
- **Performance:** Can meet 5-second HTTP response time requirement (NFR-001)
- **MVP Speed:** Faster to deploy and iterate than managing servers

**Service Breakdown:**
1. **Web Application Service** - React SPA hosted on S3/CloudFront
2. **API Gateway Service** - RESTful API for document operations
3. **AI Service** - Python Lambda for Anthropic API integration
4. **Document Processing Service** - PDF/Word parsing and OCR
5. **Export Service** - Word/PDF generation
6. **Authentication Service** - User authentication and authorization

**Communication:**
- Synchronous: API Gateway → Lambda functions
- Asynchronous: SQS queues for long-running AI operations

---

## Technology Stack

### Frontend
**Framework:** React 18+

**Rationale:**
- Specified in project brief
- Large ecosystem and community support
- Excellent for complex, interactive UIs
- Strong typing with TypeScript support

**Supporting Libraries:**
- **State Management:** Redux Toolkit or Zustand (for complex state like collaborative editing)
- **UI Components:** Material-UI or Chakra UI (for rapid, accessible component development)
- **Rich Text Editor:** Draft.js, Slate, or TipTap (for document editing with collaboration features)
- **Real-Time Collaboration:** Y.js or Automerge (for Google Docs-style collaboration)
- **API Client:** React Query (for server state management and caching)
- **Routing:** React Router v6

### Backend
**Runtime:** Node.js 18+ (TypeScript)

**Rationale:**
- Specified in project brief
- Shares language with frontend (TypeScript)
- Excellent async I/O for handling multiple document uploads
- Large ecosystem for document processing

**Framework:** Express.js with Serverless Express adapter

**Supporting Libraries:**
- **Validation:** Zod or Joi (for request validation)
- **ORM:** Prisma (for type-safe database access)
- **Document Processing:** pdf-parse, mammoth (for Word documents), sharp (for images)
- **File Storage:** AWS SDK v3 (S3 operations)
- **Authentication:** AWS Cognito or Auth0

### AI Service
**Language:** Python 3.11+

**Rationale:**
- Specified in project brief
- Best ecosystem for AI/ML operations
- Official Anthropic Python SDK available
- Efficient text processing libraries

**Libraries:**
- **AI SDK:** Anthropic Python SDK or boto3 for AWS Bedrock
- **Text Processing:** spaCy or NLTK (for document analysis)
- **PDF Extraction:** PyPDF2 or pdfplumber
- **Async Support:** asyncio for concurrent processing

---

## Infrastructure and Deployment

### Cloud Provider
**Decision:** AWS

**Rationale:**
- Specified in project brief (AWS Lambda)
- Comprehensive service ecosystem
- AWS Bedrock available as Anthropic API alternative
- Free tier supports MVP deployment goals

**Core AWS Services:**
- **Compute:** Lambda (serverless functions)
- **API:** API Gateway (REST APIs)
- **Storage:** S3 (document storage), CloudFront (CDN)
- **Database:** RDS PostgreSQL (primary data)
- **Caching:** ElastiCache Redis (session management, query caching)
- **Queues:** SQS (asynchronous job processing)
- **Auth:** Cognito (user authentication)
- **Monitoring:** CloudWatch (logs and metrics)
- **CDN:** CloudFront (frontend distribution)

### Database
**Decision:** PostgreSQL (AWS RDS)

**Rationale:**
- Specified in project brief
- Robust relational model for structured data (users, firms, templates, documents)
- ACID compliance critical for legal documents
- JSON support for flexible template storage
- Strong full-text search capabilities
- AWS RDS provides managed service with automated backups

**Schema Design Principles:**
- Firm-based multi-tenancy with data isolation (NFR-014)
- Audit trail tables for compliance (NFR-012)
- Optimized indexes for common queries (NFR-002)

---

## AI Integration

**Decision:** Anthropic API (primary) with AWS Bedrock as fallback

**Rationale:**
- Anthropic Claude models excel at long-form content generation
- API-first approach simplifies integration
- AWS Bedrock provides same models with AWS-native integration
- Both support Claude 3 models suitable for legal document generation

**Model Selection:**
- **Primary:** Claude 3.5 Sonnet (balance of capability and cost)
- **Fallback:** Claude 3 Haiku (for simpler refinements, cost optimization)

**AI Architecture:**
- Prompt templates stored in database or config files
- Context window management for large source documents (up to 200MB → ~50 pages)
- Token usage tracking for cost management (NFR-041)
- Retry logic with exponential backoff (NFR-028)

---

## Authentication and Authorization

**Decision:** AWS Cognito for authentication, custom RBAC for authorization

**Rationale:**
- Managed service reduces security implementation burden
- Meets security requirements (NFR-009, NFR-010)
- Supports MFA for enhanced security
- OAuth 2.0 compliant

**Authorization Model:**
- **Roles:** Admin, Attorney, Paralegal, Read-Only
- **Permissions:** Firm-scoped (users can only access their firm's data)
- **Template Permissions:** Owner, Firm-wide, Private

---

## Testing Requirements

**Decision:** Unit + Integration Testing with Automated E2E for Critical Paths

**Rationale:**
- 80% code coverage requirement (NFR-036)
- Legal application requires high reliability
- Critical user workflows must be regression-tested

**Testing Stack:**
- **Frontend Unit Tests:** Jest + React Testing Library
- **Backend Unit Tests:** Jest + Supertest
- **Integration Tests:** Jest with test database
- **E2E Tests:** Playwright or Cypress (critical paths only)
- **API Testing:** Postman/Newman collections

**Test Coverage Goals:**
- Unit Tests: 80% code coverage minimum
- Integration Tests: All API endpoints
- E2E Tests: Core user workflows (upload → generate → export)

**Critical Paths for E2E Testing:**
1. Upload documents → Generate draft → Export to Word
2. Create template → Use template → Generate letter
3. Collaborate on document → Track changes → Approve
4. AI refinement → Multiple iterations → Accept changes

---

## CI/CD Pipeline

**Decision:** GitHub Actions for CI/CD with AWS deployment

**Workflow:**
1. **On Pull Request:**
   - Run linting (ESLint, Prettier)
   - Run unit tests
   - Run integration tests
   - Build frontend and backend
   - Report coverage

2. **On Merge to Main:**
   - Run full test suite
   - Build production artifacts
   - Deploy to staging environment
   - Run E2E tests against staging
   - (Manual approval gate)
   - Deploy to production

**Deployment Strategy:**
- Blue/green deployment for zero-downtime updates
- Automatic rollback on health check failures
- Database migrations managed with Prisma Migrate

---

## Document Storage Strategy

**Decision:** S3 with lifecycle policies

**Structure:**
- **Source Documents:** `/firms/{firmId}/sources/{documentId}/`
- **Generated Drafts:** `/firms/{firmId}/drafts/{letterId}/versions/`
- **Exported Finals:** `/firms/{firmId}/finals/{letterId}/`
- **Templates:** `/firms/{firmId}/templates/{templateId}/`

**Lifecycle Policies (NFR-042):**
- Source documents: Standard storage for 90 days → Glacier after 90 days
- Drafts: Standard storage for 30 days → Delete after 180 days (unless exported)
- Finals: Standard storage (retained per firm policy)
- Templates: Standard storage (actively used)

**Security:**
- Server-side encryption (AES-256) enabled (NFR-008)
- Presigned URLs for temporary access
- Firm-scoped access policies

---

## Real-Time Collaboration

**Decision:** WebSocket connections with operational transform

**Technology:** AWS AppSync (GraphQL subscriptions) or custom WebSocket (API Gateway WebSocket)

**Rationale:**
- Real-time collaboration is P1 requirement (FR-019)
- WebSocket provides low-latency bidirectional communication
- Operational transform ensures conflict-free concurrent editing

**Alternative Considered:** Polling-based updates (simpler but higher latency)

---

## Monitoring and Observability

**Decision:** CloudWatch + Sentry

**Monitoring:**
- **CloudWatch:** Infrastructure metrics, Lambda performance, API latency
- **Sentry:** Application error tracking and performance monitoring (NFR-039)
- **Custom Metrics:** AI generation time, cost per document, user activity

**Alerting:**
- P0 Alerts: System downtime, database failures, AI service unavailable
- P1 Alerts: Performance degradation, high error rates, cost anomalies

**Logging:**
- Structured logging (JSON format)
- Centralized logging in CloudWatch Logs
- Log retention: 30 days for debugging, 7 years for audit logs (NFR-012)

---

## Additional Technical Assumptions

### Document Size Limits
- Maximum source document: 50MB per file (NFR-006)
- Maximum batch upload: 200MB total
- Maximum generated demand letter: 50 pages (reasonable for most cases)

### Concurrency
- Database connection pooling configured for 100 concurrent users (NFR-005)
- Lambda concurrency limits set based on expected load
- Rate limiting on AI API to prevent cost overruns

### Data Retention
- User-configurable document retention policies (NFR-018)
- Default: Retain all data for 7 years (typical legal requirement)
- Audit logs retained for 7 years minimum

### Compliance Considerations
- HIPAA compliance may be required if handling medical records (to be confirmed)
- Attorney-client privilege protection (NFR-015)
- Right to be forgotten implementation (NFR-019)

### Development Environment
- Local development with Docker Compose (database, Redis)
- LocalStack for local AWS service emulation
- Shared staging environment for integration testing

### Code Quality
- TypeScript strict mode enabled
- ESLint + Prettier for code formatting
- Husky pre-commit hooks for linting and tests
- Conventional Commits for clear commit history

---

## Assumptions to Validate

The following assumptions should be validated with stakeholders or during architecture design:

1. **AI Model Access:** Confirm Anthropic API access and budget for token usage
2. **Medical Records:** Determine if HIPAA compliance is required
3. **Firm Branding:** Confirm if firms need custom branding in exported documents
4. **Third-Party Integrations:** Validate priority of document management system integrations (currently P2)
5. **Mobile Requirements:** Confirm mobile access is truly P2 or if basic mobile view is needed for MVP
6. **Data Residency:** Determine if any firms require data to be stored in specific geographic regions
7. **Single Sign-On:** Determine if SSO integration is required for enterprise clients

---

## Technology Decision Log

| Decision | Date | Rationale | Alternatives Considered |
|----------|------|-----------|------------------------|
| Monorepo | 2025-11-13 | Code sharing, atomic commits, simpler CI/CD | Polyrepo (more complex) |
| Serverless Architecture | 2025-11-13 | Cost efficiency, auto-scaling, MVP speed | EC2/ECS (more operational overhead) |
| PostgreSQL | 2025-11-13 | Required by brief, relational model fits use case | MongoDB (less suitable for structured data) |
| Anthropic API | 2025-11-13 | Best for long-form content, AWS Bedrock fallback | OpenAI GPT-4 (considered but Anthropic preferred for legal) |
| AWS Cognito | 2025-11-13 | Managed service, OAuth compliance, AWS integration | Auth0 (more features but additional cost) |
| React + TypeScript | 2025-11-13 | Required by brief, strong typing for reliability | Vue.js (not specified in brief) |
