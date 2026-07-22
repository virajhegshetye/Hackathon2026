I can see exactly what's happening from the logs. Two problems:

**Problem 1:** When the job fires, Jobs Service reads `CALLBACK_ENDPOINT` from `RPAM.JOBS` but then internally reroutes it through `ProcessJobExecutor` — because `ProcessJobDescription` creates a BPMN-process-type job. The executor ignores your `CALLBACK_ENDPOINT` and tries to advance a BPMN process instance instead. That's why your controller never gets called.

**Problem 2:** After execution, Jobs Service does cleanup:
```
update RPAM.jobs set callback_endpoint=?, endpoint=? ... 
delete from RPAM.job_details where id=?
```
It resets `CALLBACK_ENDPOINT` to null as part of its post-execution cleanup — confirming it never used it.

The root cause is `ProcessJobDescription` — it ties the job to `ProcessJobExecutor` which only knows how to advance BPMN processes. No matter what you set in `CALLBACK_ENDPOINT`, this executor type ignores it.

**The fix:** Don't use `ProcessJobDescription` at all. You need to bypass `JobsService.scheduleJob()` entirely and insert directly into both `RPAM.JOBS` and `RPAM.JOB_DETAILS` via JDBC. Jobs Service will then pick up the job from `JOB_DETAILS`, see the `RECIPIENT` CLOB contains an HTTP recipient, and fire the HTTP callback correctly.

Here's what to insert:

```java
private String scheduleCallbackJob(String queueName, Instant fireTime) {

    String jobId = UUID.randomUUID().toString();
    String callbackUrl = serviceUrl + "/scheduler/execute/" + queueName;

    ZonedDateTime fireZdt = ZonedDateTime.ofInstant(fireTime, ZoneId.systemDefault());
    Timestamp fireTimestamp = Timestamp.from(fireTime);

    // ── Step 1: Insert into RPAM.JOBS ─────────────────────────────────
    // This is what getRoboticQList() reads via GraphQL
    bamoeJdbcTemplate.update(
        "INSERT INTO RPAM.JOBS " +
        "(ID, CALLBACK_ENDPOINT, PROCESS_ID, EXPIRATION_TIME, STATUS, " +
        "PRIORITY, EXECUTION_COUNTER, RETRIES, LAST_UPDATE) " +
        "VALUES (?, ?, ?, ?, 'SCHEDULED', 1, 0, 0, SYSTIMESTAMP)",
        jobId,
        callbackUrl,
        queueName,
        fireTimestamp
    );

    // ── Step 2: Insert into RPAM.JOB_DETAILS ──────────────────────────
    // This is what Jobs Service scheduler actually reads to fire the job.
    // RECIPIENT must be JSON matching HttpRecipient structure.
    // JOB_TRIGGER must be JSON matching TimerSchedule structure.
    // Jobs Service polls JOB_DETAILS and fires based on FIRE_TIME.

    String recipientJson = "{"
        + "\"classType\":\"org.kie.kogito.jobs.service.api.recipient.http.HttpRecipient\","
        + "\"url\":\"" + callbackUrl + "\","
        + "\"method\":\"POST\","
        + "\"headers\":{\"Content-Type\":\"application/json\"},"
        + "\"queryParams\":{},"
        + "\"payload\":{"
        +   "\"classType\":\"org.kie.kogito.jobs.service.api.recipient.http.HttpRecipientStringPayloadData\","
        +   "\"data\":\"{\\\"correlationId\\\":\\\"" + jobId + "\\\"}\""
        + "}"
        + "}";

    String triggerJson = "{"
        + "\"classType\":\"org.kie.kogito.jobs.service.api.schedule.timer.TimerSchedule\","
        + "\"startTime\":\"" + fireZdt.toOffsetDateTime()
                                      .truncatedTo(ChronoUnit.MILLIS)
                                      .toString() + "\","
        + "\"repeatCount\":0,"
        + "\"delay\":0,"
        + "\"delayUnit\":\"MILLIS\""
        + "}";

    bamoeJdbcTemplate.update(
        "INSERT INTO RPAM.JOB_DETAILS " +
        "(ID, CORRELATION_ID, STATUS, LAST_UPDATE, RETRIES, EXECUTION_COUNTER, " +
        "PRIORITY, RECIPIENT, JOB_TRIGGER, FIRE_TIME, CREATED) " +
        "VALUES (?, ?, 'SCHEDULED', SYSTIMESTAMP AT TIME ZONE 'UTC', " +
        "0, 0, 1, ?, ?, ? AT TIME ZONE 'UTC', SYSTIMESTAMP AT TIME ZONE 'UTC')",
        jobId,
        queueName,   // correlationId = queueName so you can find it later
        recipientJson,
        triggerJson,
        fireTimestamp
    );

    LoggingUtils.logInfoMessage(LOGGER, null,
        "Inserted job directly: queue=" + queueName
        + " jobId=" + jobId
        + " callbackUrl=" + callbackUrl
        + " fireTime=" + fireTime);

    return jobId;
}
```

Also update `cancelAllActiveJobs()` — since you're no longer using `JobsService.scheduleJob()`, cancel needs to delete from both tables:

```java
private int cancelAllActiveJobs(String queueName) {
    List<Job> jobs = schedulerQueryService.getJobsByProcessId(queueName);
    int count = 0;

    for (Job job : jobs) {
        if (job.getStatus() == null) continue;

        if (ACTIVE_STATUSES.contains(job.getStatus().toUpperCase())) {
            try {
                // Cancel via JobsService first (handles internal scheduler state)
                boolean cancelled = jobsService.cancelJob(job.getId());
                if (!cancelled) {
                    schedulerQueryService.cancelJob(job.getId());
                }
            } catch (Exception e) {
                schedulerQueryService.cancelJob(job.getId());
            }

            // Always clean both tables directly to be safe
            try {
                bamoeJdbcTemplate.update(
                    "DELETE FROM RPAM.JOB_DETAILS WHERE ID = ?", job.getId());
            } catch (Exception e) {
                LoggingUtils.logWarnMessage(LOGGER, null,
                    "JOB_DETAILS delete failed for jobId=" + job.getId()
                    + ": " + e.getMessage());
            }

            try {
                bamoeJdbcTemplate.update(
                    "UPDATE RPAM.JOBS SET STATUS = 'CANCELED' WHERE ID = ?",
                    job.getId());
            } catch (Exception e) {
                LoggingUtils.logWarnMessage(LOGGER, null,
                    "JOBS status update failed for jobId=" + job.getId()
                    + ": " + e.getMessage());
            }

            count++;
        }
    }
    return count;
}
```

**One thing to verify first** — check what `RECIPIENT` and `JOB_TRIGGER` JSON looks like in an existing working row in `JOB_DETAILS`. Run this query and share the output:

```sql
SELECT ID, CORRELATION_ID, STATUS, 
       DBMS_LOB.SUBSTR(RECIPIENT, 500, 1) AS RECIPIENT,
       DBMS_LOB.SUBSTR(JOB_TRIGGER, 500, 1) AS JOB_TRIGGER,
       FIRE_TIME
FROM RPAM.JOB_DETAILS
WHERE ROWNUM <= 3;
```

This will confirm the exact JSON format Jobs Service expects in those CLOB columns before we commit to this approach. The `classType` field in the JSON is what Jobs Service uses to deserialize — if it differs from what I have above, the job will still not fire correctly.