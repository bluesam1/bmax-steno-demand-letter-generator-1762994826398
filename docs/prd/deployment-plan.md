# Deployment Plan

## Overview

This deployment plan outlines the strategy for deploying the Demand Letter Generator to production, including environment setup, deployment process, rollback procedures, and post-deployment validation. The plan prioritizes zero-downtime deployments, security, and cost optimization.

---

## Deployment Environments

### 1. Development Environment
**Purpose:** Local development and unit testing

**Infrastructure:**
- Local machines with Docker Compose
- PostgreSQL container
- Redis container
- LocalStack for AWS service emulation
- Mock AI service (to avoid API costs during development)

**Access:** All developers

**Data:** Synthetic test data, no real customer data

---

### 2. Staging Environment
**Purpose:** Integration testing, QA, and pre-production validation

**Infrastructure:**
- AWS environment mirroring production (scaled down)
- RDS PostgreSQL (db.t3.micro - free tier eligible)
- ElastiCache Redis (cache.t3.micro)
- Lambda functions with reduced concurrency limits
- S3 buckets with test data
- Cognito user pool (separate from production)
- CloudFront distribution

**Access:** Development team, QA team, selected stakeholders

**Data:** Realistic test data, anonymized from production or synthetic

**Deployment:** Automatic on merge to `main` branch

**Cost Target:** Stay within AWS free tier limits where possible

---

### 3. Production Environment
**Purpose:** Live customer-facing environment

**Infrastructure:**
- AWS production account (separate from staging)
- RDS PostgreSQL (db.t3.small initially, auto-scaling configured)
- ElastiCache Redis (cache.t3.small)
- Lambda functions with concurrency limits based on load
- S3 buckets with encryption and lifecycle policies
- Cognito user pool
- CloudFront distribution with WAF protection
- Route 53 for DNS management

**Access:** Operations team only (deployment automation), read-only monitoring access for developers

**Data:** Real customer data with full encryption and compliance controls

**Deployment:** Manual approval required after staging validation

**Monitoring:** Full observability with CloudWatch, Sentry, and custom metrics

---

## AWS Infrastructure Setup

### Account Structure
```
AWS Organization
├── Production Account
│   ├── us-east-1 (primary region)
│   └── us-west-2 (DR failover - future)
└── Non-Production Account
    ├── Staging Environment
    └── Development Sandbox
```

**Rationale:** Separate accounts for security isolation and cost tracking

---

### Infrastructure as Code (IaC)

**Tool:** AWS CDK (TypeScript) or Terraform

**Resources Defined in IaC:**
- VPC and networking
- RDS PostgreSQL instances
- ElastiCache Redis clusters
- S3 buckets with policies
- Lambda functions
- API Gateway configurations
- Cognito user pools
- CloudWatch alarms
- IAM roles and policies
- CloudFront distributions

**Benefits:**
- Version-controlled infrastructure
- Reproducible deployments
- Easy environment replication
- Disaster recovery capability

**Repository Structure:**
```
/infrastructure
  /staging
    - stack.ts (or main.tf)
  /production
    - stack.ts (or main.tf)
  /shared
    - constructs (reusable components)
```

---

## Deployment Pipeline

### CI/CD Tool: GitHub Actions

### Pipeline Stages

#### Stage 1: Build & Test (On Pull Request)
```yaml
1. Checkout code
2. Install dependencies (npm install, pip install)
3. Run linters (ESLint, Prettier, pylint)
4. Run type checking (TypeScript)
5. Run unit tests (Jest, pytest)
6. Generate coverage reports
7. Build frontend (React build)
8. Build backend (TypeScript compilation)
9. Run integration tests
10. Report results to PR
```

**Pass Criteria:**
- All tests pass
- 80% code coverage maintained
- No linting errors
- Successful builds

---

#### Stage 2: Deploy to Staging (On Merge to Main)
```yaml
1. Run full build & test suite
2. Build Docker images (if using containers)
3. Package Lambda functions
4. Deploy infrastructure changes (CDK/Terraform)
5. Run database migrations (Prisma Migrate)
6. Deploy Lambda functions
7. Deploy frontend to S3
8. Invalidate CloudFront cache
9. Run smoke tests
10. Run E2E test suite against staging
11. Generate deployment report
12. Notify team in Slack/Email
```

**Rollback Trigger:**
- Smoke tests fail
- E2E tests fail
- Health check failures

---

#### Stage 3: Deploy to Production (Manual Approval)
```yaml
1. Manual approval gate (requires 2 approvers)
2. Run pre-deployment checks:
   - Verify staging tests passed
   - Check no critical alerts in production
   - Verify database migration plan
3. Enable maintenance mode banner (optional for major changes)
4. Deploy infrastructure changes
5. Run database migrations (with backup)
6. Deploy Lambda functions (blue/green)
7. Gradual traffic shift (10% → 50% → 100%)
8. Deploy frontend to S3
9. Invalidate CloudFront cache
10. Run production smoke tests
11. Monitor metrics for 15 minutes
12. Complete deployment or rollback
13. Disable maintenance mode
14. Notify stakeholders
```

**Deployment Strategy:** Blue/Green with gradual traffic shifting

**Rollback Trigger:**
- Error rate exceeds 1%
- Response time exceeds SLA
- Critical functionality fails smoke tests
- Database migration fails

---

## Database Migration Strategy

### Tool: Prisma Migrate

### Migration Process

**Pre-Migration:**
1. Review migration plan with team
2. Test migration on staging with production data snapshot
3. Estimate migration time
4. Create production database backup
5. Schedule maintenance window if needed (for major migrations)

**Migration Execution:**
```bash
# Staging
npm run migrate:staging

# Production (automated in pipeline)
npm run migrate:production
```

**Migration Types:**

**Type A - Additive Changes (No Downtime):**
- Adding new tables
- Adding nullable columns
- Adding indexes (with `CONCURRENTLY` flag)
- **Strategy:** Deploy during business hours, no maintenance window

**Type B - Modification Changes (Minimal Downtime):**
- Renaming columns (with transition period)
- Changing column types
- **Strategy:** Multi-phase deployment:
  1. Add new column, deploy code using both old and new
  2. Backfill data
  3. Switch code to use new column only
  4. Remove old column in subsequent deployment

**Type C - Breaking Changes (Requires Coordination):**
- Removing tables
- Removing columns
- Complex schema restructuring
- **Strategy:** Maintenance window (low-traffic hours), communication plan

**Post-Migration:**
1. Verify migration success
2. Run data integrity checks
3. Monitor application logs for errors
4. Keep backup for 7 days before purging

---

## Rollback Procedures

### Automated Rollback Triggers
- Health check failures (3 consecutive failures)
- Error rate >1% for 5 minutes
- P99 latency >10 seconds for 5 minutes
- Critical functionality smoke test failures

### Manual Rollback Decision Points
- User-reported critical issues
- Data integrity concerns
- Security vulnerability discovered in new code

### Rollback Process

#### Application Rollback (Lambda)
```bash
# Automated via deployment pipeline
1. Shift traffic back to previous version (blue/green)
2. Verify previous version health
3. Mark deployment as failed
4. Alert team
5. Preserve logs for investigation
```

**Time to Rollback:** <5 minutes

---

#### Database Rollback
**Warning:** Database rollbacks are complex and risky

**Option 1 - Forward Fix (Preferred):**
- Deploy hotfix with corrected migration
- Preserve data integrity

**Option 2 - Restore from Backup (Last Resort):**
```bash
1. Stop application traffic (maintenance mode)
2. Restore database from pre-migration backup
3. Verify data integrity
4. Restart application with previous version
5. Data created during failed deployment is LOST
```

**Time to Rollback:** 30-60 minutes (depends on database size)

**Prevention:** Thorough testing in staging with production-like data

---

## Deployment Checklist

### Pre-Deployment
- [ ] All tests passing in staging
- [ ] Code review approved by 2+ team members
- [ ] Security scan completed (Dependabot, Snyk)
- [ ] Database migration plan reviewed and tested
- [ ] Rollback plan documented
- [ ] Stakeholders notified of deployment window
- [ ] On-call engineer assigned
- [ ] Monitoring dashboards ready
- [ ] Deployment notes prepared

### During Deployment
- [ ] Deployment pipeline initiated
- [ ] Migration completed successfully
- [ ] Smoke tests passing
- [ ] Health checks passing
- [ ] Metrics within normal ranges
- [ ] No spike in error rates
- [ ] Sample user workflows tested manually

### Post-Deployment
- [ ] Monitor metrics for 30 minutes
- [ ] Review error logs
- [ ] Verify AI service integration
- [ ] Test critical user workflows:
  - [ ] User login
  - [ ] Document upload
  - [ ] AI generation
  - [ ] Document export
- [ ] Stakeholder notification of successful deployment
- [ ] Update deployment log
- [ ] Create post-deployment report

---

## Monitoring and Observability

### Key Metrics to Monitor During Deployment

**Application Health:**
- Lambda error rate (target: <0.1%)
- Lambda duration (target: <5 seconds for API calls)
- API Gateway 5xx errors (target: 0)
- API Gateway 4xx errors (spike indicates client issues)

**Database Health:**
- Connection pool utilization (target: <80%)
- Query duration P95 (target: <2 seconds)
- Database CPU utilization (target: <70%)
- Replication lag (if using read replicas)

**AI Service Health:**
- Anthropic API success rate (target: >99%)
- Anthropic API latency (target: <30 seconds)
- AI token usage rate (cost monitoring)

**User Experience:**
- Document generation success rate (target: >98%)
- Document export success rate (target: >99%)
- End-to-end workflow completion rate

**Business Metrics:**
- Active users (should not drop)
- Documents generated per hour
- Average time per generation

### Alerts Configuration

**P0 Alerts (Immediate Response Required):**
- Production system down (all health checks failing)
- Database connection failures
- AI service unavailable
- Error rate >1% for >5 minutes

**P1 Alerts (Response Within 1 Hour):**
- Performance degradation (latency >2x normal)
- Elevated error rates (0.5-1%)
- Database CPU >85%
- Lambda throttling occurring

**P2 Alerts (Review Next Business Day):**
- Cost anomalies (>20% increase)
- Disk space warnings
- Backup failures
- Low test coverage in new PR

---

## Security Hardening for Production

### Pre-Deployment Security Checks
- [ ] All secrets stored in AWS Secrets Manager (not environment variables)
- [ ] IAM roles follow principle of least privilege
- [ ] S3 buckets are not publicly accessible
- [ ] Database is in private subnet (no public IP)
- [ ] API Gateway has rate limiting configured
- [ ] CloudFront WAF rules enabled
- [ ] TLS 1.3 enforced on all endpoints
- [ ] CORS policies properly configured
- [ ] Dependency vulnerability scan passed
- [ ] Code security scan passed (CodeQL, SonarQube)

### Post-Deployment Security Validation
- [ ] Penetration testing scheduled (for major releases)
- [ ] Security audit logs enabled and streaming
- [ ] Intrusion detection configured
- [ ] Data encryption verified (at rest and in transit)

---

## Disaster Recovery Plan

### Backup Strategy

**Database Backups:**
- Automated daily snapshots (RDS automated backups)
- Retention: 30 days
- Point-in-time recovery enabled (up to 5 minutes before failure)
- Weekly manual snapshot retained for 90 days

**Document Storage Backups:**
- S3 versioning enabled
- S3 Cross-Region Replication (to secondary region)
- Lifecycle policy: Glacier archival after 90 days

**Configuration Backups:**
- Infrastructure as Code in version control
- Secrets backed up in AWS Secrets Manager with replication
- Lambda function versions retained

### Recovery Procedures

**Scenario 1: Regional Outage**
- **RTO:** 4 hours
- **RPO:** 5 minutes
- **Procedure:**
  1. Failover to secondary region (manual process)
  2. Update DNS to point to DR region
  3. Promote read replica to primary (if cross-region replication configured)
  4. Restore services in DR region

**Scenario 2: Database Corruption**
- **RTO:** 2 hours
- **RPO:** 1 hour (last backup)
- **Procedure:**
  1. Identify corruption extent
  2. Restore from most recent clean snapshot
  3. Apply transaction logs if point-in-time recovery needed
  4. Validate data integrity
  5. Resume operations

**Scenario 3: Application Failure**
- **RTO:** 15 minutes
- **RPO:** 0 (no data loss)
- **Procedure:**
  1. Automatic rollback to previous version
  2. Investigate root cause
  3. Deploy hotfix
  4. Resume normal operations

---

## Cost Optimization Strategy

### Initial Deployment Costs (Estimated)

**Staging Environment (~$50-100/month):**
- RDS db.t3.micro: ~$15/month
- ElastiCache cache.t3.micro: ~$12/month
- Lambda: Minimal (within free tier)
- S3: ~$5/month
- CloudFront: ~$5/month
- Data transfer: ~$10/month

**Production Environment (~$200-400/month for MVP):**
- RDS db.t3.small: ~$30/month (auto-scaling configured)
- ElastiCache cache.t3.small: ~$25/month
- Lambda: ~$50/month (depends on usage)
- S3: ~$20/month (1000 documents/month)
- CloudFront: ~$20/month
- Cognito: Free tier covers initial users
- Anthropic API: ~$100-200/month (depends on usage)
- Data transfer: ~$20/month
- Monitoring (Sentry): ~$30/month

**Total Initial Monthly Cost:** ~$300-500/month

### Cost Monitoring
- Daily cost reports sent to stakeholders
- Budget alerts at 80% and 100% of monthly target
- Per-customer cost tracking for pricing model validation
- AI token usage tracked per document (NFR-041)

### Cost Optimization Actions
- Right-size RDS instances based on actual load (start small, scale up)
- Use reserved instances for predictable workloads (after 6 months)
- Implement S3 lifecycle policies (NFR-042)
- Optimize Lambda memory allocation based on performance data
- Cache frequently accessed data in Redis
- Use CloudFront caching aggressively

---

## Post-Deployment Validation

### Immediate Validation (First 24 Hours)
- Monitor error rates continuously
- Review all critical user workflows
- Check AI generation quality (sample review)
- Validate export functionality
- Test collaboration features
- Review security logs for anomalies

### Week 1 Validation
- Analyze user adoption metrics
- Review performance metrics vs. SLA
- Collect initial user feedback
- Identify any usability issues
- Review cost actuals vs. estimates

### Week 4 Validation
- Conduct user interviews (5-10 users)
- Review success metrics against goals:
  - Time savings measured?
  - User satisfaction scores
  - Adoption rate
- Plan iterative improvements

---

## Deployment Schedule

### MVP Release Timeline

**Week -4: Infrastructure Setup**
- Create AWS accounts (production, staging)
- Set up VPC, subnets, security groups
- Configure RDS, ElastiCache
- Set up CI/CD pipelines
- Configure monitoring and alerting

**Week -2: Staging Deployment**
- Deploy MVP to staging
- Conduct internal testing
- Load testing and performance tuning
- Security audit
- Fix critical issues

**Week -1: Pre-Production**
- Staging sign-off from QA
- Stakeholder demo and approval
- Final security review
- Create deployment runbook
- Schedule production deployment

**Week 0: Production Deployment**
- **Day 1 (Monday):** Deploy to production (morning)
- **Day 1-2:** Monitor closely, gather early feedback
- **Day 3:** Review metrics, make adjustments
- **Day 5 (Friday):** Week 1 retrospective

**Week 1-4: Stabilization**
- Bug fixes and hotfixes as needed
- Performance optimization
- User onboarding support
- Gather feedback for next iteration

---

## Deployment Communication Plan

### Stakeholder Notifications

**Pre-Deployment:**
- Email to all stakeholders 1 week before
- Slack announcement 1 day before
- In-app banner for staging users

**During Deployment:**
- Slack updates at key milestones
- Status page updated (if applicable)
- On-call team on standby

**Post-Deployment:**
- Success announcement in Slack/Email
- Deployment notes shared
- Known issues documented
- Feedback channels opened

### User Communication

**For Major Releases:**
- Release notes published
- What's new highlights
- Video walkthrough of new features (if significant)
- Email to active users

**For Minor Releases:**
- In-app changelog
- Brief release notes

---

## Continuous Improvement

### Post-Deployment Review (After Each Deployment)
- What went well?
- What could be improved?
- Were there any surprises?
- Did rollback plan work (if used)?
- Update deployment playbook

### Quarterly Review
- Review deployment success rate
- Analyze deployment duration trends
- Evaluate MTTR (Mean Time To Recovery)
- Assess cost efficiency
- Plan infrastructure improvements

---

## Appendix: Deployment Commands

### Infrastructure Deployment
```bash
# Staging
cd infrastructure/staging
cdk deploy --all

# Production (requires approval)
cd infrastructure/production
cdk deploy --all --require-approval
```

### Database Migration
```bash
# Generate migration
npm run migrate:dev -- --name <migration_name>

# Apply to staging
npm run migrate:staging

# Apply to production
npm run migrate:production
```

### Application Deployment
```bash
# Build frontend
cd packages/web-app
npm run build

# Deploy frontend to S3
aws s3 sync build/ s3://demand-letter-app-prod --delete

# Invalidate CloudFront
aws cloudfront create-invalidation --distribution-id <id> --paths "/*"

# Deploy Lambda functions
cd packages/api
npm run deploy:production
```

### Rollback
```bash
# Rollback Lambda to previous version
aws lambda update-function-code \
  --function-name demand-letter-api \
  --s3-bucket lambda-deployments \
  --s3-key artifacts/previous-version.zip

# Rollback database (forward fix preferred)
npm run migrate:rollback
```

---

## Success Criteria for Deployment

The deployment is considered successful when:
- [ ] All smoke tests passing
- [ ] Error rate <0.1% for 24 hours
- [ ] Response times within SLA (5 seconds for API)
- [ ] 10+ successful end-to-end user workflows completed
- [ ] No critical bugs reported
- [ ] All monitoring dashboards show green
- [ ] Security validation passed
- [ ] Stakeholder sign-off received

**Definition of Done:** MVP in production, serving real users, with all P0 requirements functional and monitored.
