# Data Model

This document describes the core data entities and their relationships for the Demand Letter Generator. This logical data model will guide database schema design and API contract development.

## Entity Relationship Overview

```
Firm (1) ──── (N) User
Firm (1) ──── (N) Template
Firm (1) ──── (N) DemandLetter
User (1) ──── (N) DemandLetter (creator)
DemandLetter (1) ──── (N) SourceDocument
DemandLetter (1) ──── (N) Version
DemandLetter (1) ──── (N) Collaboration
Template (1) ──── (N) DemandLetter (uses template)
User (N) ──── (N) DemandLetter (via Collaboration)
```

---

## Core Entities

### Firm
Represents a law firm organization using the system.

**Attributes:**
- `id` (UUID, PK) - Unique identifier
- `name` (String, required) - Firm name
- `subscription_tier` (Enum: Free, Professional, Enterprise) - Subscription level
- `settings` (JSON) - Firm-wide configuration
- `created_at` (Timestamp)
- `updated_at` (Timestamp)
- `is_active` (Boolean) - Account status

**Relationships:**
- Has many Users
- Has many Templates
- Has many DemandLetters

**Business Rules:**
- Firm data must be logically isolated (NFR-014)
- Firm can set default templates
- Firm can configure branding elements (stored in settings JSON)

---

### User
Represents an individual user (attorney, paralegal, admin) within a firm.

**Attributes:**
- `id` (UUID, PK) - Unique identifier
- `firm_id` (UUID, FK) - Foreign key to Firm
- `email` (String, unique, required) - User email
- `name` (String, required) - Full name
- `role` (Enum: Admin, Attorney, Paralegal, ReadOnly) - User role
- `auth_provider_id` (String) - Cognito user ID
- `preferences` (JSON) - User preferences
- `created_at` (Timestamp)
- `updated_at` (Timestamp)
- `last_login_at` (Timestamp)
- `is_active` (Boolean) - Account status

**Relationships:**
- Belongs to one Firm
- Creates many DemandLetters (as creator)
- Collaborates on many DemandLetters
- Creates many Templates

**Business Rules:**
- Email must be unique across the system
- Users can only access resources within their firm
- Role determines permissions (RBAC)

---

### Template
Represents a reusable demand letter template.

**Attributes:**
- `id` (UUID, PK) - Unique identifier
- `firm_id` (UUID, FK) - Foreign key to Firm
- `creator_id` (UUID, FK) - Foreign key to User who created it
- `name` (String, required) - Template name
- `description` (Text) - Template description
- `content` (Text, required) - Template content with variables
- `variables` (JSON) - Array of variable definitions with metadata
- `category` (String) - Case type category (e.g., "Personal Injury", "Contract Dispute")
- `visibility` (Enum: Private, FirmWide) - Sharing level
- `is_default` (Boolean) - Whether this is the firm's default template
- `usage_count` (Integer) - Number of times used (for analytics)
- `created_at` (Timestamp)
- `updated_at` (Timestamp)
- `is_active` (Boolean) - Soft delete flag

**Relationships:**
- Belongs to one Firm
- Created by one User
- Used by many DemandLetters

**Business Rules:**
- Variables use double-brace syntax: `{{variable_name}}`
- Only one default template per firm per category
- Private templates only visible to creator
- FirmWide templates visible to all users in firm

**Example Variables JSON:**
```json
[
  {"name": "client_name", "type": "string", "required": true, "description": "Full name of client"},
  {"name": "incident_date", "type": "date", "required": true, "description": "Date of incident"},
  {"name": "damages_amount", "type": "currency", "required": false, "description": "Total damages claimed"}
]
```

---

### DemandLetter
Represents a demand letter document being created or completed.

**Attributes:**
- `id` (UUID, PK) - Unique identifier
- `firm_id` (UUID, FK) - Foreign key to Firm
- `creator_id` (UUID, FK) - Foreign key to User who created it
- `template_id` (UUID, FK, nullable) - Foreign key to Template (if used)
- `title` (String, required) - Document title
- `client_name` (String) - Client name for organization
- `case_reference` (String) - Case number or reference
- `status` (Enum: Draft, InReview, Final, Exported) - Document status
- `current_version_id` (UUID, FK) - Foreign key to current Version
- `metadata` (JSON) - Additional metadata (client info, case details)
- `created_at` (Timestamp)
- `updated_at` (Timestamp)
- `completed_at` (Timestamp, nullable) - When finalized
- `is_archived` (Boolean) - Archive status

**Relationships:**
- Belongs to one Firm
- Created by one User
- Optionally uses one Template
- Has many SourceDocuments
- Has many Versions
- Has many Collaborations
- Has current Version (points to latest)

**Business Rules:**
- Status transitions: Draft → InReview → Final → Exported
- Once status is Final, content should be locked (versioned)
- All modifications create new versions
- Firm-scoped access only

---

### SourceDocument
Represents uploaded source materials for AI analysis.

**Attributes:**
- `id` (UUID, PK) - Unique identifier
- `demand_letter_id` (UUID, FK) - Foreign key to DemandLetter
- `uploaded_by_id` (UUID, FK) - Foreign key to User
- `filename` (String, required) - Original filename
- `file_type` (String) - MIME type
- `file_size` (Integer) - Size in bytes
- `s3_key` (String, required) - S3 storage key
- `s3_bucket` (String, required) - S3 bucket name
- `document_type` (Enum: MedicalRecord, Correspondence, Photo, Report, Other) - Categorization
- `extracted_text` (Text) - OCR/extracted text content
- `extraction_status` (Enum: Pending, Processing, Completed, Failed) - Processing status
- `page_count` (Integer) - Number of pages
- `created_at` (Timestamp)
- `updated_at` (Timestamp)

**Relationships:**
- Belongs to one DemandLetter
- Uploaded by one User

**Business Rules:**
- Files stored in S3 with encryption (NFR-008)
- Maximum file size 50MB (NFR-006)
- Text extraction happens asynchronously
- S3 keys use structure: `/firms/{firmId}/sources/{documentId}/{filename}`

---

### Version
Represents a version of a demand letter's content.

**Attributes:**
- `id` (UUID, PK) - Unique identifier
- `demand_letter_id` (UUID, FK) - Foreign key to DemandLetter
- `version_number` (Integer, required) - Sequential version number
- `content` (Text, required) - Full document content
- `created_by_id` (UUID, FK) - Foreign key to User who created this version
- `change_description` (String) - Description of changes made
- `ai_refinement_prompt` (Text, nullable) - AI prompt used (if AI-generated)
- `created_at` (Timestamp)
- `is_current` (Boolean) - Whether this is the current version

**Relationships:**
- Belongs to one DemandLetter
- Created by one User

**Business Rules:**
- Version numbers are sequential starting at 1
- Only one version can be current (`is_current = true`)
- All content changes create new version (audit trail)
- Version 1 is the initial AI-generated draft

---

### Collaboration
Represents a user's participation in editing a demand letter.

**Attributes:**
- `id` (UUID, PK) - Unique identifier
- `demand_letter_id` (UUID, FK) - Foreign key to DemandLetter
- `user_id` (UUID, FK) - Foreign key to User
- `permission` (Enum: View, Comment, Edit) - Access level
- `invited_by_id` (UUID, FK) - Foreign key to User who granted access
- `created_at` (Timestamp)
- `last_accessed_at` (Timestamp) - Last time user viewed/edited

**Relationships:**
- Belongs to one DemandLetter
- Belongs to one User
- Invited by one User

**Business Rules:**
- Creator automatically gets Edit permission
- Users can only collaborate on letters within their firm
- Minimum one user with Edit permission must exist

---

### Comment
Represents comments on specific parts of a demand letter.

**Attributes:**
- `id` (UUID, PK) - Unique identifier
- `demand_letter_id` (UUID, FK) - Foreign key to DemandLetter
- `user_id` (UUID, FK) - Foreign key to User
- `parent_comment_id` (UUID, FK, nullable) - For threaded replies
- `content` (Text, required) - Comment text
- `position` (JSON) - Location in document (paragraph, character offset)
- `is_resolved` (Boolean) - Resolution status
- `resolved_by_id` (UUID, FK, nullable) - User who resolved it
- `resolved_at` (Timestamp, nullable)
- `created_at` (Timestamp)
- `updated_at` (Timestamp)

**Relationships:**
- Belongs to one DemandLetter
- Created by one User
- Optionally has parent Comment (for threading)
- Optionally resolved by one User

**Business Rules:**
- Comments can be threaded (replies)
- Only creator or admin can resolve comments
- Position JSON format: `{"paragraph": 5, "offset": 120}`

---

### Export
Represents an exported document.

**Attributes:**
- `id` (UUID, PK) - Unique identifier
- `demand_letter_id` (UUID, FK) - Foreign key to DemandLetter
- `version_id` (UUID, FK) - Foreign key to Version exported
- `exported_by_id` (UUID, FK) - Foreign key to User
- `format` (Enum: DOCX, PDF) - Export format
- `s3_key` (String, required) - S3 storage key
- `s3_bucket` (String, required) - S3 bucket name
- `file_size` (Integer) - Size in bytes
- `created_at` (Timestamp)
- `download_count` (Integer) - Number of downloads

**Relationships:**
- Belongs to one DemandLetter
- Exports specific Version
- Created by one User

**Business Rules:**
- Exports are snapshots of specific versions
- S3 keys use structure: `/firms/{firmId}/finals/{documentId}/v{versionNumber}.{format}`
- Presigned URLs for secure downloads

---

### AuditLog
Represents audit trail for compliance and security.

**Attributes:**
- `id` (UUID, PK) - Unique identifier
- `firm_id` (UUID, FK) - Foreign key to Firm
- `user_id` (UUID, FK, nullable) - Foreign key to User (null for system actions)
- `resource_type` (String) - Type of resource (DemandLetter, Template, User, etc.)
- `resource_id` (UUID) - ID of resource
- `action` (String) - Action performed (created, updated, deleted, viewed, exported)
- `changes` (JSON) - Details of changes made
- `ip_address` (String) - User IP address
- `user_agent` (String) - User browser/client
- `created_at` (Timestamp)

**Relationships:**
- Belongs to one Firm
- Optionally belongs to one User

**Business Rules:**
- All document access, modifications, and exports must be logged (NFR-012)
- Audit logs retained for 7 years minimum
- Cannot be modified or deleted (append-only)
- Critical for legal compliance

---

## Supporting Entities

### AIGenerationLog
Tracks AI usage for cost management and analytics.

**Attributes:**
- `id` (UUID, PK)
- `demand_letter_id` (UUID, FK)
- `user_id` (UUID, FK)
- `operation_type` (Enum: InitialGeneration, Refinement)
- `prompt_tokens` (Integer) - Tokens in prompt
- `completion_tokens` (Integer) - Tokens in completion
- `total_tokens` (Integer) - Total tokens used
- `model_used` (String) - AI model name
- `cost_estimate` (Decimal) - Estimated cost in USD
- `duration_ms` (Integer) - Processing time in milliseconds
- `created_at` (Timestamp)

**Purpose:** Cost tracking, analytics, optimization (NFR-041)

---

## Data Access Patterns

### Common Queries

**Get User's Recent Demand Letters:**
```sql
SELECT dl.* FROM demand_letters dl
WHERE dl.creator_id = :userId
  AND dl.firm_id = :firmId
ORDER BY dl.updated_at DESC
LIMIT 20;
```

**Get Firm Templates by Category:**
```sql
SELECT t.* FROM templates t
WHERE t.firm_id = :firmId
  AND t.category = :category
  AND (t.visibility = 'FirmWide' OR t.creator_id = :userId)
  AND t.is_active = true
ORDER BY t.usage_count DESC;
```

**Get Current Version Content:**
```sql
SELECT v.content FROM versions v
JOIN demand_letters dl ON v.demand_letter_id = dl.id
WHERE dl.id = :demandLetterId
  AND v.is_current = true;
```

**Get Collaboration Users:**
```sql
SELECT u.*, c.permission FROM collaborations c
JOIN users u ON c.user_id = u.id
WHERE c.demand_letter_id = :demandLetterId;
```

### Indexes Required

**Critical Indexes for Performance (NFR-002):**
- `users(email)` - UNIQUE index for login
- `users(firm_id)` - Firm-based queries
- `demand_letters(firm_id, updated_at)` - Recent letters listing
- `demand_letters(creator_id, updated_at)` - User's letters
- `templates(firm_id, category)` - Template lookup
- `versions(demand_letter_id, is_current)` - Current version queries
- `source_documents(demand_letter_id)` - Document loading
- `collaborations(demand_letter_id)` - Collaboration checks
- `collaborations(user_id, demand_letter_id)` - Permission checks
- `audit_logs(firm_id, created_at)` - Audit trail queries
- `audit_logs(resource_type, resource_id)` - Resource-specific audits

---

## Data Security and Privacy

### Encryption
- All sensitive text fields encrypted at rest (NFR-008)
- S3 buckets use server-side encryption (AES-256)
- Database connections use TLS

### Multi-Tenancy Isolation
- All queries must filter by `firm_id` (NFR-014)
- Row-level security policies in PostgreSQL
- Application-level firm context validation

### Data Retention
- Active documents: No automatic deletion
- Archived documents: Retained per firm policy (configurable)
- Audit logs: 7-year minimum retention (NFR-012)
- Source documents: Lifecycle policy moves to Glacier after 90 days

### GDPR Compliance (NFR-016, NFR-019)
- User data can be exported (data portability)
- User data can be permanently deleted (right to erasure)
- Deletion cascade: User → DemandLetters → Versions → SourceDocuments
- Audit logs anonymized on user deletion (retain logs, anonymize user)

---

## Schema Evolution

**Migration Strategy:**
- Use Prisma Migrate for schema changes
- All migrations must be reversible
- Test migrations on staging before production
- Zero-downtime migrations required (add columns, deprecate old)

**Version 1.0 Schema:** Initial schema as described above
**Future Considerations:**
- Analytics tables for usage metrics
- Notification preferences for collaboration
- Saved AI prompt templates
- Document signature tracking (for future e-signature feature)
