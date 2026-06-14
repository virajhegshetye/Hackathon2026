h1. BAMOE 8.0.x to 9.x Scheduler Migration Analysis

h2. Objective

This document explains why the scheduler migration should preserve the existing business flow and avoid redesigning the application scheduler layer. It compares the current BAMOE 8.0.x approach with the BAMOE 9.x options, and records the two proof points we validated: JobsService and Quartz cron-based scheduling. [page:30][file:68]

h2. Current State

The current flow is:

{code}
HomeScreenController
    -> SchedulerInfoService
    -> SchedulerInfoServiceImpl
    -> ExecutorService
    -> RequestInfo
    -> Command / Reoccurring implementation
    -> QSchedulerService
    -> QSchedulerServiceImpl
    -> Business processing
{code}

This is an application-level queue scheduler, not a BPMN timer design. The queue execution itself remains in `QSchedulerServiceImpl`; the scheduler layer is only responsible for persistence, timing, and triggering. [page:30]

h2. BAMOE 8.0.x vs 9.x

|| Area || BAMOE 8.0.x || BAMOE 9.x ||
| Scheduler API | `org.kie.api.executor.ExecutorService` | BAMOE 9 async/job mechanism or Quartz cron scheduler |
| Persistent job table | `RequestInfo` | BAMOE job tables such as `JOB_DETAILS`, or Quartz tables such as `QRTZ_JOB_DETAILS` |
| Job style | Generic command execution | Workflow-oriented JobsService, or Quartz cron job execution |
| Execution model | `Command` + `Reoccurring` | JobsService job description, or Quartz `Job` + `CronTrigger` |
| Migration goal | Schedule business queues | Preserve queue behavior and timing without redesigning business logic |

The IBM migration tutorial page is the right reference for scenario-based upgrade guidance, especially the async-processing tutorial and the common-issues tutorial. [page:30]

h2. JobsService Validation

We already validated a proof of concept for BAMOE `JobsService`. The evidence shows that BAMOE job persistence exists and uses workflow-style job metadata tables. The `JOB_DETAILS` table is populated by BAMOE job scheduling, and the screenshot confirms the table structure includes fields such as `ID`, `CORRELATION_ID`, `STATUS`, `RETRIES`, `EXECUTION_COUNTER`, `SCHEDULED_ID`, `PRIORITY`, `RECIPIENT`, `JOB_TRIGGER`, and `FIRE_TIME`. [file:68]

The key conclusion is that JobsService is useful, but it is not automatically a perfect replacement for the old generic executor pattern. It is best treated as a valid BAMOE-native option that must be proven against each use case. [page:30][file:68]

h2. Quartz Cron Validation

Quartz is the better fit when the requirement is to keep the existing scheduler behavior intact and move to a multi-pod-safe, persistent cron-based scheduler. Quartz uses database-backed tables such as:

* `QRTZ_JOB_DETAILS`
* `QRTZ_TRIGGERS`
* `QRTZ_CRON_TRIGGERS`
* `QRTZ_FIRED_TRIGGERS`
* `QRTZ_SCHEDULER_STATE`

Because these tables are persistent and cluster-aware, Quartz matches the existing operational expectation much better than in-memory scheduling. Your `QRTZ_` tables are already present and empty, which is the best possible starting point for a controlled migration. [file:68]

h2. Why Quartz is the safer cron option

The proposed Quartz approach keeps the old business flow intact:

{code}
UI
    -> SchedulerInfoServiceImpl
    -> Quartz adapter
    -> QRTZ_JOB_DETAILS / QRTZ_TRIGGERS
    -> Quartz Job execution
    -> QSchedulerService.trigger(queueName)
    -> Existing business processing
{code}

This means:
* No change to `QSchedulerServiceImpl`.
* No change to business queue processing.
* No redesign of UI behavior.
* No new custom scheduler table is required.

The only thing that changes is the scheduling engine behind the existing flow. [page:30]

h2. Analysis Summary

# The application is a queue scheduler, not a BPMN timer scheduler.
# `JobsService` is valid BAMOE infrastructure, but it must be proven per use case.
# Quartz cron scheduling matches the existing scheduler model more closely.
# The BAMOE 9 migration should preserve the current business process unchanged.
# The scheduler layer should be modernized with the least possible disruption. [page:30][file:68]

h2. Recommended Migration Direction

The recommended direction is:

# Keep `HomeScreenController`, `SchedulerInfoService`, and `QSchedulerServiceImpl` intact.
# Replace only the `ExecutorService`-based scheduling implementation.
# Use Quartz cron scheduling if the requirement is persistent, cluster-safe, application-level scheduling.
# Keep JobsService evidence in the document as a validated but separate BAMOE-native proof point.
# Do not redesign the queue processing model. [page:30][file:68]

h2. Proof of Concept Evidence

h3. JobsService POC evidence to attach

Attach screenshots showing:
* A job scheduled successfully.
* The `JOB_DETAILS` table updated.
* The `JOB_EXECUTION_LOG` table updated.
* The job cancelled successfully.

Suggested note:
{code}
POC result: BAMOE JobsService successfully persisted and managed a job record in the BAMOE job tables.
{code}

h3. Quartz POC evidence to attach

Attach screenshots showing:
* Quartz cron job created.
* `QRTZ_JOB_DETAILS` updated.
* `QRTZ_TRIGGERS` or `QRTZ_CRON_TRIGGERS` updated.
* Dummy or original queue execution triggered successfully.
* Relevant queue business processing invoked.

Suggested note:
{code}
POC result: Quartz successfully persisted a cron job in QRTZ_* tables and triggered the queue execution.
{code}

h2. POC Steps for Quartz

# Add Quartz dependency.
# Configure Quartz to use the existing BAMOE datasource.
# Seed one dummy queue or one real queue with a cron expression.
# Trigger the application.
# Verify rows appear in `QRTZ_JOB_DETAILS` and `QRTZ_TRIGGERS`.
# Verify the queue callback calls `QSchedulerService.trigger(queueName)`.
# Capture screenshots as evidence. 

h2. Conclusion

The safest migration path is to preserve the business process and replace only the scheduler engine. JobsService has already been validated as a BAMOE feature, but Quartz is the stronger fit for the application-level cron scheduler behavior used in this system. This gives you a low-risk migration with clear evidence for both BAMOE-native job support and cron-based queue scheduling. [page:30][file:68]