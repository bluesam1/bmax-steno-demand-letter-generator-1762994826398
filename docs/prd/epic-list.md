# Epic List

## Overview

This document outlines the high-level epics for the Demand Letter Generator MVP. Each epic represents a significant, deployable increment of functionality that delivers value to users. Epics are sequenced to follow agile best practices: foundational infrastructure first, followed by core features, and then enhancement capabilities.

---

## Epic Sequencing Principles

1. **Epic 1 establishes foundation** - Project setup, core infrastructure, and a minimal working feature
2. **Each epic builds incrementally** - Subsequent epics add functionality on top of previous work
3. **Each epic is deployable** - Every epic results in a functional system that can be tested end-to-end
4. **Value delivery focus** - Each epic delivers tangible user or business value

---

## Epic 1: Foundation & Authentication
**Status:** Planned
**Priority:** P0 (Must-Have)
**Estimated Duration:** 2-3 weeks

### Goal
Establish project infrastructure, development environment, CI/CD pipeline, and user authentication system. Deploy a working "hello world" application to demonstrate the full deployment pipeline is functional.

### Value Delivered
- Development team can work efficiently with proper tooling
- Secure user authentication enables all future features
- CI/CD pipeline enables rapid iteration
- Firm-based multi-tenancy foundation established

### Key Capabilities
- Monorepo setup with React, Node.js, Python services
- AWS infrastructure provisioned (Lambda, API Gateway, RDS, S3, Cognito)
- User registration and login
- Firm creation and user-firm association
- Basic dashboard with "Hello World" content
- Automated testing and deployment pipeline

### Dependencies
- AWS account setup
- Anthropic API key or AWS Bedrock access
- Development environment provisioned

---

## Epic 2: Document Upload & Storage
**Status:** Planned
**Priority:** P0 (Must-Have)
**Estimated Duration:** 2 weeks

### Goal
Enable users to upload source documents (PDF, Word, images) and store them securely in S3 with proper organization and access control. Extract text from uploaded documents for AI processing.

### Value Delivered
- Users can upload case materials to the system
- Document extraction pipeline ready for AI consumption
- Secure, scalable document storage established
- Foundation for demand letter generation workflow

### Key Capabilities
- Multi-format file upload (PDF, DOCX, JPG, PNG)
- Batch upload support (multiple files at once)
- S3 storage with encryption and firm-based organization
- Text extraction from PDFs and Word documents
- OCR for scanned documents and images
- Document preview and management UI
- Source document metadata tracking

### Dependencies
- Epic 1 (authentication and infrastructure)
- AWS S3 and Lambda configured
- OCR service or library selected

---

## Epic 3: AI-Powered Draft Generation
**Status:** Planned
**Priority:** P0 (Must-Have)
**Estimated Duration:** 3 weeks

### Goal
Implement core AI functionality to analyze uploaded source documents and generate an initial draft demand letter. This is the primary value proposition of the product.

### Value Delivered
- **First usable product increment!** Users can generate demand letters with AI
- Attorneys save significant time on initial drafting
- Demonstration of AI capability builds user trust
- Core product workflow functional end-to-end

### Key Capabilities
- Anthropic API integration (Claude 3.5 Sonnet)
- Prompt engineering for demand letter generation
- Source document analysis and key information extraction
- Draft generation from source materials
- Draft display and review interface
- Version history (V1 is initial AI generation)
- Loading states and progress indicators during generation
- Error handling for AI service failures

### Dependencies
- Epic 2 (document upload and extraction)
- Anthropic API key active with sufficient quota
- Prompt templates designed and tested
- Token usage tracking implemented

---

## Epic 4: Template Management
**Status:** Planned
**Priority:** P0 (Must-Have)
**Estimated Duration:** 2 weeks

### Goal
Enable users to create, manage, and utilize firm-specific templates for demand letters, ensuring consistency and adherence to firm standards.

### Value Delivered
- Firms can standardize their demand letter format
- Templates with variables enable rapid customization
- Consistency across attorneys in the same firm
- Quality and professionalism of output improved

### Key Capabilities
- Template creation interface with rich text editor
- Variable system ({{client_name}}, {{incident_date}}, etc.)
- Template library browsing and search
- Firm-wide template sharing
- Private vs. firm-wide template visibility
- Template selection during demand letter generation
- AI generation using selected template structure
- Template usage tracking and analytics

### Dependencies
- Epic 3 (AI generation established)
- Rich text editor selected and integrated
- Variable substitution logic implemented

---

## Epic 5: AI Refinement & Iteration
**Status:** Planned
**Priority:** P0 (Must-Have)
**Estimated Duration:** 2 weeks

### Goal
Allow users to refine AI-generated drafts through natural language instructions, enabling iterative improvement of demand letters without manual editing.

### Value Delivered
- Users can customize and improve AI output efficiently
- Multiple refinement rounds enable perfection of content
- Maintains time-saving benefits while giving user control
- Builds trust through transparency and control

### Key Capabilities
- Chat-style AI refinement interface
- Natural language instruction processing
- Preview of proposed changes before applying
- Accept/reject refinement workflow
- Refinement history and undo capability
- Suggested refinement prompts (pre-built)
- Version creation for each refinement
- Context preservation across multiple refinements

### Dependencies
- Epic 3 (initial generation)
- Epic 4 (templates for structure)
- AI prompt templates for refinement tasks

---

## Epic 6: Document Export & Delivery
**Status:** Planned
**Priority:** P0 (Must-Have)
**Estimated Duration:** 1-2 weeks

### Goal
Enable users to export finalized demand letters to Word and PDF formats with proper formatting, and manage the final document delivery workflow.

### Value Delivered
- **Complete MVP workflow!** Users can generate and export demand letters
- Standard format compatibility (Word, PDF) for legal practice
- Professional output suitable for client and opposing counsel delivery
- Tracking of exported versions for compliance

### Key Capabilities
- Export to Microsoft Word (.docx) format
- Export to PDF format
- Formatting preservation (headers, footers, fonts, spacing)
- Firm branding elements in exported documents (if configured)
- Download functionality with secure presigned URLs
- Export history and tracking
- Version snapshot at export time
- Audit logging of exports

### Dependencies
- Epic 5 (refinement complete, content finalized)
- Document generation libraries selected (docx, PDF)
- Firm branding configuration system

---

## Epic 7: Collaborative Editing (P1 - Post-MVP)
**Status:** Planned (Post-MVP)
**Priority:** P1 (Should-Have)
**Estimated Duration:** 3-4 weeks

### Goal
Enable real-time collaboration on demand letters with multiple users editing simultaneously, tracking changes, and adding comments.

### Value Delivered
- Attorneys and paralegals can work together efficiently
- Change tracking ensures accountability and review
- Comments enable discussion without external tools
- Team productivity increased through collaboration

### Key Capabilities
- Real-time collaborative editing (Google Docs style)
- Presence indicators (who's viewing/editing)
- Change tracking with attribution
- Inline comments and threaded discussions
- Resolve/unresolve comment workflow
- Collaboration permissions (view, comment, edit)
- Notification system for comments and changes

### Dependencies
- Epic 6 (core MVP complete)
- Real-time collaboration library selected (Y.js, Automerge)
- WebSocket infrastructure (API Gateway WebSocket or AppSync)

---

## Epic 8: Advanced Features & Optimization (P2 - Future)
**Status:** Future Consideration
**Priority:** P2 (Nice-to-Have)
**Estimated Duration:** Ongoing

### Goal
Continuously improve the product based on user feedback, add integrations, and optimize for scale and cost efficiency.

### Potential Features
- Document management system integrations
- Case management platform integrations
- Email delivery of demand letters
- Settlement calculator
- Deadline tracking
- Usage analytics dashboard
- Mobile responsive improvements
- Performance optimizations
- AI model fine-tuning with firm-specific data

### Approach
- Prioritize based on user feedback and business value
- Incremental releases of individual features
- Continuous optimization of existing features

---

## MVP Scope Definition

### What's In MVP (Epics 1-6)
The MVP includes all functionality required for a user to:
1. Register and log in
2. Upload source documents
3. Generate an AI-powered draft demand letter
4. Use or create templates for consistency
5. Refine the draft with AI assistance
6. Export the final document to Word or PDF

**Timeline:** Approximately 12-14 weeks for full MVP

### What's Post-MVP (Epic 7+)
Collaboration features, integrations, and advanced analytics are valuable but not required for the core workflow. These will be prioritized based on:
- User feedback from MVP
- Usage metrics and adoption rates
- Business value and ROI
- Technical feasibility and effort

---

## Epic Dependency Chart

```
Epic 1: Foundation & Authentication
    ↓
Epic 2: Document Upload & Storage
    ↓
Epic 3: AI-Powered Draft Generation
    ↓
Epic 4: Template Management
    ↓
Epic 5: AI Refinement & Iteration
    ↓
Epic 6: Document Export & Delivery
    ↓ (MVP Complete)
Epic 7: Collaborative Editing (Post-MVP)
    ↓
Epic 8: Advanced Features (Ongoing)
```

---

## Release Strategy

### Phase 1: Internal Alpha (After Epic 3)
- **Audience:** Steno internal team
- **Goal:** Validate core AI generation capability
- **Duration:** 1-2 weeks
- **Success Criteria:** 5+ successful demand letter generations

### Phase 2: Closed Beta (After Epic 6 - MVP Complete)
- **Audience:** 3-5 friendly law firm clients
- **Goal:** Validate full workflow and gather feedback
- **Duration:** 4 weeks
- **Success Criteria:**
  - 50+ demand letters generated
  - Time savings of 40%+ reported
  - No critical bugs blocking usage

### Phase 3: General Availability
- **Audience:** All existing Steno clients, new prospects
- **Goal:** Drive adoption and revenue
- **Timing:** After Beta success criteria met
- **Success Criteria:**
  - 80% user adoption (from brief)
  - 50% time savings (from brief)
  - Positive NPS score

---

## Epic Acceptance Criteria

Each epic is considered complete when:
- [ ] All user stories in the epic are implemented
- [ ] Unit tests passing with 80% coverage
- [ ] Integration tests passing
- [ ] E2E tests for critical paths passing
- [ ] Code review approved
- [ ] Deployed to staging and tested
- [ ] Product owner acceptance
- [ ] Documentation updated
- [ ] No P0 or P1 bugs outstanding

---

## Risk Mitigation by Epic

### Epic 1 Risks
- **Risk:** Infrastructure setup delays
- **Mitigation:** Start early, use infrastructure as code, seek AWS support if needed

### Epic 3 Risks
- **Risk:** AI quality insufficient for legal documents
- **Mitigation:** Extensive prompt engineering, testing with real documents, attorney review

### Epic 5 Risks
- **Risk:** Refinement workflow too complex
- **Mitigation:** User testing, simplified UX, pre-built prompt suggestions

### Epic 7 Risks
- **Risk:** Real-time collaboration complexity
- **Mitigation:** Use proven libraries (Y.js), limit initial scope, thorough testing

---

## Next Steps

1. **Epic 1 Story Breakdown:** Create detailed user stories for Foundation & Authentication epic
2. **Sprint Planning:** Allocate stories to 2-week sprints
3. **Architecture Design:** Technical architecture to support all epics
4. **Design Mockups:** UX design for Epic 1-3 screens to guide development
