All 11 Quartz tables already exist in the `RPAM` schema. This is perfect — no DDL needed at all. The tables are:

- `QRTZ_BLOB_TRIGGERS`
- `QRTZ_CALENDARS`
- `QRTZ_CRON_TRIGGERS`
- `QRTZ_FIRED_TRIGGERS`
- `QRTZ_JOB_DETAILS`
- `QRTZ_LOCKS`
- `QRTZ_PAUSED_TRIGGER_GRPS`
- `QRTZ_SCHEDULER_STATE`
- `QRTZ_SIMPLE_TRIGGERS`
- `QRTZ_SIMPROP_TRIGGERS`
- `QRTZ_TRIGGERS`

This is the complete standard Quartz schema. You can proceed directly to implementation.

---

**One important check first** — verify if these tables are currently empty or already have data from a previous Quartz usage:

```sql
SELECT 'QRTZ_JOB_DETAILS' T, COUNT(*) C FROM QRTZ_JOB_DETAILS
UNION ALL
SELECT 'QRTZ_TRIGGERS', COUNT(*) FROM QRTZ_TRIGGERS
UNION ALL
SELECT 'QRTZ_CRON_TRIGGERS', COUNT(*) FROM QRTZ_CRON_TRIGGERS
UNION ALL
SELECT 'QRTZ_SIMPLE_TRIGGERS', COUNT(*) FROM QRTZ_SIMPLE_TRIGGERS
UNION ALL
SELECT 'QRTZ_FIRED_TRIGGERS', COUNT(*) FROM QRTZ_FIRED_TRIGGERS;
```

Share the counts. If tables are empty, you're clear to start. If they have stale data from old deployments, we'll need a targeted cleanup before startup seeding runs — otherwise `seedCronJob()` will correctly skip already-existing jobs (which is fine if the data is valid, but problematic if it's stale from a different app context).

---

**Also check the `SCHED_NAME` column** in existing data (if any):

```sql
SELECT DISTINCT SCHED_NAME FROM QRTZ_JOB_DETAILS;
```

Quartz uses `SCHED_NAME` (set via `spring.quartz.properties.org.quartz.scheduler.instanceName=CASScheduler`) to namespace all rows. If the existing data has a different scheduler name, your new instance won't see it and will create fresh rows — which is actually fine and safe.

---

Once you confirm those counts, the implementation plan is fully ready to execute. The complete file list again for clarity:

| File | Action |
|---|---|
| `build.gradle` | Add `spring-boot-starter-quartz` |
| `application-dev.properties` | Add Quartz JDBCJobStore config |
| `SchedulerUtils.java` | New — holds cron expressions from properties |
| `CommandExecutorJob.java` | New — Quartz Job bridge to your `QSchedulerService` |
| `ExecutorServiceAdapter.java` | New — replaces `kieExecutorService` calls |
| `SchedulerStartupSeeder.java` | New — seeds all 13 queues on startup |
| `SchedulerInfoServiceImpl.java` | Modify — swap ~5 call sites only |
| Everything else | Untouched |

Run those count queries and share the results.

All tables are completely empty (all counts = 0) and `SCHED_NAME` returns no rows. This is the best possible situation — a clean slate with the correct schema already in place.

**You are 100% ready to implement. No cleanup, no conflicts, no migration of existing data needed.**

---

Here is the complete, final, copy-paste ready implementation. All files in one place:

---

**`build.gradle`** — add one line to dependencies:
```groovy
implementation 'org.springframework.boot:spring-boot-starter-quartz'
```

---

**`application-dev.properties`** — add this block:
```properties
# Quartz - JDBCJobStore (multi-pod safe, Oracle)
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

# Robotic Queue Cron Runtimes
roboticqueue.olarespons eq.runtime=0 0 * * * ?
roboticqueue.nointerfaceq.runtime=0 10 * * * ?
roboticqueue.ntcholdq.runtime=0 0 0/4 * * ?
roboticqueue.ngcback.runtime=0 0 0/2 * * ?
roboticqueue.logCleanup.runtime=0 0 23 * * ?
roboticqueue.caslssq.runtime=0 30 0/4 * * ?
roboticqueue.nodeq.runtime=0 30 0/2 * * ?
roboticqueue.nouploadq.runtime=0 30 0/4 * * ?
roboticqueue.batchResubmitq.runtime=0 40 0/4 * * ?
roboticqueue.lapsedProcess.runtime=0 0 8 * * ?
```

**Important:** Quartz needs to use the same datasource as your BAMOE (`spring.datasource.bamoe`). Add this config class:

---

**`QuartzConfig.java`** — wires Quartz to your existing Oracle datasource:
```java
package com.barclays.api.status.mgmt.config;

import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.autoconfigure.quartz.QuartzDataSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.quartz.SpringBeanJobFactory;

import javax.sql.DataSource;

@Configuration
public class QuartzConfig {

    /**
     * Tells Quartz to use the BAMOE datasource (RPAM schema)
     * where QRTZ_* tables live.
     */
    @Bean
    @QuartzDataSource
    public DataSource quartzDataSource(
            @Qualifier("bamoeDataSource") DataSource bamoeDataSource) {
        return bamoeDataSource;
    }

    /**
     * Enables Spring autowiring inside Quartz Job classes.
     * Without this, @Autowired in CommandExecutorJob won't work.
     */
    @Bean
    public SpringBeanJobFactory springBeanJobFactory() {
        return new SpringBeanJobFactory();
    }
}
```

> **Note:** Verify the bean name of your BAMOE datasource. In your project it's likely `bamoeDataSource` or `datasource` — check your datasource config class for the exact `@Bean` name and adjust accordingly.

---

**`SchedulerUtils.java`**:
```java
package com.barclays.api.status.mgmt.schedular.util;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
public class SchedulerUtils {

    @Value("${roboticqueue.olarespons eq.runtime}")
    private String olaresponseeqRuntime;

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

    public String getOlaresponseqRuntime()   { return olaresponseeqRuntime; }
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

---

**`CommandExecutorJob.java`**:
```java
package com.barclays.api.status.mgmt.schedular.adapter;

import com.barclays.api.status.mgmt.schedular.service.QSchedulerService;
import com.barclays.api.status.mgmt.util.SchedulerConstants;
import lombok.extern.slf4j.Slf4j;
import org.quartz.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@DisallowConcurrentExecution
@PersistJobDataAfterExecution
public class CommandExecutorJob implements Job {

    public static final String QUEUE_COMMAND_KEY = "commandName";

    @Autowired
    private QSchedulerService qSchedulerService;

    @Override
    public void execute(JobExecutionContext context) 
            throws JobExecutionException {
        String commandName = context.getMergedJobDataMap()
                                    .getString(QUEUE_COMMAND_KEY);
        String queueName = toQueueName(commandName);
        log.info("CommandExecutorJob triggered: commandName={}, queueName={}",
                commandName, queueName);
        try {
            qSchedulerService.trigger(queueName);
            log.info("CommandExecutorJob completed: queueName={}", queueName);
        } catch (Exception e) {
            log.error("CommandExecutorJob failed: queueName={}", queueName, e);
            throw new JobExecutionException(e, false);
        }
    }

    private String toQueueName(String commandName) {
        if (commandName.contains("OlaResponseQ"))       return SchedulerConstants.OLARESPONSEQ;
        if (commandName.contains("NoInterfaceQ"))       return SchedulerConstants.NOINTERFACEQ;
        if (commandName.contains("NTCHoldQ")
         || commandName.contains("NtcHoldQ"))           return SchedulerConstants.NTCHOLDQ;
        if (commandName.contains("NGCBck")
         || commandName.contains("NGCBack"))            return "NGCBack";
        if (commandName.contains("LogCleanUp"))         return "LogCleanUp";
        if (commandName.contains("CASLSSQ"))            return SchedulerConstants.CASLSSQ;
        if (commandName.contains("NODEQ"))              return SchedulerConstants.NODEQ;
        if (commandName.contains("NoUploadQ"))          return SchedulerConstants.NOUPLOADQ;
        if (commandName.contains("BatchResubmitQ"))     return SchedulerConstants.BATCHRESUBMITQ;
        if (commandName.contains("LapsedProcess"))      return "LapsedProcess";
        if (commandName.contains("FailedSoftToHard"))   return "FailedSoftToHard";
        if (commandName.contains("ApplicationExpiry"))  return "ApplicationExpiryProcess";
        if (commandName.contains("AdvancedLogCleanUp")) return "AdvancedLogCleanUp";
        if (commandName.contains("SMSRetry"))           return "SMSRetry";
        return commandName;
    }
}
```

---

**`ExecutorServiceAdapter.java`**:
```java
package com.barclays.api.status.mgmt.schedular.adapter;

import lombok.extern.slf4j.Slf4j;
import org.quartz.*;
import org.quartz.impl.matchers.GroupMatcher;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.*;

@Slf4j
@Component
public class ExecutorServiceAdapter {

    public static final String JOB_GROUP     = "BAMOE_SCHEDULER";
    public static final String TRIGGER_GROUP = "BAMOE_TRIGGER";
    public static final String CRON_SUFFIX   = "_CRON";
    public static final String ONCE_SUFFIX   = "_ONCE";

    @Autowired
    private Scheduler quartzScheduler;

    /**
     * Replaces: kieExecutorService.scheduleRequest(commandName, date, ctx)
     */
    public void scheduleRequest(String commandName, Date fireTime,
                                Map<String, Object> contextData)
            throws SchedulerException {

        String jobKey  = toJobKey(commandName);
        JobKey jk      = JobKey.jobKey(jobKey, JOB_GROUP);
        TriggerKey onceTk = TriggerKey.triggerKey(
                jobKey + ONCE_SUFFIX, TRIGGER_GROUP);

        // Remove any existing one-time trigger
        if (quartzScheduler.checkExists(onceTk)) {
            quartzScheduler.unscheduleJob(onceTk);
        }

        // Create job if not seeded yet
        if (!quartzScheduler.checkExists(jk)) {
            JobDataMap dm = new JobDataMap();
            dm.put(CommandExecutorJob.QUEUE_COMMAND_KEY, commandName);
            if (contextData != null) dm.putAll(contextData);

            JobDetail job = JobBuilder.newJob(CommandExecutorJob.class)
                    .withIdentity(jk)
                    .usingJobData(dm)
                    .storeDurably(true)
                    .build();
            quartzScheduler.addJob(job, false);
        }

        // Schedule one-time trigger
        Trigger once = TriggerBuilder.newTrigger()
                .withIdentity(onceTk)
                .forJob(jk)
                .startAt(fireTime)
                .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                        .withMisfireHandlingInstructionFireNow())
                .build();

        quartzScheduler.scheduleJob(once);
        log.info("Scheduled one-time job {} at {}", jobKey, fireTime);
    }

    /**
     * Replaces: kieExecutorService.cancelRequest(id)
     */
    public boolean cancelRequest(String commandName) throws SchedulerException {
        String jobKey     = toJobKey(commandName);
        TriggerKey onceTk = TriggerKey.triggerKey(jobKey + ONCE_SUFFIX, TRIGGER_GROUP);
        TriggerKey cronTk = TriggerKey.triggerKey(jobKey + CRON_SUFFIX, TRIGGER_GROUP);

        boolean cancelled = false;
        if (quartzScheduler.checkExists(onceTk)) {
            quartzScheduler.unscheduleJob(onceTk);
            cancelled = true;
            log.info("Cancelled one-time trigger for {}", jobKey);
        }
        if (quartzScheduler.checkExists(cronTk)) {
            quartzScheduler.pauseTrigger(cronTk);
            cancelled = true;
            log.info("Paused cron trigger for {}", jobKey);
        }
        return cancelled;
    }

    /**
     * Replaces: kieExecutorService.getQueuedRequests(QueryContext)
     * Returns data in same format as old getRoboticQList()
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
                if (state == Trigger.TriggerState.NORMAL
                 || state == Trigger.TriggerState.WAITING) {
                    Date tf = t.getNextFireTime();
                    // One-time override takes priority (earlier time)
                    if (tf != null && (nextFire == null || tf.before(nextFire))) {
                        nextFire = tf;
                    }
                }
            }

            if (nextFire != null) {
                JobDetail detail = quartzScheduler.getJobDetail(jk);
                String commandName = detail.getJobDataMap()
                        .getString(CommandExecutorJob.QUEUE_COMMAND_KEY);
                // Strip "ServiceImpl" to get display name matching old UI
                String displayName = jk.getName().replace("ServiceImpl", "");

                Map<String, String> entry = new LinkedHashMap<>();
                entry.put("commandName", commandName);
                entry.put("displayName", displayName);
                entry.put("nextFireTime", nextFire.toString());
                result.add(entry);
            }
        }

        result.sort(Comparator.comparing(m -> m.get("nextFireTime")));
        return result;
    }

    /**
     * Replaces: getRequestsByCommand(name, STATUS.RUNNING)
     */
    public boolean isRunning(String commandName) throws SchedulerException {
        String jobKey = toJobKey(commandName);
        for (JobExecutionContext ctx :
                quartzScheduler.getCurrentlyExecutingJobs()) {
            if (ctx.getJobDetail().getKey().getName().equals(jobKey))
                return true;
        }
        return false;
    }

    /**
     * Seeds a recurring cron job on startup.
     * Safe to call repeatedly — skips if job already exists.
     */
    public void seedCronJob(String commandName, String cronExpression)
            throws SchedulerException {
        String jobKey = toJobKey(commandName);
        JobKey jk     = JobKey.jobKey(jobKey, JOB_GROUP);
        TriggerKey cronTk = TriggerKey.triggerKey(
                jobKey + CRON_SUFFIX, TRIGGER_GROUP);

        if (quartzScheduler.checkExists(jk)) {
            log.info("Job {} already seeded, skipping", jobKey);
            return;
        }

        JobDataMap dm = new JobDataMap();
        dm.put(CommandExecutorJob.QUEUE_COMMAND_KEY, commandName);

        JobDetail job = JobBuilder.newJob(CommandExecutorJob.class)
                .withIdentity(jk)
                .usingJobData(dm)
                .storeDurably(true)
                .build();

        CronTrigger cron = TriggerBuilder.newTrigger()
                .withIdentity(cronTk)
                .forJob(jk)
                .withSchedule(CronScheduleBuilder
                        .cronSchedule(cronExpression)
                        .withMisfireHandlingInstructionDoNothing())
                .build();

        quartzScheduler.scheduleJob(job, cron);
        log.info("Seeded cron job {} with expression '{}'",
                jobKey, cronExpression);
    }

    public String toJobKey(String commandName) {
        return commandName.contains(".")
                ? commandName.substring(commandName.lastIndexOf('.') + 1)
                : commandName;
    }
}
```

---

**`SchedulerStartupSeeder.java`**:
```java
package com.barclays.api.status.mgmt.schedular.adapter;

import com.barclays.api.status.mgmt.schedular.util.SchedulerUtils;
import com.barclays.api.status.mgmt.util.SchedulerConstants;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.ApplicationListener;
import org.springframework.stereotype.Component;

@Slf4j
@Component
public class SchedulerStartupSeeder
        implements ApplicationListener<ApplicationReadyEvent> {

    @Autowired private ExecutorServiceAdapter adapter;
    @Autowired private SchedulerUtils schedulerUtils;

    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        log.info("SchedulerStartupSeeder: seeding queue cron jobs...");
        seed(SchedulerConstants.Q_NAME_OLARESQ,
             schedulerUtils.getOlaresponseqRuntime());
        seed(SchedulerConstants.Q_NAME_NOINTERFACEQ,
             schedulerUtils.getNointerfaceqRuntime());
        seed(SchedulerConstants.Q_NAME_NTCHOLDQ,
             schedulerUtils.getNtcholdqRuntime());
        seed(SchedulerConstants.Q_NAME_NGCBA,
             schedulerUtils.getNgcbackRuntime());
        seed(SchedulerConstants.Q_NAME_LOGCLEANUP,
             schedulerUtils.getLogCleanupRuntime());
        seed(SchedulerConstants.Q_NAME_CASLSSQ,
             schedulerUtils.getCaslssqRuntime());
        seed(SchedulerConstants.Q_NAME_NODEQ,
             schedulerUtils.getNodeqRuntime());
        seed(SchedulerConstants.Q_NAME_NOUPLOADQ,
             schedulerUtils.getNouploadqRuntime());
        seed(SchedulerConstants.Q_NAME_BATCHRESUBMITQ,
             schedulerUtils.getBatchResubmitqRuntime());
        seed(SchedulerConstants.Q_NAME_LAPSEDPROCESS,
             schedulerUtils.getLapsedProcessRuntime());
        log.info("SchedulerStartupSeeder: all jobs seeded.");
    }

    private void seed(String commandName, String cronExpression) {
        try {
            adapter.seedCronJob(commandName, cronExpression);
        } catch (Exception e) {
            log.error("Failed seeding {}: {}", commandName, e.getMessage(), e);
        }
    }
}
```

---

**`SchedulerInfoServiceImpl.java`** — only the changed parts:

```java
// REMOVE:
// @Autowired
// private ExecutorService kieExecutorService;
// @Autowired
// private KJarDeploymentResourceList kjars;  ← keep if used elsewhere

// ADD:
@Autowired
private ExecutorServiceAdapter executorServiceAdapter;

// ── getRoboticQList ───────────────────────────────────────────────────
@Override
public List<Map> getRoboticQList() {
    List<Map> res = new ArrayList<>();
    try {
        List<Map<String, String>> raw = 
            executorServiceAdapter.getQueuedRequests();
        for (Map<String, String> entry : raw) {
            Map<String, String> response = new HashMap<>();
            // Matches old format: {"OlaResponseQ": "2026-06-14 08:00:00.0"}
            response.put(entry.get("displayName"), 
                         entry.get("nextFireTime"));
            res.add(response);
        }
    } catch (Exception e) {
        LoggingUtils.logErrorMessage(logger, null, e.getMessage());
    }
    return res;
}

// ── updateDate ────────────────────────────────────────────────────────
@Override
public String updateDate(Object requestData) {
    try {
        String jsonString = new Gson().toJson(requestData, Object.class);
        Map<String, Object> reqObj = 
            ValidationUtils.getStringRequestAsMap(jsonString);
        LoggingUtils.logInfoMessage(logger, reqObj.toString(), 
                                    requestData.toString());

        String queueName      = (String) reqObj.get(
                SchedulerConstants.ROBOTIC_QUEUE_NAME);
        String commandName    = getCommandName(queueName);
        String statusIndicator = (String) reqObj.get("statusIndicator");

        if (statusIndicator.equalsIgnoreCase("OFF")) {
            executorServiceAdapter.cancelRequest(commandName);
            LoggingUtils.logInfoMessage(logger, "OFF Scheduler:", queueName);
            return queueName + " Scheduler Has Been Switch OFF";

        } else {
            // ON path
            if (executorServiceAdapter.isRunning(commandName)) {
                return queueName + " Is already in a running State";
            }
            Date newDate = dateConvert((String) reqObj.get("newTimer"));
            // Cancel existing one-time trigger if present, then reschedule
            executorServiceAdapter.cancelRequest(commandName);
            executorServiceAdapter.scheduleRequest(commandName, newDate,
                    getContextData());
            return queueName + " Has Been Rescheduled At "
                    + reqObj.get("newTimer");
        }

    } catch (Exception e) {
        LoggingUtils.logErrorMessage(logger, null, e.getMessage());
        return "Update Fail";
    }
}

// ── helper ────────────────────────────────────────────────────────────
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

// ── REMOVE these methods entirely (no longer needed): ─────────────────
// getCommandContextObj()
// updateRequestInfo()
// cancelRequestInfo()
// getQList()
```

---

**Verification after first startup:**

```sql
-- Should show 10 rows (one per queue)
SELECT JOB_NAME, JOB_GROUP, DESCRIPTION 
FROM QRTZ_JOB_DETAILS 
WHERE SCHED_NAME = 'CASScheduler';

-- Should show 10 cron triggers
SELECT JOB_NAME, TRIGGER_NAME, CRON_EXPRESSION, NEXT_FIRE_TIME 
FROM QRTZ_CRON_TRIGGERS 
WHERE SCHED_NAME = 'CASScheduler';
```

If those return 10 rows each after startup, the implementation is working correctly and you can test the UI flow end-to-end.

Good catch — the property key is `roboticqueue.olaresponseq.runtime` (no space, all lowercase). Update `SchedulerUtils.java` and `application-dev.properties` accordingly:

---

**`application-dev.properties`** — corrected keys:
```properties
# Quartz - JDBCJobStore (multi-pod safe, Oracle)
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

# Robotic Queue Cron Runtimes (exact keys from old properties)
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
```

---

**`SchedulerUtils.java`** — corrected `@Value` keys:
```java
package com.barclays.api.status.mgmt.schedular.util;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

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

---

Everything else (`CommandExecutorJob`, `ExecutorServiceAdapter`, `SchedulerStartupSeeder`, `SchedulerInfoServiceImpl`, `QuartzConfig`) stays exactly as previously provided. Only these two files needed the key correction.

Also note — `retriggerOnlineapp.runtime=#{300 * 1000}` is already handled separately by `RetriggerOnlineAppsSchedular` using `@Scheduled(fixedRateString)` — that class stays completely untouched.