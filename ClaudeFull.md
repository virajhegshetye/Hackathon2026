# BAMOE Scheduler Migration — Complete Code Package

**Project:** status-mgmt-api (Barclaycard)
**Migration:** BAMOE 8.0 (jBPM `ExecutorService`) → BAMOE 9.3/9.4 (Quartz Scheduler with `JDBCJobStore`)
**Scope:** Full replacement of the robotic queue scheduler backend. UI, controller, and external behavior remain unchanged.

---

## 1. What This Package Contains

This document is a single-file reference containing every piece of code needed for the scheduler migration, plus the exact steps to apply it to your project.

| # | File | Type |
|---|------|------|
| 1 | `ExecutionResults.java` | New |
| 2 | `QSchedulerService.java` | Modified (import only) |
| 3 | `SchedulerConstants.java` | Modified (additive) |
| 4 | `SchedulerUtils.java` | New |
| 5 | `QuartzConfig.java` | New |
| 6 | `CommandExecutorJob.java` | New |
| 7 | `ExecutorServiceAdapter.java` | New |
| 8 | `SchedulerStartupSeeder.java` | New |
| 9 | `UserTaskInstance.java` | Modified (additive) |
| 10 | `user-tasks-by-name.graphql` | New |
| 11 | `QSchedulerServiceImpl.java` | Modified (full rewrite) |
| 12 | `SchedulerInfoServiceImpl.java` | Modified (targeted changes) |

Files NOT changed and NOT included here (left exactly as-is): `HomeScreenController.java`, `GraphQLRequest.java`, `GraphQLResponse.java`, `GraphQLQueryLoader.java` (only one line added — see Step 6), `ProcessUtilService` / `ProcessApiServiceImpl` (already migrated, reused as-is), `RetriggerOnlineAppsSchedular.java`.

---

## 2. Why This Approach (Quick Recap)

| Option | Verdict | Reason |
|--------|---------|--------|
| Kogito Jobs Service | Rejected | BPMN-process-only design. Cannot execute arbitrary Java classes by name. No equivalent to `getQueuedRequests()` / `cancelRequest()` outside BPMN context. Confirmed via IBM migration docs and Apache KIE Jobs Service wiki. |
| Spring `TaskScheduler` / `@Scheduled` | Rejected | In-memory only — not safe across multiple OpenShift pods. State lost on pod restart. Same class of problem previously seen with BPMN timers. |
| **Quartz Scheduler + JDBCJobStore** | **Selected** | DB-backed (uses existing empty `QRTZ_*` tables in RPAM schema), cluster-safe via pessimistic locking, full cron + one-time trigger support, executes arbitrary Java via `Job` interface, native Spring Boot integration. |

---

## 3. Step-by-Step Migration Instructions

### Step 1 — Add the Quartz dependency

In `build.gradle`, add to the dependencies block:

```groovy
implementation 'org.springframework.boot:spring-boot-starter-quartz'
```

### Step 2 — Confirm Quartz tables exist

All 11 `QRTZ_*` tables must already exist (empty) in the RPAM schema — this was confirmed during investigation, so no DDL is required. If your environment differs, run Quartz's standard `oracle` schema script first.

### Step 3 — Add Quartz + cron properties

Add the full property block from **Section 5 (Configuration Reference)** below to `application-dev.properties` (and other environment property files as needed).

### Step 4 — Create all new files

Create the 7 new Java files and 1 new GraphQL file listed in the table above, using the code in **Section 4**. Preserve the exact package paths shown in each section header.

### Step 5 — Replace the 3 modified Java files

Replace the existing content of:
- `QSchedulerService.java`
- `QSchedulerServiceImpl.java`
- `SchedulerInfoServiceImpl.java`

with the versions in Section 4. Apply the additive changes to `SchedulerConstants.java` and `UserTaskInstance.java` (merge new fields/constants into your existing files rather than overwriting, since these already contain unrelated content from earlier migration work).

### Step 6 — Register the new GraphQL query

In `GraphQLQueryLoader.java`, add one line inside the constructor:

```java
loadQuery("userTasksByName", "graphql/queries/user-tasks-by-name.graphql");
```

### Step 7 — Verify the datasource bean name

Open `QuartzConfig.java` and confirm `@Qualifier("bamoeDataSource")` matches the actual `@Bean` name of your BAMOE Oracle `DataSource` in your project's `DataSourceConfig` class. Update if different.

### Step 8 — Delete the 14 old Command classes

Delete these files entirely — all replaced by `CommandExecutorJob.java`:

```
OlaResponseQServiceImpl.java
NoInterfaceQServiceImpl.java
NTCHoldQServiceImpl.java
NGCBckServiceImpl.java
LogCleanUpServiceImpl.java
CASLSSQServiceImpl.java
NODEQServiceImpl.java
NoUploadQServiceImpl.java
BatchResubmitQServiceImpl.java
LapsedProcessServiceImpl.java
FailedSoftToHardServiceImpl.java
ApplicationExpiryProcessServiceImpl.java
AdvancedLogCleanUpServiceImpl.java
SMSRetryServiceImpl.java
```

### Step 9 — Build and deploy

Build the project and deploy to your test environment. On startup, `SchedulerStartupSeeder` will automatically seed 10 cron jobs into `QRTZ_JOB_DETAILS` / `QRTZ_CRON_TRIGGERS` (idempotent — safe across multiple pod restarts).

### Step 10 — Verify with SQL

Run the verification queries in **Section 6** to confirm jobs were seeded correctly.

### Step 11 — Test the UI end-to-end

- Load the Home Screen scheduler table → confirm `getRoboticQList()` returns all 10 queues with correct next-fire times.
- Submit **ON** with a new date/time for a queue → confirm success message `"<Queue> Has Been Rescheduled At ..."` and that `QRTZ_SIMPLE_TRIGGERS` shows the new time.
- Submit **OFF** for a queue → confirm the trigger is paused/removed and the UI reflects it.
- Let a queue fire naturally at its cron time → confirm tasks are claimed/started/completed correctly and `VEC_CASELOCK` is updated.

### Step 12 — Watch the `getCaseId()` fallback chain

Once real scheduler tasks (`OLARESPONSEQ` etc.) appear in `RPAM.TASKS` after deployment, check which fallback strategy in `getCaseId()` is actually being used (`businessKey`, `externalReferenceId`, `referenceName`, or the DB lookup). Simplify the method once the reliable field is confirmed.

---

## 4. Full Source Code

### `com/barclays/api/status/mgmt/schedular/model/ExecutionResults.java`

**Status:** NEW  
**Purpose:** Replaces org.kie.api.executor.ExecutionResults.

```java
package com.barclays.api.status.mgmt.schedular.model;

import java.util.HashMap;
import java.util.Map;

/**
 * Replaces org.kie.api.executor.ExecutionResults which was removed in BAMOE 9.x.
 * Mirrors the exact API surface used in QSchedulerServiceImpl and QSchedulerService.
 *
 * OLD: import org.kie.api.executor.ExecutionResults;
 * NEW: import com.barclays.api.status.mgmt.schedular.model.ExecutionResults;
 */
public class ExecutionResults {

    private Map<String, Object> data = new HashMap<>();

    public void setData(Map<String, Object> data) {
        this.data = data;
    }

    public Map<String, Object> getData() {
        return data;
    }

    /**
     * Get individual value by key.
     * Matches old ExecutionResults.getData(String identifier)
     */
    public Object getData(String identifier) {
        return data.get(identifier);
    }

    /**
     * Set individual value by key.
     * Matches old ExecutionResults.setData(String identifier, Object value)
     */
    public void setData(String identifier, Object value) {
        this.data.put(identifier, value);
    }

    @Override
    public String toString() {
        return "ExecutionResults{data=" + data + "}";
    }
}

```

### `com/barclays/api/status/mgmt/schedular/service/QSchedulerService.java`

**Status:** MODIFIED  
**Purpose:** Interface - import-only change.

```java
package com.barclays.api.status.mgmt.schedular.service;

// CHANGE: Remove org.kie.api.executor.ExecutionResults
// import org.kie.api.executor.ExecutionResults;  ← DELETE THIS

// CHANGE: Use local ExecutionResults replacement
import com.barclays.api.status.mgmt.schedular.model.ExecutionResults;

/**
 * MIGRATION NOTE:
 * Only import changed. org.kie.api.executor.ExecutionResults removed in BAMOE 9.x.
 * Replaced with local com.barclays.api.status.mgmt.schedular.model.ExecutionResults.
 * All method signatures unchanged.
 */
public interface QSchedulerService {

    ExecutionResults trigger(String queueName);

    ExecutionResults triggerPumaRefStatus();

    boolean populateCaseIdInPurgeTable();

    ExecutionResults triggerLapsedProcess();

    ExecutionResults triggerFailedSoftToHard();

    ExecutionResults scheduleProcessAbort();

    ExecutionResults abortProcessForCaseId(String caseId);
}

```

### `com/barclays/api/status/mgmt/util/SchedulerConstants.java`

**Status:** MODIFIED  
**Purpose:** All existing constants kept verbatim, 7 new constants added at the bottom.

```java
package com.barclays.api.status.mgmt.util;

/**
 * MIGRATION NOTE:
 * All existing constants UNCHANGED.
 * Q_NAME_* constants (full class names) are still used as:
 *   1. commandName stored in QRTZ_JOB_DETAILS.JOB_DATA
 *   2. Input to ExecutorServiceAdapter.toJobKey() to derive short job key
 *   3. Input to CommandExecutorJob.toQueueName() to map to queue name string
 *   4. Input to SchedulerInfoServiceImpl.getCommandName() unchanged
 *
 * NEW additions at bottom: queue name string constants for queues
 * that previously had no string constant (only had Q_NAME_* full class name).
 */
public class SchedulerConstants {

    private SchedulerConstants() {
        // no usages
    }

    // ── Existing Q_NAME_* constants - ALL UNCHANGED ───────────────────────────
    public static final String Q_NAME_OLARESQ =
        "com.barclays.api.status.mgmt.schedular.service.impl.OlaResponseQServiceImpl";
    public static final String Q_NAME_CASLSSQ =
        "com.barclays.api.status.mgmt.schedular.service.impl.CASLSSQServiceImpl";
    public static final String Q_NAME_NOINTERFACEQ =
        "com.barclays.api.status.mgmt.schedular.service.impl.NoInterfaceQServiceImpl";
    public static final String Q_NAME_NTCHOLDQ =
        "com.barclays.api.status.mgmt.schedular.service.impl.NTCHoldQServiceImpl";
    public static final String Q_NAME_NGCBA =
        "com.barclays.api.status.mgmt.schedular.service.impl.NGCBckServiceImpl";
    public static final String Q_NAME_LOGCLEANUP =
        "com.barclays.api.status.mgmt.schedular.service.impl.LogCleanUpServiceImpl";
    public static final String Q_NAME_REQUESTINFOLOGCLEANUP =
        "com.barclays.api.status.mgmt.schedular.service.impl.RequestInfoCleanUpServiceImpl";
    public static final String Q_NAME_NODEQ =
        "com.barclays.api.status.mgmt.schedular.service.impl.NODEQServiceImpl";
    public static final String Q_NAME_NOUPLOADQ =
        "com.barclays.api.status.mgmt.schedular.service.impl.NoUploadQServiceImpl";
    public static final String Q_NAME_BATCHRESUBMITQ =
        "com.barclays.api.status.mgmt.schedular.service.impl.BatchResubmitQServiceImpl";
    public static final String Q_NAME_LAPSEDPROCESS =
        "com.barclays.api.status.mgmt.schedular.service.impl.LapsedProcessServiceImpl";
    public static final String Q_NAME_FAILEDSOFTTOHARD =
        "com.barclays.api.status.mgmt.schedular.service.impl.FailedSoftToHardServiceImpl";
    public static final String Q_NAME_APP_EXPIRY_PROCESS =
        "com.barclays.api.status.mgmt.schedular.service.impl.ApplicationExpiryProcessServiceImpl";
    public static final String Q_NAME_PER_CASE_ABORT =
        "com.barclays.api.status.mgmt.schedular.service.impl.PerCaseAbortServiceImpl";
    public static final String Q_NAME_ADVCLEAN =
        "com.barclays.api.status.mgmt.schedular.service.impl.AdvancedLogCleanUpServiceImpl";
    public static final String Q_NAME_ADVCLEANCOMMAND =
        "com.barclays.api.status.mgmt.schedular.service.impl.AdvancedLogCleanupCommand";
    public static final String Q_NAME_SMSRETRY =
        "com.barclays.api.status.mgmt.schedular.service.impl.SMSRetryServiceImpl";

    // ── Existing query and string constants - ALL UNCHANGED ──────────────────
    public static final String FAILEDSOFTTOHARD_IDENTIFICATION_QUERY =
        "SELECT CASEID FROM VEC_EQ_FAILD_HRD_CNVRSN WHERE STOH_IND='N' AND "
        + "TRUNC(APPSIGNEDDATE)>=TRUNC(SYSDATE-3)";

    public static final String RETRY             = "0";
    public static final String READY             = "Ready";
    public static final String NOINTERFACEQ      = "NOINTERFACEQ";
    public static final String OLARESPONSEQ      = "OLARESPONSEQ";
    public static final String NODEQ             = "NODEQ";
    public static final String CASLSSQ           = "CASLSSQ";
    public static final String NOUPLOADQ         = "NOUPLOADQ";
    public static final String BATCHRESUBMITQ    = "BATCHRESUBMITQ";
    public static final String NTCHOLDQ          = "NTCHOLDQ";
    public static final String ROBOTIC_QUEUE_NAME = "roboticQueueName";
    public static final String TOTAL_TRIGGERED   = "Total-Triggered";
    public static final String TOTAL_FAILED      = "Total-Failed";
    public static final String END_TIME          = "End-Time";
    public static final String RETRIES           = "retries";
    public static final String OWNER             = "owner";
    public static final String DEPLOYMENT_ID     = "DeploymentId";
    public static final String SCHEDULAR_START_EVENT = "SCHEDULAR_INSTANCE_STARTED";
    public static final String ENABLE_CLEANUP    = "isCleanupEnabled";
    public static final String Q_NAME_PERCASELOGCLEANUP =
        "com.barclays.api.status.mgmt.schedular.service.impl.PerCaseLogCleanUpServiceImpl";
    public static final String CASE_ID           = "CASEID";
    public static final String GET_SMS_FAILED_CASE =
        "SELECT PRODUCTAPPID FROM vec_olinterface where INTERFACECODE='IF070' "
        + "and INTSTATUSIND IN ('E','T') and CALLQTY=1";
    public static final String OLDER_THAN_PERIOD = "OlderThanPeriod";
    public static final String OLDER_THAN        = "OlderThan";
    public static final String SMS_RETRY_SCHEDULER = "SMS_RETRY_SCHEDULER ::";
    public static final String STATUS            = "Status";
    public static final String TOTAL_CASES       = "Total-Cases";

    // ── NEW: Queue name string constants added for migration ─────────────────
    // These are the short queue name strings passed to QSchedulerServiceImpl.trigger()
    // Previously these queues only had Q_NAME_* (full class name) constants.
    // Now needed by CommandExecutorJob.toQueueName() switch statement.
    public static final String NGCBACK           = "NGCBack";
    public static final String LOGCLEANUP        = "LogCleanUp";
    public static final String LAPSEDPROCESS     = "LapsedProcess";
    public static final String FAILEDSOFTTOHARD  = "FailedSoftToHard";
    public static final String APP_EXPIRY_PROCESS = "ApplicationExpiryProcess";
    public static final String ADVANCEDLOGCLEANUP = "AdvancedLogCleanUp";
    public static final String SMSRETRY          = "SMSRetry";
}

```

### `com/barclays/api/status/mgmt/schedular/util/SchedulerUtils.java`

**Status:** NEW  
**Purpose:** Holds @Value injected cron expressions, replaces old Reoccurring logic source.

```java
package com.barclays.api.status.mgmt.schedular.util;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

/**
 * NEW CLASS - Migration replacement for old SchedulerUtils.getScheduleTime() calls.
 *
 * In BAMOE 8.x, each Command impl (OlaResponseQServiceImpl etc.) called:
 *   SchedulerUtils schedulerUtils = BeanUtils.getBean(SchedulerUtils.class);
 *   return DateAndTimeUtil.getDateTimeForString(schedulerUtils.getOlaresponseqRuntime());
 *
 * In BAMOE 9.x, SchedulerStartupSeeder uses this class to seed
 * Quartz CronTriggers on startup, replacing Reoccurring.getScheduleTime().
 *
 * Properties sourced from application-dev.properties:
 *   roboticqueue.olaresponseq.runtime=0 0 * * * ?
 *   roboticqueue.nointerfaceq.runtime=0 10 * * * ?
 *   etc.
 */
@Component
public class SchedulerUtils {

    @Value("${roboticqueue.olaresponseq.runtime}")
    private String olaresponseqRuntime;

    @Value("${roboticqueue.nointerfaceq.runtime}")
    private String nointerfaceqRuntime;

    @Value("${roboticqueue.ntcholdq.runtime}")
    private String ntcholdqRuntime;

    @Value("${roboticqueue.ngcback.runtime}")
    private String ngcbackRuntime;

    @Value("${roboticqueue.logCleanup.runtime}")
    private String logCleanupRuntime;

    @Value("${roboticqueue.caslssq.runtime}")
    private String caslssqRuntime;

    @Value("${roboticqueue.nodeq.runtime}")
    private String nodeqRuntime;

    @Value("${roboticqueue.nouploadq.runtime}")
    private String nouploadqRuntime;

    @Value("${roboticqueue.batchResubmitq.runtime}")
    private String batchResubmitqRuntime;

    @Value("${roboticqueue.lapsedProcess.runtime}")
    private String lapsedProcessRuntime;

    public String getOlaresponseqRuntime()   { return olaresponseqRuntime; }
    public String getNointerfaceqRuntime()   { return nointerfaceqRuntime; }
    public String getNtcholdqRuntime()       { return ntcholdqRuntime; }
    public String getNgcbackRuntime()        { return ngcbackRuntime; }
    public String getLogCleanupRuntime()     { return logCleanupRuntime; }
    public String getCaslssqRuntime()        { return caslssqRuntime; }
    public String getNodeqRuntime()          { return nodeqRuntime; }
    public String getNouploadqRuntime()      { return nouploadqRuntime; }
    public String getBatchResubmitqRuntime() { return batchResubmitqRuntime; }
    public String getLapsedProcessRuntime()  { return lapsedProcessRuntime; }
}

```

### `com/barclays/api/status/mgmt/config/QuartzConfig.java`

**Status:** NEW  
**Purpose:** Wires Quartz JDBCJobStore to the BAMOE Oracle datasource.

```java
package com.barclays.api.status.mgmt.config;

import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.autoconfigure.quartz.QuartzDataSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.quartz.SpringBeanJobFactory;

import javax.sql.DataSource;

/**
 * NEW CLASS - Quartz configuration for BAMOE 9.x scheduler migration.
 *
 * Two responsibilities:
 * 1. Wire Quartz to the BAMOE Oracle datasource (RPAM schema)
 *    where QRTZ_* tables live (all 11 tables confirmed present and empty).
 * 2. Register SpringBeanJobFactory so @Autowired works inside
 *    Quartz Job classes (specifically CommandExecutorJob).
 *
 * IMPORTANT: The @QuartzDataSource annotation tells Spring Boot
 * auto-configuration to use this datasource for Quartz JDBCJobStore
 * instead of the default primary datasource.
 *
 * Verify: check your DataSource config for the exact @Bean name
 * of the BAMOE datasource. It may be "bamoeDataSource" or "dataSource"
 * depending on your DataSourceConfig class.
 */
@Configuration
public class QuartzConfig {

    /**
     * Points Quartz JDBCJobStore to the RPAM schema datasource
     * where QRTZ_* tables are present.
     *
     * Adjust @Qualifier value to match your actual BAMOE DataSource bean name.
     */
    @Bean
    @QuartzDataSource
    public DataSource quartzDataSource(
            @Qualifier("bamoeDataSource") DataSource bamoeDataSource) {
        return bamoeDataSource;
    }

    /**
     * Enables Spring dependency injection inside Quartz Job classes.
     * Without this, @Autowired fields in CommandExecutorJob will be null.
     */
    @Bean
    public SpringBeanJobFactory springBeanJobFactory() {
        return new SpringBeanJobFactory();
    }
}

```

### `com/barclays/api/status/mgmt/schedular/adapter/CommandExecutorJob.java`

**Status:** NEW  
**Purpose:** Single Quartz Job replacing all 14 old Command implementations.

```java
package com.barclays.api.status.mgmt.schedular.adapter;

import com.barclays.api.bap.bld.lib.logging.LogFactory;
import com.barclays.api.bap.bld.lib.logging.Logger;
import com.barclays.api.status.mgmt.schedular.service.QSchedulerService;
import com.barclays.api.status.mgmt.util.SchedulerConstants;
import org.quartz.DisallowConcurrentExecution;
import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.quartz.PersistJobDataAfterExecution;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

/**
 * NEW CLASS - Quartz Job implementation that replaces all 14 Command implementations:
 *
 *   OlaResponseQServiceImpl, NoInterfaceQServiceImpl, NTCHoldQServiceImpl,
 *   NGCBckServiceImpl, LogCleanUpServiceImpl, CASLSSQServiceImpl,
 *   NODEQServiceImpl, NoUploadQServiceImpl, BatchResubmitQServiceImpl,
 *   LapsedProcessServiceImpl, FailedSoftToHardServiceImpl,
 *   ApplicationExpiryProcessServiceImpl, AdvancedLogCleanUpServiceImpl,
 *   SMSRetryServiceImpl
 *
 * OLD pattern (each Command impl):
 *   public ExecutionResults execute(CommandContext ctx) throws Exception {
 *       SchedulerUtils.startScheduler();
 *       QSchedulerService schedulerService = BeanUtils.getBean(QSchedulerService.class);
 *       ExecutionResults results = schedulerService.trigger(SchedulerConstants.OLARESPONSEQ);
 *       SchedulerUtils.stopScheduler();
 *       return results;
 *   }
 *
 * NEW pattern (single CommandExecutorJob):
 *   execute() → toQueueName(commandName) → qSchedulerService.trigger(queueName)
 *
 * @DisallowConcurrentExecution: Prevents the same job from running
 * simultaneously across multiple pods. Quartz JDBCJobStore enforces
 * this via DB locking on QRTZ_FIRED_TRIGGERS. Replaces the multi-pod
 * safety issue that existed with BPMN timers in BAMOE 8.x.
 *
 * @PersistJobDataAfterExecution: Ensures job data map is persisted
 * back to QRTZ_JOB_DETAILS after each execution.
 */
@Component
@DisallowConcurrentExecution
@PersistJobDataAfterExecution
public class CommandExecutorJob implements Job {

    private static final Logger LOGGER =
        LogFactory.getLogger(CommandExecutorJob.class);

    /**
     * Key used to store commandName in Quartz JobDataMap.
     * Value is the full Q_NAME_* constant e.g.
     * "com.barclays.api.status.mgmt.schedular.service.impl.OlaResponseQServiceImpl"
     */
    public static final String QUEUE_COMMAND_KEY = "commandName";

    @Autowired
    private QSchedulerService qSchedulerService;

    @Override
    public void execute(JobExecutionContext context)
            throws JobExecutionException {

        String commandName = context.getMergedJobDataMap()
                .getString(QUEUE_COMMAND_KEY);
        String queueName = toQueueName(commandName);

        LOGGER.info("CommandExecutorJob triggered: commandName={}, queueName={}",
                commandName, queueName);
        try {
            // Replaces: SchedulerUtils.startScheduler()
            LOGGER.info("Scheduler started for queue: {}", queueName);

            // Core trigger - same as old Command.execute() calling
            // schedulerService.trigger(SchedulerConstants.OLARESPONSEQ)
            qSchedulerService.trigger(queueName);

            // Replaces: SchedulerUtils.stopScheduler()
            LOGGER.info("Scheduler completed for queue: {}", queueName);

        } catch (Exception e) {
            LOGGER.error("CommandExecutorJob failed for queue: {} - {}",
                    queueName, e.getMessage());
            // false = do not re-fire immediately on failure
            throw new JobExecutionException(e, false);
        }
    }

    /**
     * Maps Q_NAME_* full class name constant to queue name string
     * used in QSchedulerServiceImpl.trigger(queueName).
     *
     * Mapping derived from SchedulerConstants Q_NAME_* values:
     * Last segment of class name → queue name string constant
     *
     * e.g. "...OlaResponseQServiceImpl" → "OLARESPONSEQ"
     *      "...NODEQServiceImpl"        → "NODEQ"
     */
    private String toQueueName(String commandName) {
        // Extract short class name from full package name
        String shortName = commandName != null && commandName.contains(".")
                ? commandName.substring(commandName.lastIndexOf('.') + 1)
                : commandName;

        if (shortName == null) {
            LOGGER.error("commandName is null in JobDataMap");
            return "";
        }

        switch (shortName) {
            case "OlaResponseQServiceImpl":
                return SchedulerConstants.OLARESPONSEQ;        // "OLARESPONSEQ"
            case "NoInterfaceQServiceImpl":
                return SchedulerConstants.NOINTERFACEQ;        // "NOINTERFACEQ"
            case "NTCHoldQServiceImpl":
                return SchedulerConstants.NTCHOLDQ;            // "NTCHOLDQ"
            case "NGCBckServiceImpl":
                return SchedulerConstants.NGCBACK;             // "NGCBack"
            case "LogCleanUpServiceImpl":
                return SchedulerConstants.LOGCLEANUP;          // "LogCleanUp"
            case "CASLSSQServiceImpl":
                return SchedulerConstants.CASLSSQ;             // "CASLSSQ"
            case "NODEQServiceImpl":
                return SchedulerConstants.NODEQ;               // "NODEQ"
            case "NoUploadQServiceImpl":
                return SchedulerConstants.NOUPLOADQ;           // "NOUPLOADQ"
            case "BatchResubmitQServiceImpl":
                return SchedulerConstants.BATCHRESUBMITQ;      // "BATCHRESUBMITQ"
            case "LapsedProcessServiceImpl":
                return SchedulerConstants.LAPSEDPROCESS;       // "LapsedProcess"
            case "FailedSoftToHardServiceImpl":
                return SchedulerConstants.FAILEDSOFTTOHARD;    // "FailedSoftToHard"
            case "ApplicationExpiryProcessServiceImpl":
                return SchedulerConstants.APP_EXPIRY_PROCESS;  // "ApplicationExpiryProcess"
            case "AdvancedLogCleanUpServiceImpl":
                return SchedulerConstants.ADVANCEDLOGCLEANUP;  // "AdvancedLogCleanUp"
            case "SMSRetryServiceImpl":
                return SchedulerConstants.SMSRETRY;            // "SMSRetry"
            default:
                LOGGER.warn("Unknown commandName shortName: {}, using as-is",
                        shortName);
                return commandName;
        }
    }
}

```

### `com/barclays/api/status/mgmt/schedular/adapter/ExecutorServiceAdapter.java`

**Status:** NEW  
**Purpose:** Wraps Quartz Scheduler with the same API surface as the old ExecutorService.

```java
package com.barclays.api.status.mgmt.schedular.adapter;

import com.barclays.api.bap.bld.lib.logging.LogFactory;
import com.barclays.api.bap.bld.lib.logging.Logger;
import org.quartz.CronScheduleBuilder;
import org.quartz.CronTrigger;
import org.quartz.JobBuilder;
import org.quartz.JobDataMap;
import org.quartz.JobDetail;
import org.quartz.JobExecutionContext;
import org.quartz.JobKey;
import org.quartz.Scheduler;
import org.quartz.SchedulerException;
import org.quartz.SimpleScheduleBuilder;
import org.quartz.Trigger;
import org.quartz.TriggerBuilder;
import org.quartz.TriggerKey;
import org.quartz.impl.matchers.GroupMatcher;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.Comparator;
import java.util.Date;
import java.util.HashMap;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

/**
 * NEW CLASS - Adapter that wraps Quartz Scheduler to provide the same
 * API surface as the old org.kie.api.executor.ExecutorService.
 *
 * Used by SchedulerInfoServiceImpl as a drop-in replacement.
 *
 * OLD (BAMOE 8.x):
 *   @Autowired
 *   private ExecutorService kieExecutorService;
 *
 * NEW (BAMOE 9.x):
 *   @Autowired
 *   private ExecutorServiceAdapter executorServiceAdapter;
 *
 * Method mapping:
 *   kieExecutorService.scheduleRequest()     → scheduleRequest()
 *   kieExecutorService.cancelRequest()       → cancelRequest()
 *   kieExecutorService.getQueuedRequests()   → getQueuedRequests()
 *   kieExecutorService.getRequestsByCommand() → isQueued() / isRunning()
 *
 * All state persisted to QRTZ_* Oracle tables in RPAM schema.
 * Multi-pod safe via Quartz JDBCJobStore cluster locking.
 */
@Component
public class ExecutorServiceAdapter {

    private static final Logger LOGGER =
        LogFactory.getLogger(ExecutorServiceAdapter.class);

    /** Quartz job group for all scheduler queue jobs */
    public static final String JOB_GROUP     = "BAMOE_SCHEDULER";

    /** Quartz trigger group for all scheduler queue triggers */
    public static final String TRIGGER_GROUP = "BAMOE_TRIGGER";

    /** Suffix for cron-based recurring triggers (seeded on startup) */
    public static final String CRON_SUFFIX   = "_CRON";

    /**
     * Suffix for one-time override triggers
     * (created when UI reschedules a specific run)
     */
    public static final String ONCE_SUFFIX   = "_ONCE";

    @Autowired
    private Scheduler quartzScheduler;

    // ─────────────────────────────────────────────────────────────────────────
    // scheduleRequest()
    // Replaces: kieExecutorService.scheduleRequest(commandName, date, ctx)
    // ─────────────────────────────────────────────────────────────────────────

    /**
     * Schedules a one-time fire at the given date for the specified queue.
     * Used when UI submits ON action with a new datetime.
     *
     * Creates a SimpleTrigger (one-time) that fires at fireTime.
     * If a previous one-time trigger exists it is removed first.
     * The underlying cron job is preserved - only the next run is overridden.
     *
     * @param commandName Q_NAME_* constant e.g. Q_NAME_OLARESQ
     * @param fireTime    Date/time to fire the job
     * @param contextData Optional context data to store in JobDataMap
     */
    public void scheduleRequest(String commandName,
                                Date fireTime,
                                Map<String, Object> contextData)
            throws SchedulerException {

        String jobKey  = toJobKey(commandName);
        JobKey jk      = JobKey.jobKey(jobKey, JOB_GROUP);
        TriggerKey onceTk = TriggerKey.triggerKey(
                jobKey + ONCE_SUFFIX, TRIGGER_GROUP);

        // Remove any existing one-time override trigger
        if (quartzScheduler.checkExists(onceTk)) {
            quartzScheduler.unscheduleJob(onceTk);
            LOGGER.info("Removed existing one-time trigger for {}", jobKey);
        }

        // Create job if not already seeded by SchedulerStartupSeeder
        if (!quartzScheduler.checkExists(jk)) {
            JobDataMap dm = new JobDataMap();
            dm.put(CommandExecutorJob.QUEUE_COMMAND_KEY, commandName);
            if (contextData != null) {
                dm.putAll(contextData);
            }
            JobDetail job = JobBuilder.newJob(CommandExecutorJob.class)
                    .withIdentity(jk)
                    .usingJobData(dm)
                    .storeDurably(true)
                    .build();
            quartzScheduler.addJob(job, false);
            LOGGER.info("Created new job definition for {}", jobKey);
        }

        // Schedule one-time SimpleTrigger
        Trigger onceTrigger = TriggerBuilder.newTrigger()
                .withIdentity(onceTk)
                .forJob(jk)
                .startAt(fireTime)
                .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                        .withMisfireHandlingInstructionFireNow())
                .build();

        quartzScheduler.scheduleJob(onceTrigger);
        LOGGER.info("Scheduled one-time job {} to fire at {}", jobKey, fireTime);
    }

    // ─────────────────────────────────────────────────────────────────────────
    // cancelRequest()
    // Replaces: kieExecutorService.cancelRequest(requestInfo.getId())
    // ─────────────────────────────────────────────────────────────────────────

    /**
     * Cancels/pauses the scheduler for the specified queue.
     * Used when UI submits OFF action.
     *
     * One-time override trigger is removed entirely.
     * Cron trigger is paused (not deleted) so it can be resumed later.
     *
     * @param commandName Q_NAME_* constant e.g. Q_NAME_OLARESQ
     * @return true if any trigger was cancelled/paused
     */
    public boolean cancelRequest(String commandName) throws SchedulerException {
        String jobKey     = toJobKey(commandName);
        TriggerKey onceTk = TriggerKey.triggerKey(
                jobKey + ONCE_SUFFIX, TRIGGER_GROUP);
        TriggerKey cronTk = TriggerKey.triggerKey(
                jobKey + CRON_SUFFIX, TRIGGER_GROUP);

        boolean cancelled = false;

        // Remove one-time trigger entirely
        if (quartzScheduler.checkExists(onceTk)) {
            quartzScheduler.unscheduleJob(onceTk);
            cancelled = true;
            LOGGER.info("Cancelled one-time trigger for {}", jobKey);
        }

        // Pause cron trigger (preserves the cron schedule for future re-enable)
        if (quartzScheduler.checkExists(cronTk)) {
            quartzScheduler.pauseTrigger(cronTk);
            cancelled = true;
            LOGGER.info("Paused cron trigger for {}", jobKey);
        }

        return cancelled;
    }

    // ─────────────────────────────────────────────────────────────────────────
    // getQueuedRequests()
    // Replaces: kieExecutorService.getQueuedRequests(new QueryContext(0, 30))
    // ─────────────────────────────────────────────────────────────────────────

    /**
     * Returns all currently scheduled (active) queue jobs with their next fire time.
     * Used by SchedulerInfoServiceImpl.getRoboticQList() to populate the UI table.
     *
     * Returns same format as old RequestInfo list:
     *   [{commandName, displayName, nextFireTime}]
     *
     * displayName matches old UI format (e.g. "OlaResponseQ" not "OlaResponseQServiceImpl")
     *
     * @return List of maps with commandName, displayName, nextFireTime
     */
    public List<Map<String, String>> getQueuedRequests() throws SchedulerException {
        List<Map<String, String>> result = new ArrayList<>();

        for (JobKey jk : quartzScheduler.getJobKeys(
                GroupMatcher.jobGroupEquals(JOB_GROUP))) {

            List<? extends Trigger> triggers =
                    quartzScheduler.getTriggersOfJob(jk);

            Date nextFire = null;
            for (Trigger t : triggers) {
                Trigger.TriggerState state =
                        quartzScheduler.getTriggerState(t.getKey());

                // Only include active triggers (NORMAL or WAITING)
                if (state == Trigger.TriggerState.NORMAL
                        || state == Trigger.TriggerState.WAITING) {
                    Date tf = t.getNextFireTime();
                    // One-time override fires earlier - take the minimum
                    if (tf != null
                            && (nextFire == null || tf.before(nextFire))) {
                        nextFire = tf;
                    }
                }
            }

            if (nextFire != null) {
                JobDetail detail = quartzScheduler.getJobDetail(jk);
                String commandName = detail.getJobDataMap()
                        .getString(CommandExecutorJob.QUEUE_COMMAND_KEY);

                // Strip "ServiceImpl" to match old UI display format
                // e.g. "OlaResponseQServiceImpl" → "OlaResponseQ"
                String displayName = jk.getName().replace("ServiceImpl", "");

                Map<String, String> entry = new LinkedHashMap<>();
                entry.put("commandName", commandName);
                entry.put("displayName", displayName);
                // toString() format matches old RequestInfo.getTime().toString()
                entry.put("nextFireTime", nextFire.toString());
                result.add(entry);
            }
        }

        // Sort by nextFireTime ascending - matches old behaviour
        result.sort(Comparator.comparing(m -> m.get("nextFireTime")));
        return result;
    }

    // ─────────────────────────────────────────────────────────────────────────
    // isQueued()
    // Replaces: getRequestsByCommand(commandName, Arrays.asList(STATUS.QUEUED))
    // ─────────────────────────────────────────────────────────────────────────

    /**
     * Returns true if the specified queue has an active (NORMAL/WAITING) trigger.
     * Used in updateDate() to check if a job is already scheduled.
     *
     * @param commandName Q_NAME_* constant
     * @return true if trigger is in NORMAL/WAITING state
     */
    public boolean isQueued(String commandName) throws SchedulerException {
        String jobKey = toJobKey(commandName);
        for (String suffix : Arrays.asList(ONCE_SUFFIX, CRON_SUFFIX)) {
            TriggerKey tk = TriggerKey.triggerKey(jobKey + suffix, TRIGGER_GROUP);
            if (quartzScheduler.checkExists(tk)) {
                Trigger.TriggerState state =
                        quartzScheduler.getTriggerState(tk);
                if (state == Trigger.TriggerState.NORMAL
                        || state == Trigger.TriggerState.WAITING) {
                    return true;
                }
            }
        }
        return false;
    }

    // ─────────────────────────────────────────────────────────────────────────
    // isRunning()
    // Replaces: getRequestsByCommand(commandName, Arrays.asList(STATUS.RUNNING))
    // ─────────────────────────────────────────────────────────────────────────

    /**
     * Returns true if the specified queue job is currently executing.
     * Used in updateDate() to prevent rescheduling a running job.
     *
     * @param commandName Q_NAME_* constant
     * @return true if job is currently executing on any pod
     */
    public boolean isRunning(String commandName) throws SchedulerException {
        String jobKey = toJobKey(commandName);
        for (JobExecutionContext ctx :
                quartzScheduler.getCurrentlyExecutingJobs()) {
            if (ctx.getJobDetail().getKey().getName().equals(jobKey)) {
                return true;
            }
        }
        return false;
    }

    // ─────────────────────────────────────────────────────────────────────────
    // seedCronJob()
    // Called by SchedulerStartupSeeder - replaces Reoccurring.getScheduleTime()
    // ─────────────────────────────────────────────────────────────────────────

    /**
     * Seeds a recurring cron job into QRTZ_JOB_DETAILS + QRTZ_CRON_TRIGGERS.
     * Called once per queue on application startup by SchedulerStartupSeeder.
     *
     * Idempotent: skips silently if job already exists.
     * Safe for multi-pod deployment - all pods attempt to seed,
     * first pod wins, others skip.
     *
     * Replaces: OlaResponseQServiceImpl.getScheduleTime() [Reoccurring interface]
     * which registered jobs with the jBPM executor on startup.
     *
     * @param commandName    Q_NAME_* constant - stored in JobDataMap
     * @param cronExpression Quartz cron expression from properties file
     */
    public void seedCronJob(String commandName, String cronExpression)
            throws SchedulerException {

        String jobKey = toJobKey(commandName);
        JobKey jk     = JobKey.jobKey(jobKey, JOB_GROUP);
        TriggerKey cronTk = TriggerKey.triggerKey(
                jobKey + CRON_SUFFIX, TRIGGER_GROUP);

        if (quartzScheduler.checkExists(jk)) {
            LOGGER.info("Job {} already exists in QRTZ_JOB_DETAILS, skipping seed",
                    jobKey);
            return;
        }

        JobDataMap dm = new JobDataMap();
        dm.put(CommandExecutorJob.QUEUE_COMMAND_KEY, commandName);

        JobDetail job = JobBuilder.newJob(CommandExecutorJob.class)
                .withIdentity(jk)
                .usingJobData(dm)
                .storeDurably(true)
                .build();

        CronTrigger cronTrigger = TriggerBuilder.newTrigger()
                .withIdentity(cronTk)
                .forJob(jk)
                .withSchedule(CronScheduleBuilder
                        .cronSchedule(cronExpression)
                        // Do not fire missed executions (e.g. after restart)
                        .withMisfireHandlingInstructionDoNothing())
                .build();

        quartzScheduler.scheduleJob(job, cronTrigger);
        LOGGER.info("Seeded cron job {} with expression '{}'",
                jobKey, cronExpression);
    }

    // ─────────────────────────────────────────────────────────────────────────
    // toJobKey()
    // Derives short Quartz job key from full Q_NAME_* class name constant
    // ─────────────────────────────────────────────────────────────────────────

    /**
     * Converts full Q_NAME_* class name to short Quartz job key.
     *
     * e.g.
     *   "com.barclays.api.status.mgmt.schedular.service.impl.OlaResponseQServiceImpl"
     *   → "OlaResponseQServiceImpl"
     *
     * This short name is stored in QRTZ_JOB_DETAILS.JOB_NAME column.
     *
     * @param commandName Full Q_NAME_* constant or short name
     * @return Short class name used as Quartz job key
     */
    public String toJobKey(String commandName) {
        if (commandName == null) return "";
        return commandName.contains(".")
                ? commandName.substring(commandName.lastIndexOf('.') + 1)
                : commandName;
    }
}

```

### `com/barclays/api/status/mgmt/schedular/adapter/SchedulerStartupSeeder.java`

**Status:** NEW  
**Purpose:** Seeds all 10 cron jobs into Quartz on application startup.

```java
package com.barclays.api.status.mgmt.schedular.adapter;

import com.barclays.api.bap.bld.lib.logging.LogFactory;
import com.barclays.api.bap.bld.lib.logging.Logger;
import com.barclays.api.status.mgmt.schedular.util.SchedulerUtils;
import com.barclays.api.status.mgmt.util.SchedulerConstants;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.ApplicationListener;
import org.springframework.stereotype.Component;

/**
 * NEW CLASS - Seeds all robotic queue cron jobs into Quartz on application startup.
 *
 * REPLACES: The Reoccurring interface and getScheduleTime() method
 * that each old Command implementation provided.
 *
 * OLD pattern (each *ServiceImpl had this):
 *   @Override
 *   public Date getScheduleTime() {
 *       SchedulerUtils schedulerUtils = BeanUtils.getBean(SchedulerUtils.class);
 *       return DateAndTimeUtil.getDateTimeForString(
 *           schedulerUtils.getOlaresponseqRuntime());
 *   }
 *   → jBPM executor registered this as a recurring job on startup
 *
 * NEW pattern (single seeder for all queues):
 *   onApplicationEvent() → seedCronJob(commandName, cronExpression)
 *   → Quartz CronTrigger persisted to QRTZ_CRON_TRIGGERS + QRTZ_JOB_DETAILS
 *
 * Queues seeded (10 total - matching old application.properties cron entries):
 *   OLARESQ, NOINTERFACEQ, NTCHOLDQ, NGCBA, LOGCLEANUP,
 *   CASLSSQ, NODEQ, NOUPLOADQ, BATCHRESUBMITQ, LAPSEDPROCESS
 *
 * Queues NOT seeded (no cron property - triggered ad-hoc or by other means):
 *   FAILEDSOFTTOHARD, APP_EXPIRY_PROCESS, ADVCLEAN, SMSRETRY, PERCASEABORT
 *
 * Multi-pod safety:
 *   seedCronJob() is idempotent - checks if job exists before inserting.
 *   All pods call this on startup. First pod wins, others skip.
 *   No race condition because Quartz JDBCJobStore uses DB locking.
 *
 * ApplicationReadyEvent fires after full Spring context is ready,
 * ensuring Quartz Scheduler bean is fully initialised before seeding.
 */
@Component
public class SchedulerStartupSeeder
        implements ApplicationListener<ApplicationReadyEvent> {

    private static final Logger LOGGER =
        LogFactory.getLogger(SchedulerStartupSeeder.class);

    @Autowired
    private ExecutorServiceAdapter adapter;

    @Autowired
    private SchedulerUtils schedulerUtils;

    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        LOGGER.info("SchedulerStartupSeeder: seeding all robotic queue cron jobs...");

        // Each seed() call corresponds to one old *ServiceImpl Command class
        // and one roboticqueue.*.runtime property in application.properties

        seed(SchedulerConstants.Q_NAME_OLARESQ,
             schedulerUtils.getOlaresponseqRuntime());
        // Was: OlaResponseQServiceImpl.getScheduleTime()
        //      roboticqueue.olaresponseq.runtime=0 0 * * * ?

        seed(SchedulerConstants.Q_NAME_NOINTERFACEQ,
             schedulerUtils.getNointerfaceqRuntime());
        // Was: NoInterfaceQServiceImpl.getScheduleTime()
        //      roboticqueue.nointerfaceq.runtime=0 10 * * * ?

        seed(SchedulerConstants.Q_NAME_NTCHOLDQ,
             schedulerUtils.getNtcholdqRuntime());
        // Was: NTCHoldQServiceImpl.getScheduleTime()
        //      roboticqueue.ntcholdq.runtime=0 0 0/4 * * ?

        seed(SchedulerConstants.Q_NAME_NGCBA,
             schedulerUtils.getNgcbackRuntime());
        // Was: NGCBckServiceImpl.getScheduleTime()
        //      roboticqueue.ngcback.runtime=0 0 0/2 * * ?

        seed(SchedulerConstants.Q_NAME_LOGCLEANUP,
             schedulerUtils.getLogCleanupRuntime());
        // Was: LogCleanUpServiceImpl.getScheduleTime()
        //      roboticqueue.logCleanup.runtime=0 0 23 * * ?

        seed(SchedulerConstants.Q_NAME_CASLSSQ,
             schedulerUtils.getCaslssqRuntime());
        // Was: CASLSSQServiceImpl.getScheduleTime()
        //      roboticqueue.caslssq.runtime=0 30 0/4 * * ?

        seed(SchedulerConstants.Q_NAME_NODEQ,
             schedulerUtils.getNodeqRuntime());
        // Was: NODEQServiceImpl.getScheduleTime()
        //      roboticqueue.nodeq.runtime=0 30 0/2 * * ?

        seed(SchedulerConstants.Q_NAME_NOUPLOADQ,
             schedulerUtils.getNouploadqRuntime());
        // Was: NoUploadQServiceImpl.getScheduleTime()
        //      roboticqueue.nouploadq.runtime=0 30 0/4 * * ?

        seed(SchedulerConstants.Q_NAME_BATCHRESUBMITQ,
             schedulerUtils.getBatchResubmitqRuntime());
        // Was: BatchResubmitQServiceImpl.getScheduleTime()
        //      roboticqueue.batchResubmitq.runtime=0 40 0/4 * * ?

        seed(SchedulerConstants.Q_NAME_LAPSEDPROCESS,
             schedulerUtils.getLapsedProcessRuntime());
        // Was: LapsedProcessServiceImpl.getScheduleTime()
        //      roboticqueue.lapsedProcess.runtime=0 0 8 * * ?

        LOGGER.info("SchedulerStartupSeeder: all 10 queue cron jobs seeded.");
    }

    private void seed(String commandName, String cronExpression) {
        try {
            adapter.seedCronJob(commandName, cronExpression);
        } catch (Exception e) {
            LOGGER.error("Failed to seed cron job for {}: {}",
                    commandName, e.getMessage());
        }
    }
}

```

### `com/barclays/api/status/mgmt/model/graphql/UserTaskInstance.java`

**Status:** MODIFIED  
**Purpose:** Added externalReferenceId field and nested processInstance.businessKey.

```java
package com.barclays.api.status.mgmt.model.graphql;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import lombok.Data;
import java.time.Instant;
import java.util.List;

/**
 * MIGRATION CHANGE:
 * Added two new fields to support caseId resolution in QSchedulerServiceImpl:
 *
 * 1. externalReferenceId - maps to RPAM.TASKS.EXTERNAL_REFERENCE_ID column
 *    May hold caseId - verify once scheduler tasks appear in TASKS table
 *
 * 2. processInstance (nested) - for businessKey access via GraphQL nested query
 *    processInstance { businessKey } returns the caseId from Data Index
 *    This is the primary strategy for getCaseId() in QSchedulerServiceImpl
 *
 * All other fields unchanged from original migrated version.
 */
@Data
@JsonIgnoreProperties(ignoreUnknown = true)
public class UserTaskInstance {

    // ── Original fields - ALL UNCHANGED ──────────────────────────────────────
    private String id;
    private String name;
    private String state;
    private String processInstanceId;
    private String actualOwner;
    private String referenceName;
    private Instant lastUpdate;
    private Instant started;
    private List<String> potentialGroups;
    private List<String> potentialUsers;

    // ── NEW fields added for scheduler migration ──────────────────────────────

    /**
     * NEW - Maps to RPAM.TASKS.EXTERNAL_REFERENCE_ID column.
     * Used as fallback strategy 2 for getCaseId() in QSchedulerServiceImpl.
     * Verify content once scheduler queue tasks appear in TASKS table.
     */
    private String externalReferenceId;

    /**
     * NEW - Nested ProcessInstance reference for businessKey access.
     * Populated when GraphQL query includes: processInstance { businessKey }
     * businessKey = caseId (numeric application ID) in this system.
     * Primary strategy 1 for getCaseId() in QSchedulerServiceImpl.
     */
    private ProcessInstanceRef processInstance;

    /**
     * Minimal nested object to carry businessKey from GraphQL
     * ProcessInstance { businessKey } nested query.
     */
    @Data
    @JsonIgnoreProperties(ignoreUnknown = true)
    public static class ProcessInstanceRef {
        private String businessKey;
    }
}

```

### `src/main/resources/graphql/queries/user-tasks-by-name.graphql`

**Status:** NEW  
**Purpose:** New GraphQL query used by QSchedulerServiceImpl.fetchTasksByName().

```graphql
# NEW FILE - Added for scheduler migration
# Replaces: advanceRuntimeDataService.queryUserTasksByVariables(attributes, ...)
#           where attributes filtered by TASK_ATTR_NAME (queue name) and TASK_ATTR_STATUS
#
# Used by: QSchedulerServiceImpl.fetchTasksByName(queueName)
# Called by: CommandExecutorJob → QSchedulerService.trigger(queueName)
#
# processInstance { businessKey } nested query provides caseId
# without needing a separate GraphQL call per task.
# businessKey in BAMOE 9.x Data Index = numeric application caseId

query UserTasksByName($name: String!, $states: [String!]) {
  UserTaskInstances(
    where: {
      and: [
        { name: { equal: $name } },
        { state: { in: $states } }
      ]
    }
    pagination: { limit: 900, offset: 0 }
  ) {
    id
    name
    state
    processInstanceId
    actualOwner
    started
    lastUpdate
    potentialGroups
    potentialUsers
    referenceName
    externalReferenceId
    processInstance {
      businessKey
    }
  }
}

```

### `com/barclays/api/status/mgmt/schedular/service/impl/QSchedulerServiceImpl.java`

**Status:** MODIFIED (FULL REWRITE)  
**Purpose:** Core trigger() logic preserved; all jBPM APIs replaced with GraphQL + REST.

```java
package com.barclays.api.status.mgmt.schedular.service.impl;

import com.barclays.api.bap.bld.lib.logging.LogFactory;
import com.barclays.api.bap.bld.lib.logging.Logger;
import com.barclays.api.status.mgmt.config.GraphQLQueryLoader;
import com.barclays.api.status.mgmt.model.graphql.GraphQLRequest;
import com.barclays.api.status.mgmt.model.graphql.GraphQLResponse;
import com.barclays.api.status.mgmt.model.graphql.UserTaskInstance;
import com.barclays.api.status.mgmt.schedular.model.ExecutionResults;
import com.barclays.api.status.mgmt.schedular.service.QSchedulerService;
import com.barclays.api.status.mgmt.util.SchedulerConstants;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.commons.lang3.exception.ExceptionUtils;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;

import java.time.Instant;
import java.time.LocalTime;
import java.time.temporal.ChronoUnit;
import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

/**
 * MIGRATION: Complete rewrite of QSchedulerServiceImpl.
 *
 * OLD (BAMOE 8.x) APIs removed and replaced:
 * ┌──────────────────────────────────────────────────┬──────────────────────────────────────────┐
 * │ OLD jBPM 8 API                                   │ NEW BAMOE 9.x API                        │
 * ├──────────────────────────────────────────────────┼──────────────────────────────────────────┤
 * │ RuntimeDataService.getTasksAssigned...()         │ GraphQL UserTasksByName query             │
 * │ AdvanceRuntimeDataService.queryUserTasksByVars() │ GraphQL via graphQlRestClient            │
 * │ UserTaskInstanceWithPotOwnerDesc                 │ UserTaskInstance (model.graphql)          │
 * │ UserTaskService.claim(taskId, userId)            │ REST /usertasks/instance/{id}/transition  │
 * │ UserTaskService.start(taskId, userId)            │ REST /usertasks/instance/{id}/transition  │
 * │ UserTaskService.complete(taskId, userId, vars)   │ REST /usertasks/instance/{id}/transition  │
 * │ ProcessService.setProcessVariable()              │ ProcessUtilService.setProcessVariable()   │
 * │ task.getCorrelationKey() → caseId               │ processInstance.businessKey (GraphQL)     │
 * │ task.getCreatedOn()                              │ UserTaskInstance.started (Instant)        │
 * │ task.getActualOwner()                            │ UserTaskInstance.actualOwner              │
 * │ PeopleAssignments.getPotentialOwners()           │ UserTaskInstance.potentialUsers           │
 * │ kjars.getMainContainer() filter                  │ Removed (no KJar concept in 9.x)         │
 * └──────────────────────────────────────────────────┴──────────────────────────────────────────┘
 *
 * trigger() method logic is PRESERVED exactly.
 * Only the API calls to fetch and process tasks are changed.
 *
 * Other interface methods (triggerLapsedProcess, triggerFailedSoftToHard etc.)
 * are kept as stubs pending separate migration work.
 */
@Service
public class QSchedulerServiceImpl implements QSchedulerService {

    private static final Logger LOGGER =
        LogFactory.getLogger(QSchedulerServiceImpl.class);

    // Task state constants - replaces old Status.Ready / Status.Reserved enum
    private static final String STATE_READY    = "Ready";
    private static final String STATE_RESERVED = "Reserved";
    private static final List<String> ACTIVE_STATES =
        List.of(STATE_READY, STATE_RESERVED);

    // ── Dependencies ──────────────────────────────────────────────────────────

    // Replaces: graphQlRestClient (already present in migrated project)
    private final RestClient graphQlRestClient;

    // Replaces: localRestClient for user task transitions
    private final RestClient localRestClient;

    private final ObjectMapper objectMapper;
    private final GraphQLQueryLoader queryLoader;

    // Replaces: queryUtils (already present in migrated project)
    private final QueryUtils queryUtils;

    // Replaces: processService.setProcessVariable()
    private final ProcessUtilService processUtilService;

    // Replaces: migrationUtils (already present in migrated project)
    private final MigrationUtils migrationUtils;

    // Replaces: variableIdentifierService (already present in migrated project)
    private final VariableIdentifierService variableIdentifierService;

    // Used for: VEC_CASELOCK update + OLA filter queries
    private final JdbcTemplate businessJdbcTemplate;

    public QSchedulerServiceImpl(
            RestClient graphQlRestClient,
            @Qualifier("localRestClient") RestClient localRestClient,
            ObjectMapper objectMapper,
            GraphQLQueryLoader queryLoader,
            QueryUtils queryUtils,
            ProcessUtilService processUtilService,
            MigrationUtils migrationUtils,
            VariableIdentifierService variableIdentifierService,
            @Qualifier("businessJdbcTemplate") JdbcTemplate businessJdbcTemplate) {
        this.graphQlRestClient        = graphQlRestClient;
        this.localRestClient          = localRestClient;
        this.objectMapper             = objectMapper;
        this.queryLoader              = queryLoader;
        this.queryUtils               = queryUtils;
        this.processUtilService       = processUtilService;
        this.migrationUtils           = migrationUtils;
        this.variableIdentifierService = variableIdentifierService;
        this.businessJdbcTemplate     = businessJdbcTemplate;
    }

    // ═════════════════════════════════════════════════════════════════════════
    // trigger() - MAIN SCHEDULER ENTRY POINT
    // Called by CommandExecutorJob at scheduled time for each queue
    // Logic preserved from old implementation, only APIs changed
    // ═════════════════════════════════════════════════════════════════════════

    @Override
    public ExecutionResults trigger(String queueName) {
        ExecutionResults executionResults = new ExecutionResults();
        Map<String, Object> results = new HashMap<>();
        results.put("Queue-Name", queueName);
        results.put("Start-Time", LocalTime.now());

        int failedCases        = 0;
        int noOfTasksTriggered = 0;
        String requestId = InterfaceUtils.getCorrelationId();

        try {
            LOGGER.info("Starting Scheduler Run for {} :: {}", queueName, requestId);

            // ── Step 1: Fetch active tasks for this queue ─────────────────────
            // OLD: advanceRuntimeDataService.queryUserTasksByVariables(attributes...)
            // NEW: GraphQL UserTasksByName query via graphQlRestClient
            List<UserTaskInstance> tasks = fetchTasksByName(queueName);

            if (tasks == null || tasks.isEmpty()) {
                LOGGER.info("No tasks found for queue: {}", queueName);
                results.put(SchedulerConstants.TOTAL_TRIGGERED, 0);
                results.put(SchedulerConstants.TOTAL_FAILED, 0);
                results.put(SchedulerConstants.END_TIME, LocalTime.now());
                executionResults.setData(results);
                return executionResults;
            }

            int maxCount = tasks.size();
            LOGGER.info("No. Of Cases for Execution: {}", maxCount);
            results.put("Total-Tasks", maxCount);

            // ── Step 2: OLARESPONSEQ-specific filtering ───────────────────────
            // Logic preserved exactly from old implementation
            if (SchedulerConstants.OLARESPONSEQ.equalsIgnoreCase(queueName)
                    && !tasks.isEmpty()) {
                tasks = applyOlaResponseQFilters(tasks, queueName);
            }

            LOGGER.info("Total Cases after filtering: {}", tasks.size());

            // ── Step 3: Process each task ─────────────────────────────────────
            for (UserTaskInstance pendingTask : tasks) {
                try {
                    String caseId = getCaseId(pendingTask);

                    // Migration check - preserved from old code
                    if (MigrationUtils.isMigrationRunnerActive()) {
                        migrationUtils.migrate(caseId);
                    }

                    String actualOwner = pendingTask.getActualOwner();
                    // OLD: getQueueUser(pendingTask, queueName)
                    // NEW: resolveOwner() using potentialUsers list
                    String ownerToUse  = resolveOwner(
                            pendingTask, actualOwner, queueName);

                    LOGGER.info("actualOwner: {} caseId: {}",
                            actualOwner, caseId);
                    LOGGER.info("resolvedOwner: {} caseId: {}",
                            ownerToUse, caseId);

                    if (ownerToUse != null) {
                        executeTask(pendingTask, ownerToUse,
                                caseId, requestId, queueName);
                        noOfTasksTriggered++;
                    }

                } catch (Exception e) {
                    LOGGER.error("Failed processing task in queue {}: {}",
                            queueName,
                            ExceptionUtils.getRootCauseMessage(e));
                    failedCases++;
                }
            }

            results.put(SchedulerConstants.TOTAL_TRIGGERED, noOfTasksTriggered);
            results.put(SchedulerConstants.TOTAL_FAILED, failedCases);
            results.put(SchedulerConstants.END_TIME, LocalTime.now());

            LOGGER.info("Total triggered in {} :: {}", queueName, noOfTasksTriggered);
            LOGGER.info("Total failed in {} :: {}", queueName, failedCases);

        } catch (Exception e) {
            results.put("ERROR", ExceptionUtils.getMessage(e));
            LOGGER.error("Scheduler run failed for {}: {}",
                    queueName, e.getMessage());
        }

        executionResults.setData(results);
        return executionResults;
    }

    // ═════════════════════════════════════════════════════════════════════════
    // fetchTasksByName()
    // REPLACES: advanceRuntimeDataService.queryUserTasksByVariables(attributes...)
    //           where attributes = [in(STATUS, "Ready","Reserved"),
    //                               equalTo(NAME, queueName),
    //                               in(DEPLOYMENT_ID, kjars...)]
    // ═════════════════════════════════════════════════════════════════════════

    private List<UserTaskInstance> fetchTasksByName(String queueName) {
        try {
            GraphQLRequest request = new GraphQLRequest(
                queryLoader.getQuery("userTasksByName"),
                Map.of(
                    "name",   queueName,
                    "states", ACTIVE_STATES
                )
            );

            String responseBody = graphQlRestClient
                    .post()
                    .body(request)
                    .retrieve()
                    .body(String.class);

            GraphQLResponse<Map<String, Object>> response =
                objectMapper.readValue(responseBody,
                    new TypeReference<GraphQLResponse<Map<String, Object>>>() {});

            if (response != null
                    && response.getData() != null
                    && response.getData().containsKey("UserTaskInstances")) {
                return objectMapper.convertValue(
                    response.getData().get("UserTaskInstances"),
                    new TypeReference<List<UserTaskInstance>>() {}
                );
            }
        } catch (Exception e) {
            LOGGER.error("GraphQL fetchTasksByName failed for {}: {}",
                    queueName, e.getMessage());
        }
        return Collections.emptyList();
    }

    // ═════════════════════════════════════════════════════════════════════════
    // applyOlaResponseQFilters()
    // PRESERVES: All OLARESPONSEQ-specific filtering logic from old trigger()
    // Only changed: task object type from UserTaskInstanceWithPotOwnerDesc
    //               to UserTaskInstance and date comparison uses Instant
    // ═════════════════════════════════════════════════════════════════════════

    private List<UserTaskInstance> applyOlaResponseQFilters(
            List<UserTaskInstance> tasks, String queueName) {

        // ── Filter 1: Task must be older than 1 hour ──────────────────────────
        // OLD: c.getCreatedOn().getTime() <= DateAndTimeUtil.getPreviousHrTime().getTime()
        // NEW: task.getStarted().isBefore(oneHourAgo) using Instant
        Instant oneHourAgo = Instant.now().minus(1, ChronoUnit.HOURS);
        tasks = tasks.stream()
                .filter(t -> t.getStarted() != null
                          && t.getStarted().isBefore(oneHourAgo))
                .collect(Collectors.toList());

        if (tasks.isEmpty()) return tasks;

        // ── Filter 2: Exclude CO Accept cases ────────────────────────────────
        // Logic preserved exactly from old OLA filter block
        StringBuilder caseIdStr = new StringBuilder();
        for (UserTaskInstance t : tasks) {
            caseIdStr.append("'").append(getCaseId(t)).append("'").append(",");
        }
        String caseIds = caseIdStr.substring(0, caseIdStr.length() - 1);

        String coAppsQuery =
            "SELECT P.ID FROM VEC_PRODUCTAPP P, VEC_DECENGINE DE, VEC_ACTPARMS V "
            + "WHERE P.CR_DECENGINEID=DE.ID AND DE.PRODUCTAPPID=P.ID "
            + "AND V.PRODUCTAPPID=P.ID AND DE.SHARESCREEN3='A' "
            + "AND P.ID IN (" + caseIds + ") AND V.AGGRIND='N'";

        List<String> coCaseIds = new ArrayList<>();
        try {
            List<Map<String, Object>> coApps =
                queryUtils.selectQuery(coAppsQuery, null);
            if (coApps != null) {
                coApps.forEach(row ->
                    coCaseIds.add(String.valueOf(row.get("ID"))));
            }
        } catch (Exception e) {
            LOGGER.error("CO apps query failed: {}", e.getMessage());
        }

        LOGGER.info("Total Amazon CO Accept cases: {}", coCaseIds.size());
        tasks = tasks.stream()
                .filter(t -> !coCaseIds.contains(getCaseId(t)))
                .collect(Collectors.toList());

        if (tasks.isEmpty()) return tasks;

        // ── Filter 3: Exclude QUEUE_IND='C' completed cases ──────────────────
        // Logic preserved exactly from old OLA filter block
        StringBuilder remaining = new StringBuilder();
        tasks.forEach(t -> remaining.append("'")
                                    .append(getCaseId(t))
                                    .append("'").append(","));
        String remainingIds = remaining.substring(0, remaining.length() - 1);

        String completedQuery =
            "SELECT PRODUCTAPPID FROM VEC_ACTPARMS "
            + "WHERE QUEUE_IND = 'C' AND PRODUCTAPPID IN (" + remainingIds + ")";

        List<String> completedCaseIds = new ArrayList<>();
        try {
            List<Map<String, Object>> completedApps =
                queryUtils.selectQuery(completedQuery, null);
            if (completedApps != null) {
                completedApps.forEach(row -> {
                    String caseId = String.valueOf(row.get("PRODUCTAPPID"));
                    completedCaseIds.add(caseId);
                    // Abort process for completed cases - same as old code
                    processUtilService.abortCurrentProcessForCaseId(caseId);
                });
            }
        } catch (Exception e) {
            LOGGER.error("Completed apps query failed: {}", e.getMessage());
        }

        LOGGER.info("Total completed cases: {}", completedCaseIds.size());
        tasks = tasks.stream()
                .filter(t -> !completedCaseIds.contains(getCaseId(t)))
                .collect(Collectors.toList());

        LOGGER.info("Total Cases after OLA filtering: {}", tasks.size());
        return tasks;
    }

    // ═════════════════════════════════════════════════════════════════════════
    // executeTask()
    // REPLACES: executeTask(UserTaskInstanceWithPotOwnerDesc, userId, ...)
    //           which used UserTaskService.claim/start/complete
    // NEW: Uses BAMOE 9.x REST API /usertasks/instance/{id}/transition
    // ═════════════════════════════════════════════════════════════════════════

    private void executeTask(UserTaskInstance pendingTask,
                             String userId,
                             String caseId,
                             String requestId,
                             String queueName) {
        String taskId       = pendingTask.getId();
        String processInstId = pendingTask.getProcessInstanceId();

        try {
            // ── Step 1: Claim if Ready ────────────────────────────────────────
            // OLD: if (pendingTask.getStatus().equals(Status.Ready.name()))
            //          userTaskService.claim(taskId, userId)
            // NEW: REST POST /usertasks/instance/{id}/transition?user={user}
            if (STATE_READY.equals(pendingTask.getState())) {
                localRestClient.post()
                    .uri("/usertasks/instance/{taskId}/transition?user={user}",
                            taskId, userId)
                    .body(Map.of("transitionId", "claim"))
                    .retrieve()
                    .toBodilessEntity();
            }

            // ── Step 2: Start task ────────────────────────────────────────────
            // OLD: userTaskService.start(taskId, userId)
            // NEW: REST POST with transitionId: start
            localRestClient.post()
                .uri("/usertasks/instance/{taskId}/transition?user={user}",
                        taskId, userId)
                .body(Map.of("transitionId", "start"))
                .retrieve()
                .toBodilessEntity();

            // ── Step 3: Set process variable ──────────────────────────────────
            // OLD: processService.setProcessVariable(processInstanceId, "initiator", userId)
            // NEW: ProcessUtilService (already migrated in project)
            processUtilService.setProcessVariable(processInstId, "initiator", userId);

            // ── Step 4: Update case lock table ────────────────────────────────
            // Table: VECTUSCASE.VEC_CASELOCK
            // Columns confirmed: CASEID NUMBER(10), USERCODE VARCHAR2(50)
            updateUserInCaseLockTable(caseId, userId);

            // ── Step 5: Get task variables ────────────────────────────────────
            // OLD: variableIdentifierService.getTaskVariableForTaskName(nextPath)
            // NEW: Same service - already migrated in project
            TaskVariable userTaskVariable =
                variableIdentifierService.getTaskVariableForTaskName(queueName);

            Map<String, Object> taskVariables = null;
            if (userTaskVariable != null) {
                userTaskVariable.setData(requestId);
                userTaskVariable.setCaseId(caseId);
                userTaskVariable.setInitiator(userId);
                taskVariables = userTaskVariable.getTaskVariables();
            }

            // ── Step 6: Complete task ─────────────────────────────────────────
            // OLD: userTaskService.complete(taskId, userId, userTaskVariable.getTaskVariables())
            // NEW: REST POST with transitionId: complete + data payload
            Map<String, Object> completeBody = new HashMap<>();
            completeBody.put("transitionId", "complete");
            if (taskVariables != null) {
                completeBody.put("data", taskVariables);
            }

            localRestClient.post()
                .uri("/usertasks/instance/{taskId}/transition?user={user}",
                        taskId, userId)
                .body(completeBody)
                .retrieve()
                .toBodilessEntity();

            // ── Step 7: History log ───────────────────────────────────────────
            // OLD: queryUtils.historylog(...) - preserved unchanged
            queryUtils.historylog(
                "Application triggered via Schedular",
                caseId, processInstId, userId);

            LOGGER.info("Task {} completed for caseId {} by {}",
                    taskId, caseId, userId);

        } catch (Exception e) {
            LOGGER.error("executeTask failed taskId={} caseId={}: {}",
                    taskId, caseId,
                    ExceptionUtils.getRootCauseMessage(e));
            throw e;
        }
    }

    // ═════════════════════════════════════════════════════════════════════════
    // resolveOwner()
    // REPLACES: getQueueUser(UserTaskInstanceWithPotOwnerDesc, queueName)
    //           which used PeopleAssignments.getPotentialOwners() / OrganizationalEntity
    // NEW: Uses UserTaskInstance.actualOwner and potentialUsers (List<String>)
    // ═════════════════════════════════════════════════════════════════════════

    private String resolveOwner(UserTaskInstance task,
                                String actualOwner,
                                String queueName) {
        // OLD: if (actualOwner != null && BPMNConstants.registeredUsers.contains(actualOwner))
        // NEW: Same logic, actualOwner field now comes from GraphQL UserTaskInstance
        if (actualOwner != null
                && BPMNConstants.registeredUsers.contains(actualOwner)) {
            LOGGER.info("Using actualOwner: {} for task: {}",
                    actualOwner, task.getId());
            return actualOwner;
        }

        // OLD: PeopleAssignments → OrganizationalEntity list iteration
        //      userTaskService.getTask(taskId).getPeopleAssignments()
        //                    .getPotentialOwners()
        // NEW: UserTaskInstance.potentialUsers (already populated by GraphQL)
        if (task.getPotentialUsers() != null) {
            for (String user : task.getPotentialUsers()) {
                if (BPMNConstants.registeredUsers.contains(user)) {
                    LOGGER.info("Found registered user {} in potentialUsers "
                                + "for task: {}", user, task.getId());
                    return user;
                }
            }
        }

        LOGGER.info("No registered owner found for task: {} queue: {}",
                task.getId(), queueName);
        return null;
    }

    // ═════════════════════════════════════════════════════════════════════════
    // getCaseId()
    // REPLACES: pendingTask.getCorrelationKey() parsing
    //           OLD: correlationKey = "2147491080:someProcessId"
    //                caseId = correlationKey.substring(0, indexOf(":"))
    // NEW: Multi-strategy approach since RPAM.TASKS.REFERENCE_NAME
    //      does NOT hold caseId (confirmed by DB analysis)
    // ═════════════════════════════════════════════════════════════════════════

    private String getCaseId(UserTaskInstance task) {

        // ── Strategy 1: processInstance.businessKey from GraphQL nested query ─
        // Populated by: processInstance { businessKey } in userTasksByName.graphql
        // businessKey in BAMOE 9.x Data Index = numeric caseId e.g. "2147491080"
        // This is the PRIMARY strategy - most reliable
        if (task.getProcessInstance() != null
                && task.getProcessInstance().getBusinessKey() != null
                && !task.getProcessInstance().getBusinessKey().isEmpty()) {
            String bk = task.getProcessInstance().getBusinessKey();
            if (bk.contains(":")) {
                return bk.substring(0, bk.indexOf(":"));
            }
            return bk;
        }

        // ── Strategy 2: externalReferenceId ──────────────────────────────────
        // Maps to RPAM.TASKS.EXTERNAL_REFERENCE_ID column
        // Verify content once scheduler tasks appear in TASKS table
        if (task.getExternalReferenceId() != null
                && !task.getExternalReferenceId().isEmpty()) {
            String ref = task.getExternalReferenceId();
            return ref.contains(":") ? ref.substring(0, ref.indexOf(":")) : ref;
        }

        // ── Strategy 3: referenceName ─────────────────────────────────────────
        // Maps to RPAM.TASKS.REFERENCE_NAME column
        // Confirmed NOT to hold caseId for current tasks but kept as fallback
        if (task.getReferenceName() != null
                && !task.getReferenceName().isEmpty()) {
            String ref = task.getReferenceName();
            return ref.contains(":") ? ref.substring(0, ref.indexOf(":")) : ref;
        }

        // ── Strategy 4: Direct DB lookup via PROCESS_INSTANCE_STATE_LOG ───────
        // RPAM.PROCESS_INSTANCE_STATE_LOG has BUSINESS_KEY column (confirmed)
        // and PROCESS_INSTANCE_ID column for joining
        return fetchCaseIdFromDb(task.getProcessInstanceId());
    }

    private String fetchCaseIdFromDb(String processInstanceId) {
        try {
            List<Map<String, Object>> rows = businessJdbcTemplate.queryForList(
                "SELECT BUSINESS_KEY FROM RPAM.PROCESS_INSTANCE_STATE_LOG "
                + "WHERE PROCESS_INSTANCE_ID = ? "
                + "AND BUSINESS_KEY IS NOT NULL "
                + "AND ROWNUM = 1",
                processInstanceId
            );
            if (!rows.isEmpty()) {
                String bk = String.valueOf(rows.get(0).get("BUSINESS_KEY"));
                return bk.contains(":") ? bk.substring(0, bk.indexOf(":")) : bk;
            }
        } catch (Exception e) {
            LOGGER.error("fetchCaseIdFromDb failed for processInstanceId={}: {}",
                    processInstanceId, e.getMessage());
        }
        // Last resort - use processInstanceId itself
        LOGGER.warn("Could not resolve caseId for processInstanceId={}, "
                  + "using processInstanceId as fallback", processInstanceId);
        return processInstanceId;
    }

    // ═════════════════════════════════════════════════════════════════════════
    // updateUserInCaseLockTable()
    // Unchanged logic from old implementation.
    // Table confirmed: VECTUSCASE.VEC_CASELOCK
    // Columns confirmed: CASEID NUMBER(10) NOT NULL, USERCODE VARCHAR2(50)
    // ═════════════════════════════════════════════════════════════════════════

    private void updateUserInCaseLockTable(String caseId, String userId) {
        try {
            int updated = businessJdbcTemplate.update(
                "UPDATE VECTUSCASE.VEC_CASELOCK "
                + "SET USERCODE = ?, CREATEDATE = SYSDATE, CREATETIME = SYSDATE "
                + "WHERE CASEID = ?",
                userId,
                Long.parseLong(caseId)
            );
            LOGGER.info("VEC_CASELOCK updated {} row(s) caseId={} userId={}",
                    updated, caseId, userId);
        } catch (NumberFormatException e) {
            LOGGER.error("caseId is not numeric: {} - skipping lock update", caseId);
        } catch (Exception e) {
            LOGGER.error("updateUserInCaseLockTable failed caseId={}: {}",
                    caseId, e.getMessage());
        }
    }

    // ═════════════════════════════════════════════════════════════════════════
    // Remaining interface methods - stubs pending separate migration
    // These are not part of the scheduler flow triggered by CommandExecutorJob
    // ═════════════════════════════════════════════════════════════════════════

    @Override
    public ExecutionResults triggerPumaRefStatus() {
        LOGGER.info("triggerPumaRefStatus - pending migration");
        return new ExecutionResults();
    }

    @Override
    public boolean populateCaseIdInPurgeTable() {
        LOGGER.info("populateCaseIdInPurgeTable - pending migration");
        return false;
    }

    @Override
    public ExecutionResults triggerLapsedProcess() {
        LOGGER.info("triggerLapsedProcess - pending migration");
        return new ExecutionResults();
    }

    @Override
    public ExecutionResults triggerFailedSoftToHard() {
        LOGGER.info("triggerFailedSoftToHard - pending migration");
        return new ExecutionResults();
    }

    @Override
    public ExecutionResults scheduleProcessAbort() {
        LOGGER.info("scheduleProcessAbort - pending migration");
        return new ExecutionResults();
    }

    @Override
    public ExecutionResults abortProcessForCaseId(String caseId) {
        LOGGER.info("abortProcessForCaseId {} - pending migration", caseId);
        return new ExecutionResults();
    }
}

```

### `com/barclays/api/status/mgmt/service/impl/SchedulerInfoServiceImpl.java`

**Status:** MODIFIED  
**Purpose:** getRoboticQList() and updateDate() rewritten to use ExecutorServiceAdapter.

```java
package com.barclays.api.status.mgmt.service.impl;

import com.barclays.api.bap.bld.lib.logging.LogFactory;
import com.barclays.api.bap.bld.lib.logging.Logger;
import com.barclays.api.status.mgmt.schedular.adapter.ExecutorServiceAdapter;
import com.barclays.api.status.mgmt.util.SchedulerConstants;
import com.google.gson.Gson;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Date;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * MIGRATION: SchedulerInfoServiceImpl
 *
 * Changes from BAMOE 8.x to BAMOE 9.x:
 *
 * REMOVED:
 *   @Autowired private ExecutorService kieExecutorService;
 *   @Autowired private KJarDeploymentResourceList kjars; (if not used elsewhere)
 *   getCommandContextObj() method
 *   updateRequestInfo() method
 *   cancelRequestInfo() method
 *   getQList() method
 *
 * ADDED:
 *   @Autowired private ExecutorServiceAdapter executorServiceAdapter;
 *
 * MODIFIED:
 *   getRoboticQList() - uses adapter.getQueuedRequests()
 *   updateDate()      - uses adapter.scheduleRequest() / cancelRequest()
 *
 * UNCHANGED:
 *   All @Value injected properties
 *   dateConvert() static method
 *   getCommandName() method
 *   Controller contract (getRoboticQList, updateDate signatures)
 *   Response format to UI
 *
 * NOTE: getRoboticQList() and updateDate() are the only two methods
 * called from HomeScreenController. HomeScreenController is UNCHANGED.
 */
@Service
public class SchedulerInfoServiceImpl implements SchedulerInfoService {

    private static final Logger logger =
        LogFactory.getLogger(SchedulerInfoServiceImpl.class);

    // ── CHANGE: Replace kieExecutorService with ExecutorServiceAdapter ────────
    // OLD: @Autowired private ExecutorService kieExecutorService;
    // NEW:
    @Autowired
    private ExecutorServiceAdapter executorServiceAdapter;

    // ── All @Value properties UNCHANGED ──────────────────────────────────────
    @Value("${records.per.transaction:900}")
    private Integer recordsPerTransaction;

    @Value("${cases.per.round:100}")
    private Integer casesPerRound;

    @Value("${cleanup.enabled:true}")
    private boolean isCleanupEnabled;

    @Value("${request.info.records.per.transaction:10000}")
    private Integer requestInfoRecordsPerTransaction;

    @Value("${request.info.records.threshold:200000}")
    private Integer requestInfoRecordsThreshold;

    @Value("${single.run.process.instance.threshold:10000}")
    private Integer singleRunProcessInstanceThreshold;

    @Value("${process.instance.threshold.value:10000}")
    private Integer processInstanceThresholdValue;

    @Value("${purge.scheduler.threads:1}")
    private Integer purgeSchedulerThreads;

    @Value("${process.instance.id.per.transaction:100}")
    private Integer processInstanceIdPerTransaction;

    @Value("${purge.scheduler.thread.intervals:30000}")
    private Integer purgeSchedulerThreadIntervals;

    // ── Constants UNCHANGED ───────────────────────────────────────────────────
    protected final String RECORDS_PER_TRANSACTION  = "RecordsPerTransaction";
    protected final String CASES_PER_ROUND          = "casesPerRound";
    protected final String REQUEST_INFO_RECORD_COUNT = "RequestInfoRecordsPerTransaction";
    protected final String REQUEST_INFO_THRESHOLD    = "RequestInfoRecordsThreshold";

    // ═════════════════════════════════════════════════════════════════════════
    // getRoboticQList()
    // Called by: HomeScreenController.getRoboticQList() GET endpoint
    // Populates UI scheduler table: Queue Name | Scheduled Time (In BST)
    //
    // CHANGE: Replace kieExecutorService.getQueuedRequests() with
    //         executorServiceAdapter.getQueuedRequests()
    //
    // Response format UNCHANGED: List<Map> where each Map = {queueName: time}
    // ═════════════════════════════════════════════════════════════════════════

    @Override
    public List<Map> getRoboticQList() {
        List<Map> res = new ArrayList<>();
        try {
            // OLD:
            //   List<RequestInfo> reqList = kieExecutorService
            //       .getQueuedRequests(new QueryContext(0, 30));
            //   List<RequestInfo> result = reqList.stream()
            //       .filter(s -> s.getCommandName().contains("com.barclays"))
            //       .collect(Collectors.toList());
            //   Collections.sort(result, Comparator.comparing(RequestInfo::getTime));
            //   for (RequestInfo req : result) {
            //       String[] qname = req.getCommandName().split("\\.");
            //       response.put(qname[qname.length-1].replace("ServiceImpl",""),
            //                    req.getTime().toString());
            //   }
            //
            // NEW: ExecutorServiceAdapter returns same format directly
            List<Map<String, String>> raw =
                executorServiceAdapter.getQueuedRequests();

            for (Map<String, String> entry : raw) {
                Map<String, String> response = new HashMap<>();
                // displayName = "OlaResponseQ", nextFireTime = "Sun Jun 14 08:00:00 BST 2026"
                // Matches old format: {qname[qname.length-1].replace("ServiceImpl","")
                //                      : req.getTime().toString()}
                response.put(entry.get("displayName"),
                             entry.get("nextFireTime"));
                res.add(response);
            }
        } catch (Exception e) {
            LoggingUtils.logErrorMessage(logger, null, e.getMessage());
        }
        return res;
    }

    // ═════════════════════════════════════════════════════════════════════════
    // updateDate()
    // Called by: HomeScreenController.timerUpdateForNoInterfaceQ() POST endpoint
    // Handles both ON (reschedule) and OFF (cancel) actions from UI
    //
    // CHANGE: Replace kieExecutorService calls with executorServiceAdapter calls
    // Logic flow UNCHANGED
    // ═════════════════════════════════════════════════════════════════════════

    @Override
    public String updateDate(Object requestData) {
        try {
            String jsonString = new Gson().toJson(requestData, Object.class);
            Map<String, Object> reqObj =
                ValidationUtils.getStringRequestAsMap(jsonString);
            LoggingUtils.logInfoMessage(logger, reqObj.toString(),
                    requestData.toString());

            String queueName       = (String) reqObj.get(
                    SchedulerConstants.ROBOTIC_QUEUE_NAME);
            String commandName     = getCommandName(queueName);
            String statusIndicator = (String) reqObj.get("statusIndicator");

            // ── OFF: Cancel the scheduler ─────────────────────────────────────
            if (statusIndicator.equalsIgnoreCase("OFF")) {
                // OLD: cancelRequestInfo(reqList, reqObj)
                //      → kieExecutorService.cancelRequest(req.getId())
                // NEW:
                executorServiceAdapter.cancelRequest(commandName);
                LoggingUtils.logInfoMessage(logger, "OFF Scheduler:", queueName);
                return queueName + " Scheduler Has Been Switch OFF";
            }

            // ── ON: Reschedule the queue ──────────────────────────────────────
            // Check if already running - same logic as before
            if (executorServiceAdapter.isRunning(commandName)) {
                return queueName + " Is already in a running State";
            }

            Date newDate = dateConvert((String) reqObj.get("newTimer"));

            // OLD: cancelRequest(existing) + scheduleRequest(commandName, date, ctx)
            //      with complex QUEUED/CANCELLED state checks and updateRequestInfo()
            // NEW: simplified - cancel any existing one-time trigger then reschedule
            //      Cron trigger is not affected (only one-time override is replaced)
            executorServiceAdapter.cancelRequest(commandName);
            executorServiceAdapter.scheduleRequest(
                    commandName, newDate, getContextData());

            LoggingUtils.logInfoMessage(logger,
                    "Rescheduled " + queueName + " at ", newDate.toString());
            return queueName + " Has Been Rescheduled At "
                    + reqObj.get("newTimer");

        } catch (Exception e) {
            LoggingUtils.logErrorMessage(logger, null, e.getMessage());
            return "Update Fail";
        }
    }

    // ═════════════════════════════════════════════════════════════════════════
    // getContextData()
    // NEW helper - replaces getCommandContextObj(RequestInfo req)
    // Returns Map instead of CommandContext (removed in BAMOE 9.x)
    // Values preserved from old getCommandContextObj() implementation
    // ═════════════════════════════════════════════════════════════════════════

    private Map<String, Object> getContextData() {
        Map<String, Object> data = new HashMap<>();
        data.put(RECORDS_PER_TRANSACTION, recordsPerTransaction);
        data.put(SchedulerConstants.ENABLE_CLEANUP, isCleanupEnabled);
        data.put(CASES_PER_ROUND, casesPerRound);
        data.put(REQUEST_INFO_RECORD_COUNT, requestInfoRecordsPerTransaction);
        data.put(REQUEST_INFO_THRESHOLD, requestInfoRecordsThreshold);
        data.put("processInstanceThresholdValue", processInstanceThresholdValue);
        data.put("singleRunProcessInstanceThreshold",
                singleRunProcessInstanceThreshold);
        data.put("processInstanceIdPerTransaction",
                processInstanceIdPerTransaction);
        data.put("purgeSchedulerThreadIntervals", purgeSchedulerThreadIntervals);
        data.put("purgeSchedulerThreads", purgeSchedulerThreads);
        data.put("processInstanceRecordsDeleted", 0);
        return data;
    }

    // ═════════════════════════════════════════════════════════════════════════
    // dateConvert() - UNCHANGED from old implementation
    // Converts UI date string to Date object
    // Format: "dd-M-yyyy hh:mm:ss a" e.g. "13-06-2026 8:56:00 PM"
    // ═════════════════════════════════════════════════════════════════════════

    public static Date dateConvert(String stringDate) throws ParseException {
        SimpleDateFormat formatter =
            new SimpleDateFormat("dd-M-yyyy hh:mm:ss a");
        Date date = formatter.parse(stringDate);
        LoggingUtils.logInfoMessage(logger,
                "New Date to be scheduled", date.toString());
        return date;
    }

    // ═════════════════════════════════════════════════════════════════════════
    // getCommandName() - UNCHANGED from old implementation
    // Maps UI queue name string to Q_NAME_* full class name constant
    // ═════════════════════════════════════════════════════════════════════════

    public String getCommandName(String s) {
        if (s == null) return null;
        if (SchedulerConstants.Q_NAME_NGCBA.contains(s))
            return SchedulerConstants.Q_NAME_NGCBA;
        else if (SchedulerConstants.Q_NAME_LOGCLEANUP.contains(s))
            return SchedulerConstants.Q_NAME_LOGCLEANUP;
        else if (SchedulerConstants.Q_NAME_NOINTERFACEQ.contains(s))
            return SchedulerConstants.Q_NAME_NOINTERFACEQ;
        else if (SchedulerConstants.Q_NAME_NTCHOLDQ.contains(s))
            return SchedulerConstants.Q_NAME_NTCHOLDQ;
        else if (SchedulerConstants.Q_NAME_OLARESQ.contains(s))
            return SchedulerConstants.Q_NAME_OLARESQ;
        else if (SchedulerConstants.Q_NAME_CASLSSQ.contains(s))
            return SchedulerConstants.Q_NAME_CASLSSQ;
        else if (SchedulerConstants.Q_NAME_NODEQ.contains(s))
            return SchedulerConstants.Q_NAME_NODEQ;
        else if (SchedulerConstants.Q_NAME_NOUPLOADQ.contains(s))
            return SchedulerConstants.Q_NAME_NOUPLOADQ;
        else if (SchedulerConstants.Q_NAME_BATCHRESUBMITQ.contains(s))
            return SchedulerConstants.Q_NAME_BATCHRESUBMITQ;
        else if (SchedulerConstants.Q_NAME_LAPSEDPROCESS.contains(s))
            return SchedulerConstants.Q_NAME_LAPSEDPROCESS;
        else if (SchedulerConstants.Q_NAME_FAILEDSOFTTOHARD.contains(s))
            return SchedulerConstants.Q_NAME_FAILEDSOFTTOHARD;
        else if (SchedulerConstants.Q_NAME_APP_EXPIRY_PROCESS.contains(s))
            return SchedulerConstants.Q_NAME_APP_EXPIRY_PROCESS;
        else if (SchedulerConstants.Q_NAME_ADVCLEAN.contains(s))
            return SchedulerConstants.Q_NAME_ADVCLEAN;
        else if (SchedulerConstants.Q_NAME_SMSRETRY.contains(s))
            return SchedulerConstants.Q_NAME_SMSRETRY;
        return null;
    }

    // ═════════════════════════════════════════════════════════════════════════
    // REMOVED METHODS - do not port these to new version
    // ═════════════════════════════════════════════════════════════════════════
    //
    // getCommandContextObj(RequestInfo req)
    //   → REMOVED: CommandContext is org.kie.api.executor class, not in 9.x
    //   → REPLACED BY: getContextData() returning Map<String,Object>
    //
    // updateRequestInfo(List<RequestInfo> reqList, Map<String,Object> reqObj)
    //   → REMOVED: RequestInfo is org.kie.api.executor class, not in 9.x
    //   → REPLACED BY: executorServiceAdapter.cancelRequest() + scheduleRequest()
    //
    // cancelRequestInfo(List<RequestInfo> reqList, Map<String,Object> reqObj)
    //   → REMOVED: same reason
    //   → REPLACED BY: executorServiceAdapter.cancelRequest()
    //
    // getQList(String qName, List<STATUS> status)
    //   → REMOVED: STATUS enum is org.kie.api.executor class, not in 9.x
    //   → REPLACED BY: executorServiceAdapter.isQueued() / isRunning()
}

```
## 5. Configuration Reference

### 5.1 `application-dev.properties` additions

```properties
# ── Quartz JDBCJobStore Configuration ─────────────────────────────────────
# Replaces: org.kie.api.executor.ExecutorService internal scheduling
# Uses: RPAM schema QRTZ_* tables (all 11 confirmed present and empty)

spring.quartz.job-store-type=jdbc
spring.quartz.jdbc.initialize-schema=never
spring.quartz.auto-startup=true
spring.quartz.properties.org.quartz.scheduler.instanceName=CASScheduler
spring.quartz.properties.org.quartz.scheduler.instanceId=AUTO
spring.quartz.properties.org.quartz.jobStore.class=org.quartz.impl.jdbcjobstore.JobStoreTX
spring.quartz.properties.org.quartz.jobStore.driverDelegateClass=org.quartz.impl.jdbcjobstore.oracle.OracleDelegate
spring.quartz.properties.org.quartz.jobStore.tablePrefix=QRTZ_
spring.quartz.properties.org.quartz.jobStore.isClustered=true
spring.quartz.properties.org.quartz.jobStore.clusterCheckinInterval=20000
spring.quartz.properties.org.quartz.threadPool.threadCount=5

# ── Robotic Queue Cron Runtimes ────────────────────────────────────────────
# UNCHANGED values from the old application.properties.
# Used by SchedulerUtils (@Value injection) and SchedulerStartupSeeder
# to seed Quartz CronTriggers on startup.

roboticqueue.olaresponseq.runtime=0 0 * * * ?
roboticqueue.nointerfaceq.runtime=0 10 * * * ?
roboticqueue.ntcholdq.runtime=0 0 0/4 * * ?
roboticqueue.ngcback.runtime=0 0 0/2 * * ?
roboticqueue.logCleanup.runtime=0 0 23 * * ?
roboticqueue.caslssq.runtime=0 30 0/4 * * ?
roboticqueue.nodeq.runtime=0 30 0/2 * * ?
roboticqueue.nouploadq.runtime=0 30 0/4 * * ?
roboticqueue.batchResubmitq.runtime=0 40 0/4 * * ?
roboticqueue.lapsedProcess.runtime=0 0 8 * * ?

# retriggerOnlineapp.runtime is SEPARATE — used by RetriggerOnlineAppsSchedular,
# which uses @Scheduled(fixedRateString). Completely unchanged by this migration.
retriggerOnlineapp.runtime=#{300 * 1000}
```

### 5.2 `build.gradle` addition

```groovy
implementation 'org.springframework.boot:spring-boot-starter-quartz'
```

### 5.3 `GraphQLQueryLoader.java` change

Add one line inside the existing constructor (alongside the other `loadQuery(...)` calls already present):

```java
loadQuery("userTasksByName", "graphql/queries/user-tasks-by-name.graphql");
```

### 5.4 Files to delete

```
OlaResponseQServiceImpl.java
NoInterfaceQServiceImpl.java
NTCHoldQServiceImpl.java
NGCBckServiceImpl.java
LogCleanUpServiceImpl.java
CASLSSQServiceImpl.java
NODEQServiceImpl.java
NoUploadQServiceImpl.java
BatchResubmitQServiceImpl.java
LapsedProcessServiceImpl.java
FailedSoftToHardServiceImpl.java
ApplicationExpiryProcessServiceImpl.java
AdvancedLogCleanUpServiceImpl.java
SMSRetryServiceImpl.java
```

All 14 had an identical pattern: `execute()` only called `QSchedulerService.trigger(queueName)`, and `getScheduleTime()` only returned a cron-based date. Both responsibilities are now covered by `CommandExecutorJob` + `SchedulerStartupSeeder`.

---

## 6. Verification SQL (run after deployment)

```sql
-- Should return 10 rows after first startup
SELECT JOB_NAME, JOB_GROUP, DESCRIPTION
FROM RPAM.QRTZ_JOB_DETAILS
WHERE SCHED_NAME = 'CASScheduler';

-- Should return 10 cron triggers
SELECT JOB_NAME, TRIGGER_NAME, CRON_EXPRESSION,
       TO_CHAR(NEXT_FIRE_TIME/1000, '99999999999') AS NEXT_FIRE_EPOCH
FROM RPAM.QRTZ_CRON_TRIGGERS
WHERE SCHED_NAME = 'CASScheduler';

-- After UI submits ON with a new time, check the one-time override:
SELECT TRIGGER_NAME, NEXT_FIRE_TIME
FROM RPAM.QRTZ_SIMPLE_TRIGGERS
WHERE SCHED_NAME = 'CASScheduler';

-- After UI submits OFF, trigger should be paused or absent:
SELECT TRIGGER_NAME, TRIGGER_STATE
FROM RPAM.QRTZ_TRIGGERS
WHERE SCHED_NAME = 'CASScheduler';
```

---

## 7. Known Pending Items (Not Covered by This Package)

These `QSchedulerService` interface methods are migrated as logging stubs only and need separate follow-up work:

- `triggerLapsedProcess()`
- `triggerFailedSoftToHard()`
- `triggerPumaRefStatus()`
- `populateCaseIdInPurgeTable()`
- `scheduleProcessAbort()`
- `abortProcessForCaseId(String caseId)`

The `getCaseId()` fallback chain in `QSchedulerServiceImpl` should be revisited once real scheduler tasks appear in `RPAM.TASKS`, to confirm which strategy is actually populated and simplify accordingly.
