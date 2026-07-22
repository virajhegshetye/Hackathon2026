Confirmed. `ProcessJobDescription` has no callback field at all — it only carries `id`, `expirationTime`, `priority`, `processId`. There is no way to set `CALLBACK_ENDPOINT` through this API.

The cleanest solution given your setup: schedule via `ProcessJobDescription` as you already have working, then immediately update `CALLBACK_ENDPOINT` in the `JOBS` table directly via JDBC. You have direct access to RPAM schema already.

Replace `scheduleCallbackJob()` with this:

```java
// ADD to constructor - inject bamoe JdbcTemplate
private final JdbcTemplate bamoeJdbcTemplate;

public SchedulerInfoServiceImpl(
        JobsService jobsService,
        SchedulerQueryServiceImpl schedulerQueryService,
        @Qualifier("bamoeJdbcTemplate") JdbcTemplate bamoeJdbcTemplate) {
    this.jobsService = jobsService;
    this.schedulerQueryService = schedulerQueryService;
    this.bamoeJdbcTemplate = bamoeJdbcTemplate;
    // remove RestClient construction - no longer needed
}
```

```java
private String scheduleCallbackJob(String queueName, Instant fireTime) {

    String callbackUrl = serviceUrl + "/scheduler/execute/" + queueName;

    // Step 1: Schedule via JobsService - this inserts into JOBS table
    // CALLBACK_ENDPOINT will be null at this point
    ProcessJobDescription jobDescription = ProcessJobDescription.of(
            ExactExpirationTime.of(
                    ZonedDateTime.ofInstant(fireTime, ZoneId.systemDefault())
            ),
            1,
            queueName
    );

    String jobId = jobsService.scheduleJob(jobDescription);

    if (jobId == null || jobId.isBlank()) {
        throw new RuntimeException(
                "JobsService returned null jobId for queue=" + queueName);
    }

    LoggingUtils.logInfoMessage(LOGGER, null,
            "Job inserted for queue=" + queueName + " jobId=" + jobId
            + " - now setting CALLBACK_ENDPOINT");

    // Step 2: Immediately patch CALLBACK_ENDPOINT in JOBS table
    // JobsService inserts the row but provides no API to set callbackEndpoint
    // Direct JDBC update is the only available path
    int updated = bamoeJdbcTemplate.update(
            "UPDATE RPAM.JOBS SET CALLBACK_ENDPOINT = ? WHERE ID = ?",
            callbackUrl,
            jobId
    );

    if (updated == 0) {
        LoggingUtils.logErrorMessage(LOGGER, null,
                "CALLBACK_ENDPOINT update affected 0 rows for jobId=" + jobId
                + " - job may not have committed yet");
    } else {
        LoggingUtils.logInfoMessage(LOGGER, null,
                "CALLBACK_ENDPOINT set successfully for queue=" + queueName
                + " jobId=" + jobId + " url=" + callbackUrl);
    }

    return jobId;
}
```

Also remove `RestClient restClient` field and its construction from the class entirely — no longer needed.

The `@Value("${kogito.service.url}") private String serviceUrl` stays as is.

This works because `JobsService.scheduleJob()` commits the row to `RPAM.JOBS` synchronously before returning the `jobId`, so the `UPDATE` immediately after will always find the row. When the job fires, Jobs Service reads `CALLBACK_ENDPOINT` from that row and POSTs to it — hitting your `SchedulerCallbackController`.

What `bamoeJdbcTemplate` qualifier name are you using in your project? Check your `StatusBusinessDbConfig` or `DevBamoeDataSourceConfig` — share the bean name if unsure and I'll adjust the qualifier.