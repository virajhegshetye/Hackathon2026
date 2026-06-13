Excellent — now the full picture is clear. The old properties reveal something critical:

**The runtimes are CRON expressions, not one-time dates:**
```properties
roboticqueue.olarespons eq.runtime=0 0 * * * ?     # every hour at :00
roboticqueue.nointerfaceq.runtime=0 10 * * * ?     # every hour at :10
roboticqueue.ntcholdq.runtime=0 0 0/4 * * ?        # every 4 hours
roboticqueue.ngcback.runtime=0 0 0/2 * * ?         # every 2 hours
roboticqueue.logCleanup.runtime=0 0 23 * * ?       # daily at 23:00
roboticqueue.caslssq.runtime=0 30 0/4 * * ?        # every 4 hours at :30
roboticqueue.nodeq.runtime=0 30 0/2 * * ?          # every 2 hours at :30
roboticqueue.nouploadq.runtime=0 30 0/4 * * ?      # every 4 hours at :30
roboticqueue.batchResubmitq.runtime=0 40 0/4 * * ? # every 4 hours at :40
roboticqueue.lapsedProcess.runtime=0 0 8 * * ?     # daily at 08:00
retriggerOnlineapp.runtime=#{300 * 1000}           # fixed rate (separate Spring scheduler)
```

And `RetriggerOnlineAppsSchedular` already uses `@Scheduled` with `fixedRateString` — that's a **separate Spring scheduler, not part of the executor** — keep it as-is.

---

**This changes the Quartz design significantly.** The queues are **recurring cron jobs**, not one-time scheduled jobs. The UI lets operators override the *next fire time* for a specific run — but the underlying schedule is cron-based.

**Revised complete architecture:**

```
Startup → seed all 13 queues as Quartz CronTriggers (from properties)
UI getRoboticQList → reads QRTZ_TRIGGERS.NEXT_FIRE_TIME for each job
UI updateRoboticQTimer (ON) → reschedule next fire as one-time SimpleTrigger override
UI updateRoboticQTimer (OFF) → pause the trigger (not delete)
After one-time fire → trigger reverts to cron (or operator reschedules again)
```

---

Here is the **complete implementation** — all files:

---

**1. `application-dev.properties` additions:**

```properties
# Quartz JDBCJobStore (multi-pod safe)
spring.quartz.job-store-type=jdbc
spring.quartz.jdbc.initialize-schema=never
spring.quartz.properties.org.quartz.jobStore.class=org.quartz.impl.jdbcjobstore.JobStoreTX
spring.quartz.properties.org.quartz.jobStore.driverDelegateClass=org.quartz.impl.jdbcjobstore.oracle.OracleDelegate
spring.quartz.properties.org.quartz.jobStore.tablePrefix=QRTZ_
spring.quartz.properties.org.quartz.jobStore.isClustered=true
spring.quartz.properties.org.quartz.scheduler.instanceId=AUTO
spring.quartz.properties.org.quartz.scheduler.instanceName=CASScheduler

# Robotic Queue Cron Runtimes (migrated from old properties)
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

---

**2. `build.gradle` addition:**

```groovy
implementation 'org.springframework.boot:spring-boot-starter-quartz'
```

---

**3. `SchedulerUtils.java`** (new — holds property values, mirrors old `SchedulerUtils`):

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

    public String getOlaresponseqRuntime()    { return olaresponseeqRuntime; }
    public String getNointerfaceqRuntime()    { return nointerfaceqRuntime; }
    public String getNtcholdqRuntime()        { return ntcholdqRuntime; }
    public String getNgcbackRuntime()         { return ngcbackRuntime; }
    public String getLogCleanupRuntime()      { return logCleanupRuntime; }
    public String getCaslssqRuntime()         { return caslssqRuntime; }
    public String getNodeqRuntime()           { return nodeqRuntime; }
    public String getNouploadqRuntime()       { return nouploadqRuntime; }
    public String getBatchResubmitqRuntime()  { return batchResubmitqRuntime; }
    public String getLapsedProcessRuntime()   { return lapsedProcessRuntime; }
}
```

---

**4. `CommandExecutorJob.java`** (Quartz Job — invokes your existing Command beans):

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
public class CommandExecutorJob implements Job {

    // Key stored in JobDataMap identifying which queue this job serves
    public static final String QUEUE_COMMAND_KEY = "commandName";

    @Autowired
    private QSchedulerService qSchedulerService;

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        String commandName = context.getMergedJobDataMap()
                                    .getString(QUEUE_COMMAND_KEY);
        String queueName = toQueueName(commandName);
        log.info("CommandExecutorJob triggered for commandName={}, queueName={}", 
                 commandName, queueName);
        try {
            qSchedulerService.trigger(queueName);
            log.info("CommandExecutorJob completed for queueName={}", queueName);
        } catch (Exception e) {
            log.error("CommandExecutorJob failed for queueName={}", queueName, e);
            throw new JobExecutionException(e);
        }
    }

    // Maps full class name → queue name constant used in QSchedulerServiceImpl
    private String toQueueName(String commandName) {
        if (commandName.contains("OlaResponseQ"))      return SchedulerConstants.OLARESPONSEQ;
        if (commandName.contains("NoInterfaceQ"))      return SchedulerConstants.NOINTERFACEQ;
        if (commandName.contains("NTCHOLDQ") 
         || commandName.contains("NTCHoldQ"))          return SchedulerConstants.NTCHOLDQ;
        if (commandName.contains("NGCBck") 
         || commandName.contains("NGCBack"))           return "NGCBack";
        if (commandName.contains("LogCleanUp"))        return "LogCleanUp";
        if (commandName.contains("CASLSSQ"))           return SchedulerConstants.CASLSSQ;
        if (commandName.contains("NODEQ"))             return SchedulerConstants.NODEQ;
        if (commandName.contains("NoUploadQ"))         return SchedulerConstants.NOUPLOADQ;
        if (commandName.contains("BatchResubmitQ"))    return SchedulerConstants.BATCHRESUBMITQ;
        if (commandName.contains("LapsedProcess"))     return "LapsedProcess";
        if (commandName.contains("FailedSoftToHard"))  return "FailedSoftToHard";
        if (commandName.contains("ApplicationExpiry")) return "ApplicationExpiryProcess";
        if (commandName.contains("AdvancedLogCleanUp"))return "AdvancedLogCleanUp";
        if (commandName.contains("SMSRetry"))          return "SMSRetry";
        return commandName;
    }
}
```

---

**5. `ExecutorServiceAdapter.java`** (complete — exact drop-in for `kieExecutorService`):

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

    public static final String JOB_GROUP = "BAMOE_SCHEDULER";
    public static final String TRIGGER_GROUP = "BAMOE_TRIGGER";
    // Suffix to distinguish one-time override triggers from cron triggers
    public static final String CRON_TRIGGER_SUFFIX = "_CRON";
    public static final String ONCE_TRIGGER_SUFFIX = "_ONCE";

    @Autowired
    private Scheduler quartzScheduler;

    /**
     * Replaces: kieExecutorService.scheduleRequest(commandName, date, ctx)
     * Schedules a one-time fire at the given date, overriding cron.
     */
    public void scheduleRequest(String commandName, Date fireTime,
                                 Map<String, Object> contextData) throws SchedulerException {
        String jobKey = toJobKey(commandName);
        JobKey jk = JobKey.jobKey(jobKey, JOB_GROUP);

        // Remove any existing one-time override trigger first
        TriggerKey onceTk = TriggerKey.triggerKey(jobKey + ONCE_TRIGGER_SUFFIX, TRIGGER_GROUP);
        if (quartzScheduler.checkExists(onceTk)) {
            quartzScheduler.unscheduleJob(onceTk);
        }

        // If job doesn't exist yet (first-time schedule, not seeded via cron)
        if (!quartzScheduler.checkExists(jk)) {
            JobDataMap dataMap = new JobDataMap();
            dataMap.put(CommandExecutorJob.QUEUE_COMMAND_KEY, commandName);
            if (contextData != null) dataMap.putAll(contextData);

            JobDetail job = JobBuilder.newJob(CommandExecutorJob.class)
                    .withIdentity(jk)
                    .usingJobData(dataMap)
                    .storeDurably(true)
                    .build();
            quartzScheduler.addJob(job, false);
        }

        // Schedule one-time trigger
        Trigger onceTrigger = TriggerBuilder.newTrigger()
                .withIdentity(onceTk)
                .forJob(jk)
                .startAt(fireTime)
                .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                        .withMisfireHandlingInstructionFireNow())
                .build();

        quartzScheduler.scheduleJob(onceTrigger);
        log.info("Scheduled one-time job {} at {}", jobKey, fireTime);
    }

    /**
     * Replaces: kieExecutorService.cancelRequest(id)
     * Pauses/removes the one-time override trigger for this queue.
     */
    public boolean cancelRequest(String commandName) throws SchedulerException {
        String jobKey = toJobKey(commandName);
        TriggerKey onceTk = TriggerKey.triggerKey(
                jobKey + ONCE_TRIGGER_SUFFIX, TRIGGER_GROUP);
        TriggerKey cronTk = TriggerKey.triggerKey(
                jobKey + CRON_TRIGGER_SUFFIX, TRIGGER_GROUP);

        boolean cancelled = false;
        if (quartzScheduler.checkExists(onceTk)) {
            quartzScheduler.unscheduleJob(onceTk);
            cancelled = true;
        }
        if (quartzScheduler.checkExists(cronTk)) {
            quartzScheduler.pauseTrigger(cronTk);
            cancelled = true;
        }
        log.info("Cancelled/paused triggers for job {}", jobKey);
        return cancelled;
    }

    /**
     * Replaces: kieExecutorService.getQueuedRequests(QueryContext)
     * Returns list of {queueDisplayName -> nextFireTime} for UI table.
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
                // Only include NORMAL (active) triggers
                if (state == Trigger.TriggerState.NORMAL 
                 || state == Trigger.TriggerState.WAITING) {
                    Date tf = t.getNextFireTime();
                    // Pick earliest next fire (one-time override takes priority)
                    if (tf != null && (nextFire == null || tf.before(nextFire))) {
                        nextFire = tf;
                    }
                }
            }

            if (nextFire != null) {
                JobDetail detail = quartzScheduler.getJobDetail(jk);
                String commandName = detail.getJobDataMap()
                        .getString(CommandExecutorJob.QUEUE_COMMAND_KEY);
                // Strip "ServiceImpl" → display name matching old UI
                String displayName = jk.getName().replace("ServiceImpl", "");

                Map<String, String> entry = new LinkedHashMap<>();
                entry.put("commandName", commandName);
                entry.put("displayName", displayName);
                // Format matches old RequestInfo.getTime().toString()
                entry.put("nextFireTime", nextFire.toString());
                result.add(entry);
            }
        }

        // Sort by nextFireTime ascending (matches old behaviour)
        result.sort(Comparator.comparing(m -> m.get("nextFireTime")));
        return result;
    }

    /**
     * Replaces: getRequestsByCommand(commandName, STATUS.QUEUED)
     */
    public boolean isQueued(String commandName) throws SchedulerException {
        String jobKey = toJobKey(commandName);
        for (String suffix : Arrays.asList(ONCE_TRIGGER_SUFFIX, CRON_TRIGGER_SUFFIX)) {
            TriggerKey tk = TriggerKey.triggerKey(jobKey + suffix, TRIGGER_GROUP);
            if (quartzScheduler.checkExists(tk)) {
                Trigger.TriggerState s = quartzScheduler.getTriggerState(tk);
                if (s == Trigger.TriggerState.NORMAL 
                 || s == Trigger.TriggerState.WAITING) return true;
            }
        }
        return false;
    }

    /**
     * Replaces: getRequestsByCommand(commandName, STATUS.RUNNING)
     */
    public boolean isRunning(String commandName) throws SchedulerException {
        String jobKey = toJobKey(commandName);
        for (JobExecutionContext ctx : quartzScheduler.getCurrentlyExecutingJobs()) {
            if (ctx.getJobDetail().getKey().getName().equals(jobKey)) return true;
        }
        return false;
    }

    /**
     * Seeds a recurring cron job on startup.
     * Called by SchedulerStartupSeeder for each queue.
     */
    public void seedCronJob(String commandName, String cronExpression) 
            throws SchedulerException {
        String jobKey = toJobKey(commandName);
        JobKey jk = JobKey.jobKey(jobKey, JOB_GROUP);
        TriggerKey cronTk = TriggerKey.triggerKey(
                jobKey + CRON_TRIGGER_SUFFIX, TRIGGER_GROUP);

        if (quartzScheduler.checkExists(jk)) {
            log.info("Job {} already exists in QRTZ_JOB_DETAILS, skipping seed", jobKey);
            return;
        }

        JobDataMap dataMap = new JobDataMap();
        dataMap.put(CommandExecutorJob.QUEUE_COMMAND_KEY, commandName);

        JobDetail job = JobBuilder.newJob(CommandExecutorJob.class)
                .withIdentity(jk)
                .usingJobData(dataMap)
                .storeDurably(true)
                .build();

        CronTrigger cronTrigger = TriggerBuilder.newTrigger()
                .withIdentity(cronTk)
                .forJob(jk)
                .withSchedule(CronScheduleBuilder
                        .cronSchedule(cronExpression)
                        .withMisfireHandlingInstructionDoNothing())
                .build();

        quartzScheduler.scheduleJob(job, cronTrigger);
        log.info("Seeded cron job {} with expression {}", jobKey, cronExpression);
    }

    public String toJobKey(String commandName) {
        // Full class name → short name e.g. "OlaResponseQServiceImpl"
        return commandName.contains(".") 
            ? commandName.substring(commandName.lastIndexOf('.') + 1) 
            : commandName;
    }
}
```

---

**6. `SchedulerStartupSeeder.java`** (replaces `Reoccurring.getScheduleTime()`):

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
        log.info("SchedulerStartupSeeder: seeding all queue cron jobs...");
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
        log.info("SchedulerStartupSeeder: seeding complete.");
    }

    private void seed(String commandName, String cronExpression) {
        try {
            adapter.seedCronJob(commandName, cronExpression);
        } catch (Exception e) {
            log.error("Failed to seed cron job for {}: {}", commandName, e.getMessage(), e);
        }
    }
}
```

---

**7. `SchedulerInfoServiceImpl.java`** — only the changed methods (rest stays identical):

```java
// REMOVE this field:
// @Autowired
// private ExecutorService kieExecutorService;

// ADD this field:
@Autowired
private ExecutorServiceAdapter executorServiceAdapter;

// ─── getRoboticQList ────────────────────────────────────────────────
@Override
public List<Map> getRoboticQList() {
    List<Map> res = new ArrayList<>();
    try {
        List<Map<String, String>> raw = executorServiceAdapter.getQueuedRequests();
        for (Map<String, String> entry : raw) {
            Map<String, String> response = new HashMap<>();
            response.put(entry.get("displayName"), entry.get("nextFireTime"));
            res.add(response);
        }
    } catch (Exception e) {
        LoggingUtils.logErrorMessage(logger, null, e.getMessage());
    }
    return res;
}

// ─── updateDate ─────────────────────────────────────────────────────
@Override
public String updateDate(Object requestData) {
    try {
        String jsonString = new Gson().toJson(requestData, Object.class);
        Map<String, Object> reqObj = ValidationUtils.getStringRequestAsMap(jsonString);
        LoggingUtils.logInfoMessage(logger, reqObj.toString(), requestData.toString());

        String queueName = (String) reqObj.get(SchedulerConstants.ROBOTIC_QUEUE_NAME);
        String commandName = getCommandName(queueName);
        String statusIndicator = (String) reqObj.get("statusIndicator");

        if (statusIndicator.equalsIgnoreCase("OFF")) {
            // Replaces: cancelRequestInfo(reqList, reqObj)
            executorServiceAdapter.cancelRequest(commandName);
            LoggingUtils.logInfoMessage(logger, "OFF Scheduler:", queueName);
            return queueName + " Scheduler Has Been Switch OFF";

        } else {
            // ON path
            if (executorServiceAdapter.isRunning(commandName)) {
                return queueName + " Is already in a running State";
            }

            Date newDate = dateConvert((String) reqObj.get("newTimer"));

            // Replaces: cancelRequest + scheduleRequest pattern
            // cancelRequest first to remove any existing one-time trigger
            executorServiceAdapter.cancelRequest(commandName);
            // Then schedule new one-time fire
            executorServiceAdapter.scheduleRequest(commandName, newDate, 
                getContextData());

            return queueName + " Has Been Rescheduled At " + reqObj.get("newTimer");
        }

    } catch (Exception e) {
        LoggingUtils.logErrorMessage(logger, null, e.getMessage());
        return "Update Fail";
    }
}

// ─── helper: build context data map ─────────────────────────────────
private Map<String, Object> getContextData() {
    Map<String, Object> data = new HashMap<>();
    data.put(RECORDS_PER_TRANSACTION, recordsPerTransaction);
    data.put(SchedulerConstants.ENABLE_CLEANUP, isCleanupEnabled);
    data.put(CASES_PER_ROUND, casesPerRound);
    data.put(REQUEST_INFO_RECORD_COUNT, requestInfoRecordsPerTransaction);
    data.put(REQUEST_INFO_THRESHOLD, requestInfoRecordsThreshold);
    data.put("processInstanceThresholdValue", processInstanceThresholdValue);
    data.put("singleRunProcessInstanceThreshold", singleRunProcessInstanceThreshold);
    data.put("processInstanceIdPerTransaction", processInstanceIdPerTransaction);
    data.put("purgeSchedulerThreadIntervals", purgeSchedulerThreadIntervals);
    data.put("purgeSchedulerThreads", purgeSchedulerThreads);
    data.put("processInstanceRecordsDeleted", 0);
    return data;
}

// Remove these methods entirely (no longer needed):
// getCommandContextObj(), updateRequestInfo(), cancelRequestInfo(), getQList()
```

---

**Files untouched:** `HomeScreenController`, `QSchedulerService`, `QSchedulerServiceImpl`, all `*ServiceImpl` Command classes, `SchedulerConstants`, `RetriggerOnlineAppsSchedular`.

**Quartz tables needed in Oracle** — run this check first:
```sql
SELECT TABLE_NAME FROM ALL_TABLES 
WHERE TABLE_NAME LIKE 'QRTZ_%' 
AND OWNER = 'RPAM';
```
If tables don't exist, share that and I'll give you the Oracle DDL. If they do exist from the old BAMOE 8 deployment, they're ready to use.