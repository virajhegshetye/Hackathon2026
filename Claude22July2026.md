Now I have the complete picture. The key insight from `JobCallbackResourceDef` is that the new BAMOE 9.x Jobs Service API uses `org.kie.kogito.jobs.service.api.Job` (not `org.kie.kogito.jobs.api.Job`) with a `HttpRecipient` + `TimerSchedule` pattern — not `callbackEndpoint` directly.

The `JobsService.scheduleJob()` you're using expects this newer `Job` format. That's why `JobBuilder` from `org.kie.kogito.jobs.api` isn't working either — it's the old API shape.

Replace your `scheduleCallbackJob()` with this:

```java
// IMPORTS to add at top of SchedulerInfoServiceImpl
import com.fasterxml.jackson.databind.ObjectMapper;
import org.kie.kogito.jobs.service.api.Job;
import org.kie.kogito.jobs.service.api.JobBuilder;
import org.kie.kogito.jobs.service.api.recipient.http.HttpRecipient;
import org.kie.kogito.jobs.service.api.recipient.http.HttpRecipientJsonPayloadData;
import org.kie.kogito.jobs.service.api.schedule.timer.TimerSchedule;
import org.kie.kogito.jobs.service.api.TemporalUnit;
import org.kie.kogito.jobs.api.JobCallbackPayload;
import java.time.OffsetDateTime;
import java.time.temporal.ChronoUnit;
```

```java
// ADD ObjectMapper to constructor injection
private final ObjectMapper objectMapper;

public SchedulerInfoServiceImpl(
        JobsService jobsService,
        SchedulerQueryServiceImpl schedulerQueryService,
        ObjectMapper objectMapper) {
    this.jobsService = jobsService;
    this.schedulerQueryService = schedulerQueryService;
    this.objectMapper = objectMapper;
}
```

```java
// REPLACE scheduleCallbackJob() entirely
private String scheduleCallbackJob(String queueName, Instant fireTime) {

    String jobId = UUID.randomUUID().toString();
    String callbackUrl = serviceUrl + "/scheduler/execute/" + queueName;

    OffsetDateTime startTime = ZonedDateTime
            .ofInstant(fireTime, ZoneId.systemDefault())
            .toOffsetDateTime()
            .truncatedTo(ChronoUnit.MILLIS);

    // Build payload - correlationId identifies this job
    JobCallbackPayload payload = new JobCallbackPayload(jobId);

    // Build HTTP recipient - this populates RECIPIENT column in JOB_DETAILS
    HttpRecipient<HttpRecipientJsonPayloadData> recipient =
            HttpRecipient.builder()
                    .forJsonPayload()
                    .url(callbackUrl)
                    .method("POST")
                    .header("Content-Type", "application/json")
                    .payload(HttpRecipientJsonPayloadData.from(
                            objectMapper.valueToTree(payload)))
                    .build();

    // Build schedule - one-time fire at the given time
    TimerSchedule schedule = TimerSchedule.builder()
            .startTime(startTime)
            .repeatCount(0)      // 0 = fire once
            .delay(0L)
            .delayUnit(TemporalUnit.MILLIS)
            .build();

    // Build the Job using the new service API
    Job job = Job.builder()
            .id(jobId)
            .correlationId(jobId)
            .recipient(recipient)
            .schedule(schedule)
            .build();

    LoggingUtils.logInfoMessage(LOGGER, null,
            "Scheduling job for queue=" + queueName
            + " callbackUrl=" + callbackUrl
            + " fireTime=" + fireTime
            + " jobId=" + jobId);

    String scheduledId = jobsService.scheduleJob(job);

    if (scheduledId == null || scheduledId.isBlank()) {
        throw new RuntimeException(
                "JobsService returned null jobId for queue=" + queueName);
    }

    LoggingUtils.logInfoMessage(LOGGER, null,
            "Job scheduled successfully queue=" + queueName
            + " jobId=" + scheduledId);

    return scheduledId;
}
```

Also **remove** `RestClient restClient` and `@Value("http://0.0.0.0:8082") private String serviceUrl` from the class — keep `serviceUrl` but change the annotation:

```java
// CHANGE from hardcoded:
@Value("http://0.0.0.0:8082")
private String serviceUrl;

// TO property-based:
@Value("${kogito.service.url}")
private String serviceUrl;
```

The reason this works: `JobCallbackResourceDef.buildCallbackPatternJob()` shows exactly how BAMOE 9.x builds jobs internally — `Job.builder()` from `org.kie.kogito.jobs.service.api` with `HttpRecipient` + `TimerSchedule`. The `RECIPIENT` column in `JOB_DETAILS` stores the `HttpRecipient` as JSON (including your callback URL), and `JOB_TRIGGER` stores the `TimerSchedule`. When the job fires, Jobs Service reads `RECIPIENT.url` and POSTs to it — which hits your `SchedulerCallbackController`.

Everything else in your `SchedulerInfoServiceImpl` stays exactly as it is.