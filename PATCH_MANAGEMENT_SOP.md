# Patch Management Operational Playbook (SOP)

**Document Version:** 1.0  
**Effective Date:** February 2026  
**Document Owner:** IT Operations  
**Classification:** Internal - Confidential  

---

## 1. Playbook Overview

### 1.1 Purpose and Scope

This operational playbook provides step-by-step procedures for executing patch deployments across the three-environment system (DEV → TEST → PROD). It translates the Patch Management Strategy into actionable tasks for operations teams.

**This playbook covers:**
- Standard patch deployments through all environments
- Emergency/security patch procedures
- Rollback execution procedures
- Troubleshooting common deployment issues

**This playbook does NOT cover:**
- Strategic patch selection (see Patch Management Strategy)
- Vendor contract negotiations
- Long-term infrastructure planning

### 1.2 Target Audience

| Role | Use This Playbook For |
|------|----------------------|
| IT Operations Engineers | Executing deployments, validations, rollbacks |
| DevOps Engineers | Pipeline execution, automation troubleshooting |
| Help Desk/Support Staff | Understanding deployment status, user communication |
| Test Engineers | TEST environment deployment and validation |
| On-Call Engineers | Emergency response, incident handling |

### 1.3 When to Use This Playbook

**Use this playbook when:**
- Deploying routine monthly patches
- Applying security patches (planned or emergency)
- Executing major version upgrades
- Performing emergency patch response
- Executing rollback procedures
- Validating post-deployment system state

**Reference the Strategy document when:**
- Determining patch classification
- Understanding approval requirements
- Defining test scope
- Creating change requests

---

## 2. Pre-Patch Procedures

### 2.1 Patch Intake and Classification

**Step 1:** Receive patch notification from one of the following sources:
- Vendor security advisory
- Internal security team
- Development team
- Change Management system

**Step 2:** Locate the associated Change Request (CR) in the ticketing system
- If no CR exists, create one following Section 2.2

**Step 3:** Verify patch classification in the CR
- P1-Critical: Active exploitation, CVSS 9.0+
- P2-High: Significant risk, CVSS 7.0-8.9
- P3-Moderate: Moderate impact, CVSS 4.0-6.9
- P4-Low: Minimal impact, non-security

**Step 4:** Document the following in the CR:
- Patch source and vendor reference (KB article, CVE number, version)
- Affected systems and components
- Expected downtime
- Rollback complexity assessment

### 2.2 Change Request Creation

**Step 1:** Create a new Change Request in the ticketing system with the following fields:

| Field | Required Information |
|-------|---------------------|
| Title | [System Name] - [Patch Type] - [Date] |
| Classification | P1/P2/P3/P4 |
| Description | Detailed patch description and purpose |
| Affected Systems | List all impacted systems |
| Change Type | Security/Standard/Infrastructure |
| Implementation Plan | Step-by-step deployment procedure |
| Rollback Plan | Detailed rollback procedure |
| Risk Assessment | Impact and risk analysis |
| Test Plan | Validation and test approach |

**Step 2:** Attach supporting documentation:
- Vendor release notes
- Security impact assessment (for P1/P2)
- Rollback procedure document
- Test case specifications

**Step 3:** Submit CR for approval based on classification:
- DEV: Development Lead approval (same day)
- TEST: Test Manager approval (24-hour lead time)
- PROD: CAB approval (5 business days standard, 4-24 hours emergency)

### 2.3 Environment Readiness Checks

**Execute the following checks before ANY deployment:**

**Step 1:** Verify environment availability
```bash
# Check if target environment is accessible
ping [environment-hostname]
ssh [user]@[environment-hostname] "hostname"
```

**Step 2:** Verify sufficient disk space
```bash
df -h
# Ensure minimum 20% free space on all volumes
```

**Step 3:** Verify network connectivity
```bash
# Test connectivity to dependent services
telnet [service-hostname] [port]
curl -I https://[service-url]/health
```

**Step 4:** Verify no conflicting changes
```bash
# Check deployment calendar for overlapping changes
# Verify no maintenance windows conflict
```

**Step 5:** Confirm all prerequisites are met
- Previous patch promotion completed
- Dependencies available
- Required access credentials valid

### 2.4 Backup Verification

**Step 1:** Identify backup type required:
- Database: Full backup before schema/data changes
- Application: Snapshot or version backup
- Configuration: Configuration file backup

**Step 2:** Verify backup completion
```bash
# For database backups
psql -U [user] -c "SELECT pg_start_backup('pre-patch-backup');"
# Verify backup file exists
ls -la /backup/[latest-backup-file]

# For application backups
ls -la /opt/[application]/backups/
```

**Step 3:** Test backup restoration (PROD only, at discretion)
```bash
# Restore to test environment to verify backup integrity
# Document restore time for RTO planning
```

**Step 4:** Document backup verification in CR
- Backup completed: [Timestamp]
- Backup location: [Path]
- Backup size: [Size]
- Restore test: [Pass/Fail/N/A]

### 2.5 Notification Requirements

**Step 1:** Send pre-deployment notification based on environment:

| Environment | Notify | Timing | Channel |
|-------------|--------|--------|---------|
| DEV | Development Team | Immediate | Teams/Email |
| TEST | QA Team, Business Analysts | 24 hours before | Teams/Email |
| PROD | All stakeholders | 48-72 hours before | Email |
| Emergency | Security Team, CAB Chair | Immediate | Phone/Teams |

**Step 2:** Include in notification:
- Deployment date and time
- Expected duration
- Affected systems
- Rollback plan availability
- Contact information

---

## 3. Deployment Procedures by Environment

### 3.1 DEV Deployment Steps

**Prerequisites:**
- [ ] Change request created and approved by Development Lead
- [ ] Patch package validated by development team
- [ ] Unit tests executed (≥90% coverage)
- [ ] Backup verified
- [ ] Rollback procedure documented

**Step 1:** Prepare the deployment environment
```bash
# Navigate to deployment directory
cd /opt/deployment/[application]

# Create deployment working directory
mkdir -p patch-$(date +%Y%m%d)

# Download patch package
wget -O patch-$(date +%Y%m%d)/patch.tar.gz [patch-source-url]
```

**Step 2:** Stop dependent services
```bash
# Stop application services
systemctl stop [application-service]

# Verify services stopped
systemctl status [application-service] | grep "Active:"
```

**Step 3:** Execute pre-deployment backup
```bash
# Backup current application directory
cp -r /opt/[application] /opt/[application]-backup-$(date +%Y%m%d)

# Backup configuration files
cp -r /etc/[application] /etc/[application]-backup-$(date +%Y%m%d)
```

**Step 4:** Apply the patch
```bash
# Extract patch package
tar -xzf patch-$(date +%Y%m%d)/patch.tar.gz -C /opt/[application]/

# Apply database migrations (if applicable)
psql -U [db-user] -d [database-name] -f /opt/[application]/migrations/$(date +%Y%m%d).sql

# Set correct permissions
chown -R [application-user]:[application-group] /opt/[application]
```

**Step 5:** Start services
```bash
# Start application services
systemctl start [application-service]

# Verify services started
systemctl status [application-service] | grep "Active:"
sleep 10
```

**Step 6:** Execute smoke tests
```bash
# Run automated smoke tests
cd /opt/[application]/tests
./smoke-test.sh

# Verify health endpoints
curl -f http://localhost:[port]/health || exit 1
curl -f http://localhost:[port]/ready || exit 1
```

**Step 7:** Verify basic functionality
```bash
# Test critical business functions
curl -X POST http://localhost:[port]/api/[critical-endpoint] -H "Content-Type: application/json" -d '{}'

# Check application logs for errors
tail -n 100 /var/log/[application]/application.log | grep -i error
```

**Step 8:** Document deployment results
- Deployment completed: [Timestamp]
- Smoke tests: [Pass/Fail]
- Errors found: [None/Describe]
- Rollback required: [Yes/No]

**Exit Criteria:**
- Application starts successfully
- Health check endpoints respond
- Smoke tests pass (100%)
- No critical errors in logs
- Basic functionality verified

### 3.2 TEST Deployment Steps

**Prerequisites:**
- [ ] DEV deployment successful with sign-off
- [ ] Test cases updated for new functionality
- [ ] Test data prepared and available
- [ ] Test environment availability confirmed
- [ ] Regression test suite ready
- [ ] Change request approved by Test Manager

**Step 1:** Verify DEV success before promoting to TEST
```bash
# Confirm DEV deployment completed
# Review DEV smoke test results
# Review DEV logs for any issues
```

**Step 2:** Prepare the TEST deployment environment
```bash
# Navigate to deployment directory
cd /opt/deployment/[application]

# Create deployment working directory
mkdir -p patch-$(date +%Y%m%d)

# Copy patch package from validated source
cp /source/patch-$(date +%Y%m%d)/patch.tar.gz /opt/deployment/[application]/patch-$(date +%Y%m%d)/
```

**Step 3:** Stop TEST environment services
```bash
# Stop application services
systemctl stop [application-service]

# Verify all connections closed
ss -tuln | grep [port]
```

**Step 4:** Execute pre-deployment backup
```bash
# Backup current TEST application
cp -r /opt/[application] /opt/[application]-backup-$(date +%Y%m%d)

# Backup TEST database
pg_dump -U [db-user] [database-name] > /backup/test-db-$(date +%Y%m%d).sql

# Document backup for rollback reference
```

**Step 5:** Apply the patch
```bash
# Extract patch package
tar -xzf patch-$(date +%Y%m%d)/patch.tar.gz -C /opt/[application]/

# Apply database migrations
psql -U [db-user] -d [database-name] -f /opt/[application]/migrations/$(date +%Y%m%d).sql

# Verify migration completion
psql -U [db-user] -d [database-name] -c "SELECT version FROM schema_migrations ORDER BY applied_at DESC LIMIT 1;"
```

**Step 6:** Start services and verify
```bash
# Start application services
systemctl start [application-service]

# Wait for startup
sleep 30

# Verify services running
systemctl status [application-service]
```

**Step 7:** Execute smoke tests (100% pass required)
```bash
# Run smoke test suite
cd /opt/[application]/tests
./smoke-test.sh

# Verify all health endpoints
for endpoint in health ready metrics; do
  curl -f http://localhost:[port]/$endpoint || { echo "FAILED: $endpoint"; exit 1; }
done
```

**Step 8:** Execute regression test suite
```bash
# Run automated regression tests
cd /opt/[application]/tests
./run-regression.sh --environment=TEST

# Target: ≥95% pass rate required
# Review test results
cat /opt/[application]/tests/results/regression-report.html
```

**Step 9:** Notify QA team for manual testing
```
Subject: TEST Deployment Complete - [System Name] - [Date]

QA Team,
The patch has been deployed to TEST environment.

Deployment Details:
- Environment: TEST
- Patch: [Patch ID/Version]
- Time Completed: [Timestamp]

Next Steps:
- Execute manual test cases
- Complete UAT testing
- Report any defects

Regression Results: [Pass Rate]%
Smoke Tests: [Pass/Fail]
```

**Step 10:** Obtain UAT sign-off
- Business process owners complete testing
- Sign-off documented in change request
- Any blocking defects resolved

**Exit Criteria:**
- Smoke tests pass (100%)
- Regression pass rate ≥95%
- All blocking/critical defects resolved
- UAT sign-off obtained
- Performance within baseline
- Security scan passed

### 3.3 PROD Deployment Steps

**Prerequisites:**
- [ ] TEST execution results approved by Test Manager
- [ ] Regression test pass rate ≥95%
- [ ] UAT sign-off obtained
- [ ] CAB approval obtained
- [ ] Rollback plan documented and tested
- [ ] Communication plan initiated
- [ ] Maintenance window confirmed

**Step 1:** Final pre-deployment verification
```bash
# Verify backup completion
ls -la /backup/prod-db-$(date +%Y%m%d).sql

# Verify patch package integrity
md5sum /source/patch-$(date +%Y%m%d)/patch.tar.gz

# Verify all pre-deployment checklist items complete
```

**Step 2:** Send deployment start notification
```
Subject: [DEPLOYMENT START] [System Name] - PROD - [Date/Time]

Deployment has commenced.

Timeline:
- [Time] - Deployment start
- [Time] - Expected completion
- [Time] - Post-deployment validation start

Monitoring: [Monitoring dashboard URL]
Support Channel: [Teams Channel / Slack]
```

**Step 3:** Enter maintenance window
```bash
# Enable maintenance mode (if applicable)
# For load balancer changes, drain connections
```

**Step 4:** Stop production services
```bash
# For rolling deployments - stop one node at a time
# For monolithic deployments - stop all services

# Stop primary application
systemctl stop [application-service]

# Verify no active connections
ss -tuln | grep [port] | wc -l
```

**Step 5:** Execute production backup
```bash
# Application backup
cp -r /opt/[application] /opt/[application]-backup-prod-$(date +%Y%m%d)

# Database backup with point-in-time capability
pg_dump -U [db-user] -d [database-name] -F c -b -v -f /backup/prod-db-$(date +%Y%m%d).dump

# Verify backup completion
ls -lh /backup/prod-db-$(date +%Y%m%d).dump
```

**Step 6:** Apply the patch
```bash
# Extract patch package
tar -xzf /source/patch-$(date +%Y%m%d)/patch.tar.gz -C /opt/[application]/

# Apply database migrations with transaction
psql -U [db-user] -d [database-name] << EOF
BEGIN;
\i /opt/[application]/migrations/$(date +%Y%m%d).sql
-- Verify migration success
SELECT COUNT(*) FROM schema_migrations WHERE version = '$(date +%Y%m%d)';
COMMIT;
EOF
```

**Step 7:** Verify patch application
```bash
# Verify files updated
ls -la /opt/[application]/[key-files]

# Verify database migrations
psql -U [db-user] -d [database-name] -c "SELECT * FROM schema_migrations ORDER BY applied_at DESC LIMIT 5;"

# Check for any migration errors in logs
tail -50 /var/log/[application]/migration.log | grep -i error
```

**Step 8:** Start production services
```bash
# Start application services
systemctl start [application-service]

# Wait for startup
sleep 30

# Verify services running
systemctl status [application-service]
```

**Step 9:** Execute post-deployment smoke tests
```bash
# Run production smoke tests
cd /opt/[application]/tests
./smoke-test.sh --environment=PROD

# Test critical business transactions
curl -X GET http://localhost:[port]/api/[critical-endpoint]

# Verify all nodes healthy
for node in node1 node2 node3; do
  curl -f http://$node:[port]/health || { echo "FAILED: $node"; exit 1; }
done
```

**Step 10:** Verify monitoring and alerting
```bash
# Verify monitoring agents reporting
curl -s http://localhost:[metrics-port]/metrics | head -20

# Verify alerting enabled
grep -i "alert" /etc/[monitoring-agent]/config.yaml

# Check for immediate alerts
curl -s [monitoring-api]/alerts?severity=critical
```

**Step 11:** Execute data integrity checks
```bash
# Verify database consistency
psql -U [db-user] -d [database-name] -c "SELECT schemaname, tablename, n_live_tup FROM pg_stat_user_tables ORDER BY n_live_tup DESC LIMIT 10;"

# Verify record counts match expected
psql -U [db-user] -d [database-name] -c "SELECT COUNT(*) FROM [critical-table];"

# Check for constraint violations
psql -U [db-user] -d [database-name] -c "SELECT * FROM pg_constraint WHERE NOT convalidated;"
```

**Step 12:** Send deployment completion notification
```
Subject: [DEPLOYMENT COMPLETE] [System Name] - PROD - [Status]

DEPLOYMENT STATUS: [SUCCESS / SUCCESS WITH ISSUES / ROLLED BACK]

Details:
- Completed: [Timestamp]
- Duration: [Total Time]
- Smoke Tests: [Pass/Fail]
- Errors: [None/Describe]

Post-Deployment Actions:
- 72-hour enhanced monitoring active
- Report any issues to [Support Channel]

Rollback Contact: [Name/Phone]
```

**Exit Criteria:**
- Post-deployment smoke tests pass
- Monitoring alerts confirmed operational
- Application health checks green
- Database integrity confirmed
- No critical errors in logs (4-hour check)
- Performance within baseline
- Business stakeholders notified

---

## 4. Post-Patch Procedures

### 4.1 Validation and Smoke Testing

**Step 1:** Execute automated smoke tests
```bash
# Run full smoke test suite
cd /opt/[application]/tests
./smoke-test.sh --environment=[ENV] --verbose

# Expected: 100% pass rate
```

**Step 2:** Verify health endpoints
```bash
# Check all health endpoints
for endpoint in health ready metrics; do
  status=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:[port]/$endpoint)
  echo "$endpoint: $status"
  if [ "$status" != "200" ]; then
    echo "ALERT: $endpoint returning $status"
  fi
done
```

**Step 3:** Execute business critical transaction tests
```bash
# Test critical user workflows
# Document test results
```

**Step 4:** Review application logs
```bash
# Check for errors in application logs
tail -500 /var/log/[application]/application.log | grep -iE "(error|exception|fatal)"

# Check for warnings
tail -500 /var/log/[application]/application.log | grep -iW "warning" | tail -20
```

### 4.2 Monitoring Verification

**Step 1:** Verify monitoring stack operational
```bash
# Verify monitoring agent running
systemctl status [monitoring-agent]

# Verify metrics being collected
curl -s http://localhost:[port]/metrics | wc -l
```

**Step 2:** Verify alerting configured
```bash
# Check critical alert rules exist
grep -i "critical" /etc/[monitoring-agent]/alerts/*.yaml

# Verify alert notifications working
# Send test alert if needed
```

**Step 3:** Document baseline metrics
```bash
# Record key metrics for comparison
curl -s http://localhost:[port]/metrics | grep -E "(cpu_memory|request_latency|error_rate)" > /tmp/baseline-$(date +%Y%m%d).txt

# Compare with previous baseline
diff /tmp/baseline-$(date +%Y%m%d).txt /tmp/baseline-previous.txt
```

**Step 4:** Configure enhanced monitoring (PROD only)
- Set increased polling frequency
- Enable additional metric collection
- Configure automated alerting for deviations
- Document monitoring duration (minimum 72 hours)

### 4.3 Documentation and Sign-Off

**Step 1:** Update change request with deployment results
```
Deployment Results:
- Start Time: [Timestamp]
- End Time: [Timestamp]
- Duration: [Time]
- Status: [Success/Failed/Rolled Back]
- Smoke Tests: [Pass/Fail]
- Errors: [None/Describe]
```

**Step 2:** Obtain required sign-offs

| Environment | Sign-Off Required | Authority |
|-------------|-------------------|-----------|
| DEV → TEST | Development Lead | Development Lead |
| TEST → PROD | Test Manager, Business Analyst | Test Manager |
| PROD | CAB Chair | CAB |

**Step 3:** Archive documentation
- Deployment logs
- Test results
- Monitoring data
- Communication records

### 4.4 Communication Closure

**Step 1:** Send post-deployment confirmation
```
Subject: [POST-DEPLOYMENT CONFIRMATION] [System Name] - [Date]

DEPLOYMENT SUMMARY
==================
System: [Name]
Environment: [DEV/TEST/PROD]
Completed: [Date/Time]
Status: [SUCCESS]

VALIDATION RESULTS
===================
Smoke Tests: [Pass]
Regression: [Pass Rate]%
Performance: [Within Baseline]
Data Integrity: [Verified]

NEXT STEPS
==========
- [72-hour monitoring active]
- [UAT sign-off required]
- [Post-implementation review scheduled]

Post-Implementation Review: [Date/Time]
```

**Step 2:** Close stakeholder communication loops
- Confirm all stakeholders received updates
- Document any outstanding items
- Schedule post-implementation review (PROD only)

**Step 3:** Schedule follow-up activities
- Post-implementation review (within 5 business days for PROD)
- 72-hour monitoring check
- User issue triage check

---

## 5. Emergency Patch Procedures

### 5.1 Trigger Criteria for Emergency Classification

**An issue qualifies for emergency patch procedures when:**

| Criterion | Threshold | Example |
|-----------|-----------|---------|
| Active Exploitation | Confirmed in-the-wild exploit | CVE with public PoC |
| CVSS Score | ≥9.0 (Critical) | Remote code execution |
| Service Availability | Immediate risk | Ransomware indicators |
| Regulatory Directive | Required immediate action | Compliance enforcement |
| Business Impact | Complete system outage potential | Critical system down |

**Step 1:** Document emergency justification
- Describe the security incident or vulnerability
- Document CVSS score and source
- Explain why normal process cannot be followed

**Step 2:** Obtain emergency approval (see Section 5.2)

### 5.2 Accelerated Approval Process

**Step 1:** Contact emergency approvers in parallel:
- CAB Chair: [Phone] [Email]
- Security Lead: [Phone] [Email]
- IT Operations Manager: [Phone] [Email]

**Step 2:** Provide emergency approval request:
```
EMERGENCY PATCH APPROVAL REQUEST
==================================
Incident: [Brief description]
Classification: P1-Critical
CVSS Score: [Score]
Vendor Reference: [CVE/KB]

Risk Assessment:
- Impact if not deployed: [Description]
- Risk if deployed: [Description]

Recommended Action: [Deploy/Defer]

Requestor: [Name/Team]
Time: [Timestamp]
```

**Step 3:** Obtain verbal approval (document in CR)
- Emergency approvers may grant verbal approval
- Document approver name, time, and conditions
- Obtain written confirmation within 24 hours

**Step 4:** Document expedited approval in CR
- Approval type: Emergency (Expedited)
- Approvers: [Names]
- Approval time: [Timestamp]
- Any conditions or limitations

### 5.3 Compressed Deployment Steps

**For emergency patches, follow standard deployment with these compressions:**

**Step 1:** Skip TEST environment (compressed path)
- Deploy DEV → PROD (skip TEST)
- Document risk acceptance

**Step 2:** Reduced testing requirements
- Execute only: Smoke test + targeted tests
- Skip: Full regression, UAT
- Accept: Enhanced post-deployment monitoring

**Step 3:** Accelerated timeline
| Phase | Standard | Emergency |
|-------|----------|-----------|
| Approval | 5 days | 4-24 hours |
| DEV Deployment | 1 day | Same day |
| PROD Deployment | 1-2 days | 4-12 hours |
| Post-Deploy Validation | Standard | 24-48 hours in TEST |

**Step 4:** Deploy with enhanced rollback readiness
```bash
# Ensure automatic rollback is configured
# Verify rollback can execute within 15 minutes
# Test rollback procedure before deployment
```

**Step 5:** Enhanced post-deployment monitoring
- Continuous monitoring for 72 hours minimum
- Dedicated on-call resource assigned
- Automated alerting on any anomaly
- Hourly check-ins for first 8 hours

### 5.4 Post-Incident Review Requirements

**Step 1:** Schedule Post-Incident Review (PIR)
- Within 5 business days of deployment
- Include: Security, Operations, Development, CAB

**Step 2:** Document PIR agenda:
- Timeline of emergency response
- Effectiveness of emergency procedures
- Any process improvements identified
- Lessons learned

**Step 3:** Complete PIR documentation
- Root cause of security issue
- Response effectiveness
- Process deviations taken
- Recommendations for future improvements
- Update procedures as needed

**Step 4:** Submit PIR to CAB for review
- Document all findings
- Archive for compliance

---

## 6. Rollback Procedures

### 6.1 Decision Triggers (When to Rollback)

**Execute rollback IMMEDIATELY when:**

| Trigger | Threshold | Action |
|---------|-----------|--------|
| Service Availability | <99.5% | Immediate rollback |
| Critical Defects | ≥5% users affected | Immediate rollback |
| Data Integrity | Any confirmed issue | Immediate rollback |
| Security Vulnerability | CVSS ≥7.0 discovered | Fast-track rollback |
| Smoke Test Failure | Any failure in PROD | Evaluate + decide |
| Performance Degradation | >20% slower | Evaluate impact |

**Step 1:** Assess severity
- Is service available?
- What percentage of users affected?
- Is data integrity maintained?
- What is the business impact?

**Step 2:** Determine rollback vs. forward-fix
- Can the issue be fixed quickly (<30 min)?
- Is rollback less risky than forward-fix?
- Do we have a tested rollback procedure?

**Step 3:** Obtain authorization if not emergency
- DEV: Lead Developer approval
- TEST: Test Manager + Development Lead (joint)
- PROD: CAB Chair + IT Operations Manager (joint)

### 6.2 Step-by-Step Rollback Instructions

#### DEV Environment Rollback

**Step 1:** Stop application services
```bash
systemctl stop [application-service]
```

**Step 2:** Restore application files
```bash
# Remove current version
rm -rf /opt/[application]

# Restore from backup
cp -r /opt/[application]-backup-[date] /opt/[application]

# Set permissions
chown -R [application-user]:[application-group] /opt/[application]
```

**Step 3:** Restore database (if applicable)
```bash
psql -U [db-user] -d [database-name] -c "DROP SCHEMA public CASCADE;"
psql -U [db-user] -d [database-name] -c "CREATE SCHEMA public;"
psql -U [db-user] -d [database-name] < /backup/dev-db-[date].sql
```

**Step 4:** Start services
```bash
systemctl start [application-service]
```

**Step 5:** Verify rollback
```bash
# Run smoke tests
cd /opt/[application]/tests
./smoke-test.sh

# Verify health
curl -f http://localhost:[port]/health
```

#### TEST Environment Rollback

**Step 1:** Coordinate with QA team
- Notify of rollback initiation
- Suspend ongoing tests

**Step 2:** Stop TEST services
```bash
systemctl stop [application-service]
```

**Step 3:** Restore TEST environment
```bash
# Restore application
cp -r /opt/[application]-backup-[date] /opt/[application]

# Restore TEST database
pg_restore -U [db-user] -d [database-name] -c /backup/test-db-[date].dump
```

**Step 4:** Verify rollback
```bash
# Run smoke tests
./smoke-test.sh --environment=TEST

# Verify database integrity
psql -U [db-user] -d [database-name] -c "SELECT COUNT(*) FROM [critical-table];"
```

**Step 5:** Notify QA of rollback completion
- Document rollback in CR
- Reschedule testing

#### PROD Environment Rollback

**Step 1:** Confirm rollback decision
- Obtain required approvals (CAB Chair + IT Ops Manager)
- Document rollback trigger in CR

**Step 2:** Send rollback notification
```
Subject: [EMERGENCY] ROLLBACK INITIATED - [System Name] - [Time]

ROLLBACK TRIGGER: [Description]
APPROVED BY: [Names]
TARGET: Complete rollback to previous version

Timeline:
- [Time] - Rollback initiated
- [Time] - Expected completion

Support Channel: [Teams/Slack]
```

**Step 3:** Enter maintenance mode
```bash
# Enable maintenance page
# Drain active connections
```

**Step 4:** Stop production services
```bash
# Stop all application services
systemctl stop [application-service]

# Verify connections closed
ss -tuln | grep [port]
```

**Step 5:** Restore production backup
```bash
# Restore application files
rm -rf /opt/[application]
cp -r /opt/[application]-backup-prod-[date] /opt/[application]

# Restore database
pg_restore -U [db-user] -d [database-name] -c --exit-on-error /backup/prod-db-[date].dump

# Verify database restored
psql -U [db-user] -d [database-name] -c "SELECT version FROM schema_migrations ORDER BY applied_at DESC LIMIT 1;"
```

**Step 6:** Start production services
```bash
# Start application
systemctl start [application-service]

# Verify startup
sleep 30
systemctl status [application-service]
```

**Step 7:** Verify rollback success
```bash
# Run smoke tests
cd /opt/[application]/tests
./smoke-test.sh --environment=PROD

# Verify data integrity
psql -U [db-user] -d [database-name] -c "SELECT COUNT(*) FROM [critical-table];"

# Verify service health
curl -f http://localhost:[port]/health
```

**Step 8:** Send rollback completion notification
```
Subject: ROLLBACK COMPLETE - [System Name] - [Time]

ROLLBACK STATUS: COMPLETED

Details:
- Completed: [Timestamp]
- Duration: [Time]
- Status: [Success/Failed]

Current State:
- Application: [Version]
- Database: [Version]
- Smoke Tests: [Pass/Fail]

Next Steps:
- [Investigate root cause]
- [Schedule post-incident review]
```

### 6.3 Verification After Rollback

**Step 1:** Verify application functionality
```bash
# Test all critical endpoints
for endpoint in health ready api/users api/orders; do
  curl -f http://localhost:[port]/$endpoint || { echo "FAILED: $endpoint"; exit 1; }
done
```

**Step 2:** Verify data integrity
```bash
# Compare record counts
psql -U [db-user] -d [database-name] -c "SELECT 'table1' as tbl, COUNT(*) FROM table1 UNION ALL SELECT 'table2', COUNT(*) FROM table2;"

# Verify referential integrity
psql -U [db-user] -d [database-name] -c "SELECT * FROM pg_constraint WHERE NOT convalidated;"
```

**Step 3:** Verify monitoring operational
```bash
# Verify metrics collection
curl -s http://localhost:[port]/metrics | head -10

# Verify alerts firing
curl -s [monitoring-api]/alerts | jq '.'
```

**Step 4:** Document rollback verification
- All smoke tests pass
- Data integrity confirmed
- Monitoring operational
- Service availability restored

### 6.4 Communication During Rollback

**Step 1:** Initial notification (within 5 minutes)
- Notify: IT Operations, Development Lead, CAB Chair
- Channel: Phone/Teams (immediate)
- Content: Rollback initiated, trigger, ETA

**Step 2:** Progress updates (every 15 minutes)
- Channel: Teams/Slack
- Content: Current step, status, issues

**Step 3:** Completion notification
- Notify: All stakeholders
- Channel: Email
- Content: Rollback complete, current state, next steps

**Step 4:** Post-incident communication
- Schedule PIR within 5 business days
- Document lessons learned
- Update procedures as needed

---

## 7. Troubleshooting Common Issues

### 7.1 Patch Deployment Failures

**Issue: Deployment script fails**

| Step | Action |
|------|--------|
| 1 | Check error message in output |
| 2 | Review deployment logs: `tail -100 /var/log/[deployment].log` |
| 3 | Verify file permissions: `ls -la /opt/[application]` |
| 4 | Verify disk space: `df -h` |
| 5 | Check dependencies: `ldd /opt/[application]/bin/[binary]` |
| 6 | Retry deployment if transient error |

**Issue: Service won't start after patch**

| Step | Action |
|------|--------|
| 1 | Check service status: `systemctl status [service]` |
| 2 | Review startup logs: `journalctl -u [service] -n 100` |
| 3 | Verify configuration syntax: `nginx -t` |
| 4 | Check port conflicts: `ss -tuln \| grep [port]` |
| 5 | Verify file permissions |
| 6 | Check dependencies: `ldd` on libraries |
| 7 | Rollback if unresolvable |

**Issue: Database migration fails**

| Step | Action |
|------|--------|
| 1 | Check migration error: `psql -U [user] -d [db] -e -f migration.sql` |
| 2 | Verify database connectivity |
| 3 | Check for syntax errors in migration |
| 4 | Verify required tables exist |
| 5 | Check data type compatibility |
| 6 | Rollback database if migration partially applied |

### 7.2 Application Compatibility Issues

**Issue: Application errors after patch**

| Step | Action |
|------|--------|
| 1 | Check application logs: `tail -200 /var/log/[app]/error.log` |
| 2 | Identify error pattern (API, database, configuration) |
| 3 | Check version compatibility: vendor documentation |
| 4 | Verify configuration files updated |
| 5 | Test with previous configuration backup |
| 6 | Escalate to development if code issue |

**Issue: Dependency conflicts**

| Step | Action |
|------|--------|
| 1 | Check dependency versions: `pip list` / `npm list` |
| 2 | Verify library compatibility |
| 3 | Check for missing dependencies |
| 4 | Update dependency versions if supported |
| 5 | Escalate to development for resolution |

### 7.3 Performance Degradation

**Issue: Application slow after patch**

| Step | Action |
|------|--------|
| 1 | Check resource utilization: `top`, `free -h`, `iostat` |
| 2 | Compare with baseline metrics |
| 3 | Check database query performance |
| 4 | Verify index usage |
| 5 | Check for new bottlenecks |
| 6 | Enable slow query logging if database |
| 7 | Rollback if >20% degradation |

**Issue: High memory usage**

| Step | Action |
|------|--------|
| 1 | Check memory usage: `free -h` |
| 2 | Identify memory-intensive processes: `ps aux --sort=-%mem \| head` |
| 3 | Check for memory leaks in logs |
| 4 | Restart affected services |
| 5 | Rollback if persistent |

### 7.4 Data Integrity Concerns

**Issue: Data inconsistency after patch**

| Step | Action |
|------|--------|
| 1 | HALT deployment immediately |
| 2 | Do NOT make any changes |
| 3 | Verify data with pre-deployment backup |
| 4 | Compare record counts |
| 5 | Check referential integrity |
| 6 | Execute ROLLBACK immediately |
| 7 | Escalate to DBA Lead |

**Issue: Database connection failures**

| Step | Action |
|------|--------|
| 1 | Check database status: `pg_isready` |
| 2 | Verify connection string |
| 3 | Check max connections: `psql -c "SHOW max_connections;"` |
| 4 | Check database logs |
| 5 | Test connectivity: `psql -U [user] -d [db] -c "SELECT 1;"` |
| 6 | Restart database if needed |

---

## 8. Checklists

### 8.1 Pre-Deployment Checklist

**General Pre-Deployment:**
- [ ] Change request created and approved
- [ ] Patch package downloaded and verified (MD5/SHA256)
- [ ] Rollback plan documented and tested
- [ ] Backup completed and verified
- [ ] Environment readiness checks passed
- [ ] Prerequisites met (dependencies, access)
- [ ] Maintenance window confirmed

**Notification:**
- [ ] Stakeholders notified per schedule
- [ ] On-call resources notified (PROD)
- [ ] Communication channels established

**Environment-Specific:**
- [ ] DEV: Development Lead approval obtained
- [ ] TEST: Test Manager approval obtained
- [ ] PROD: CAB approval obtained

### 8.2 Post-Deployment Checklist

**Validation:**
- [ ] Smoke tests executed (100% pass required)
- [ ] Health endpoints responding
- [ ] Application logs reviewed (no errors)
- [ ] Business critical functions verified

**Monitoring:**
- [ ] Monitoring alerts verified operational
- [ ] Metrics collection confirmed
- [ ] Enhanced monitoring enabled (PROD - 72 hours)

**Documentation:**
- [ ] Deployment results documented in CR
- [ ] Test results archived
- [ ] Sign-offs obtained

**Communication:**
- [ ] Deployment completion notification sent
- [ ] Stakeholders notified
- [ ] Next steps communicated

### 8.3 Rollback Checklist

**Decision:**
- [ ] Rollback trigger identified
- [ ] Authorization obtained (if required)
- [ ] Impact assessed

**Execution:**
- [ ] Services stopped
- [ ] Application files restored
- [ ] Database restored
- [ ] Services restarted
- [ ] Rollback verified

**Communication:**
- [ ] Rollback initiation notification sent
- [ ] Progress updates sent
- [ ] Completion notification sent

**Post-Rollback:**
- [ ] Smoke tests passed
- [ ] Data integrity verified
- [ ] Monitoring operational
- [ ] Root cause investigation initiated
- [ ] Post-incident review scheduled

### 8.4 Emergency Response Checklist

**Incident Detection:**
- [ ] Issue identified and severity assessed
- [ ] Emergency classification confirmed

**Activation:**
- [ ] Emergency approvers contacted
- [ ] Verbal approval obtained
- [ ] Emergency CR created/updated

**Execution:**
- [ ] Deployment executed (compressed path)
- [ ] Enhanced monitoring enabled
- [ ] 72-hour monitoring active

**Follow-Up:**
- [ ] Stakeholders notified
- [ ] Post-incident review scheduled (within 5 days)
- [ ] Documentation complete

---

## 9. Contacts and Escalation

### 9.1 On-Call Contact Matrix

| Role | Primary Contact | Backup Contact | Hours |
|------|-----------------|-----------------|-------|
| IT Operations On-Call | [Phone] | [Phone] | 24/7 |
| Development On-Call | [Phone] | [Phone] | 24/7 |
| DBA On-Call | [Phone] | [Phone] | 24/7 |
| Security On-Call | [Phone] | [Phone] | 24/7 |
| Network On-Call | [Phone] | [Phone] | 24/7 |

### 9.2 Escalation Paths

**Standard Issues (DEV/TEST):**

| Level | Contact | Response Time |
|-------|---------|---------------|
| Level 1 | On-Call Engineer | 15 minutes |
| Level 2 | Team Lead | 30 minutes |
| Level 3 | Manager | 1 hour |

**Production Issues:**

| Level | Contact | Response Time |
|-------|---------|---------------|
| Level 1 | On-Call Engineer | 15 minutes |
| Level 2 | IT Operations Manager | 30 minutes |
| Level 3 | CAB Chair | 1 hour |
| Level 4 | IT Director | 2 hours |

**Security Issues:**

| Level | Contact | Response Time |
|-------|---------|---------------|
| Level 1 | Security On-Call | Immediate |
| Level 2 | Security Lead | 15 minutes |
| Level 3 | CISO | 30 minutes |

### 9.3 Vendor Contacts

| Vendor | Support Contact | Phone | Email | Account # |
|--------|-----------------|-------|-------|-----------|
| [Database Vendor] | [Contact Name] | [Phone] | [Email] | [Account] |
| [Application Vendor] | [Contact Name] | [Phone] | [Email] | [Account] |
| [Infrastructure Vendor] | [Contact Name] | [Phone] | [Email] | [Account] |
| [Monitoring Vendor] | [Contact Name] | [Phone] | [Email] | [Account] |

### 9.4 Internal Contacts

| Team | Email | Teams Channel |
|------|-------|---------------|
| IT Operations | it-ops@[company].com | #it-operations |
| Development | dev-team@[company].com | #development |
| QA Team | qa@[company].com | #qa-team |
| Security | security@[company].com | #security-incidents |
| Change Management | cab@[company].com | #change-management |

---

## Appendix A: Quick Reference Commands

### Deployment Commands
```bash
# Stop service
systemctl stop [service]

# Start service
systemctl start [service]

# Check service status
systemctl status [service]

# View logs
tail -f /var/log/[application]/application.log

# Run smoke tests
cd /opt/[application]/tests && ./smoke-test.sh

# Health check
curl http://localhost:[port]/health
```

### Database Commands
```bash
# Backup database
pg_dump -U [user] -d [database] > /backup/db-[date].sql

# Restore database
psql -U [user] -d [database] < /backup/db-[date].sql

# Check database status
pg_isready -h [host] -p [port]

# Verify database integrity
psql -U [user] -d [database] -c "SELECT * FROM pg_database WHERE datname = '[database]';"
```

### Monitoring Commands
```bash
# Check metrics
curl http://localhost:[port]/metrics

# Check alerts
curl [monitoring-api]/alerts

# Verify monitoring agent
systemctl status [monitoring-agent]
```

---

## Appendix B: Change Request Template

```
CHANGE REQUEST
==============

CR Number: [Auto-generated]
Title: [System] - [Patch Type] - [Date]

CLASSIFICATION
==============
Type: [Security/Standard/Infrastructure]
Priority: [P1/P2/P3/P4]
CVSS Score: [Score - if applicable]

APPROVALS
=========
DEV Approval: [Name/Date]
TEST Approval: [Name/Date]
PROD Approval: [Name/Date]

DEPLOYMENT DETAILS
==================
Target Environment: [DEV/TEST/PROD]
Scheduled Date: [Date]
Scheduled Time: [Time]
Expected Duration: [Duration]
Downtime Required: [Yes/No]

ROLLBACK PLAN
=============
Rollback Procedure: [Link to procedure]
Rollback Duration: [Time]
Rollback Trigger: [Criteria]

VALIDATION
==========
Smoke Tests: [Pass/Fail]
Regression: [Pass Rate]%
UAT Sign-off: [Name/Date]

POST-DEPLOYMENT
===============
Status: [Success/Failed/Rolled Back]
Completed: [Date/Time]
Issues: [None/Describe]
```

---

**Document Control**

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | February 2026 | IT Operations | Initial release |

---

*This document is controlled. For the latest version, refer to the document management system. All printouts are uncontrolled.*
