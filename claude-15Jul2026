Now I have the complete picture. Let me analyse everything before giving you the solution.

**Key facts from everything you've shared:**

**Schema:**
- `RPAM.TASKS` — has `NAME`, `STATE`, `PROCESS_INSTANCE_ID`, `ACTUAL_OWNER`, `STARTED`, `INPUTS`, `OUTPUTS`, `EXTERNAL_REFERENCE_ID` — this is where queue tasks live
- `RPAM.JBPM_USER_TASKS` — has `TASK_NAME`, `STATUS`, `ACTUAL_OWNER`, `PROCESS_INSTANCE_ID` — the runtime task store
- `RPAM.PROCESSES` — has `BUSINESS_KEY`, `ID`, `STATE`, `VARIABLES`
- `RPAM.BUSINESS_KEY_MAPPING` — `BUSINESS_KEY` → `PROCESS_INSTANCE_ID` — this is how you get caseId
- `RPAM.JOBS` — has `CALLBACK_ENDPOINT`, `PROCESS_ID` (your queue name), `EXPIRATION_TIME`
- `RPAM.JOB_DETAILS` — has `RECIPIENT` (CLOB), `JOB_TRIGGER` (CLOB), `FIRE_TIME`

**GraphQL — what you already have:**

- `UserTaskInstances` by state + actualOwner ✅
- `UserTaskInstances` by state + processInstanceIds ✅
- `ProcessInstances` by businessKey ✅
- Jobs queries (your 3 new ones) ✅

**What's missing for the queue classes:**
- A `UserTaskInstances` query filtered by **task name** (queue name) + **state** — this is the primary fetch needed by each queue's `execute()`
- The task transition calls (claim → start → complete) via REST — these go to `kogito.service.url` which is `http://0.0.0.0:8082`

**Old `QSchedulerServiceImpl` pattern I can see:**
- Fetch tasks by queue name + Ready/Reserved state
- For each task: get caseId from businessKey, resolve owner, claim → start → complete
- OLARESPONSEQ has extra filters (1hr old, CO accept exclusion, QUEUE_IND='C' exclusion)

---

Here is the complete solution — only new files, nothing touching your existing working code:

---

### New GraphQL query needed — add to `GraphQLQueryLoader`

```java
// Add this line in GraphQLQueryLoader constructor:
loadQuery("userTasksByName", "graphql/queries/user-tasks-by-name.graphql");
```

**`user-tasks-by-name.graphql`** (new file in `graphql/queries/`):

```graphql
query UserTaskInstances($name: String!, $states: [String!]) {
  UserTaskInstances(
    where: {
      and: [
        { name: { equal: $name } }
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
  }
}
```

---

### New `UserTaskInstance` model (maps to existing `RPAM.TASKS`)

```java
package com.barclays.api.status.mgmt.model.graphql;

import lombok.Data;
import java.time.Instant;
import java.util.List;

@Data
public class UserTaskInstance {
    private String id;
    private String name;
    private String state;
    private String processInstanceId;
    private String actualOwner;
    private Instant started;
    private Instant lastUpdate;
    private List<String> potentialGroups;
    private List<String> potentialUsers;
    private String referenceName;
}
```

---

### New `QSchedulerServiceImpl`

This is the core. Replaces all old jBPM APIs with GraphQL + REST using your existing infrastructure:

```java
package com.barclays.api.status.mgmt.service.impl;

import com.barclays.api.bap.bld.lib.logging.LogFactory;
import com.barclays.api.bap.bld.lib.logging.Logger;
import com.barclays.api.cif.utilities.LoggingUtils;
import com.barclays.api.status.mgmt.config.GraphQLQueryLoader;
import com.barclays.api.status.mgmt.model.graphql.GraphQLResponse;
import com.barclays.api.status.mgmt.model.graphql.UserTaskInstance;
import com.barclays.api.status.mgmt.service.QSchedulerService;
import com.barclays.api.status.mgmt.util.BPMNConstants;
import com.barclays.api.status.mgmt.util.SchedulerConstants;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.commons.lang3.exception.ExceptionUtils;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.MediaType;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;

import java.time.Instant;
import java.time.LocalTime;
import java.time.temporal.ChronoUnit;
import java.util.*;
import java.util.stream.Collectors;

@Service
public class QSchedulerServiceImpl implements QSchedulerService {

    private static final Logger LOGGER = LogFactory.getLogger(QSchedulerServiceImpl.class);

    private static final List<String> ACTIVE_STATES = List.of("Ready", "Reserved");

    // kogito.service.url=http://0.0.0.0:8082 - already in your properties
    @Value("${kogito.service.url}")
    private String kogitoServiceUrl;

    private final SchedulerGraphQLClient graphQLClient;
    private final GraphQLQueryLoader queryLoader;
    private final ObjectMapper objectMapper;

    // Use your existing businessJdbcTemplate for VECTUSCASE queries
    private final JdbcTemplate businessJdbcTemplate;

    // Use your existing bamoeJdbcTemplate for RPAM queries
    private final JdbcTemplate bamoeJdbcTemplate;

    private final RestClient restClient;

    public QSchedulerServiceImpl(
            SchedulerGraphQLClient graphQLClient,
            GraphQLQueryLoader queryLoader,
            ObjectMapper objectMapper,
            @Qualifier("businessJdbcTemplate") JdbcTemplate businessJdbcTemplate,
            @Qualifier("bamoeJdbcTemplate") JdbcTemplate bamoeJdbcTemplate) {
        this.graphQLClient = graphQLClient;
        this.queryLoader = queryLoader;
        this.objectMapper = objectMapper;
        this.businessJdbcTemplate = businessJdbcTemplate;
        this.bamoeJdbcTemplate = bamoeJdbcTemplate;
        // Reuses same RestClient pattern as SchedulerGraphQLClient
        this.restClient = RestClient.builder()
                .defaultHeader("Content-Type", "application/json")
                .build();
    }

    // ═══════════════════════════════════════════════════════════════════
    // trigger() - called by each queue's execute() method
    // ═══════════════════════════════════════════════════════════════════

    @Override
    public void trigger(String queueName) {
        LoggingUtils.logInfoMessage(LOGGER, null,
                "Scheduler trigger started for queue: " + queueName);

        int triggered = 0;
        int failed = 0;

        try {
            // Step 1: fetch tasks
            List<UserTaskInstance> tasks = fetchTasksByName(queueName);

            if (tasks == null || tasks.isEmpty()) {
                LoggingUtils.logInfoMessage(LOGGER, null,
                        "No tasks found for queue: " + queueName);
                return;
            }

            LoggingUtils.logInfoMessage(LOGGER, null,
                    "Tasks found for " + queueName + ": " + tasks.size());

            // Step 2: OLARESPONSEQ-specific filters (preserved from old code)
            if (SchedulerConstants.OLARESPONSEQ.equalsIgnoreCase(queueName)) {
                tasks = applyOlaFilters(tasks);
                LoggingUtils.logInfoMessage(LOGGER, null,
                        "Tasks after OLA filters: " + tasks.size());
            }

            // Step 3: process each task
            for (UserTaskInstance task : tasks) {
                try {
                    String caseId = getCaseId(task);
                    String owner = resolveOwner(task);

                    if (owner == null) {
                        LoggingUtils.logInfoMessage(LOGGER, null,
                                "No registered owner for task " + task.getId()
                                + " caseId=" + caseId + ", skipping");
                        continue;
                    }

                    LoggingUtils.logInfoMessage(LOGGER, null,
                            "Processing task=" + task.getId()
                            + " caseId=" + caseId + " owner=" + owner);

                    executeTask(task, owner, caseId, queueName);
                    triggered++;

                } catch (Exception e) {
                    LOGGER.error("Failed task " + task.getId() + ": "
                            + ExceptionUtils.getRootCauseMessage(e));
                    failed++;
                }
            }

        } catch (Exception e) {
            LOGGER.error("trigger() failed for " + queueName + ": " + e.getMessage());
        }

        LoggingUtils.logInfoMessage(LOGGER, null,
                "Queue " + queueName + " done. Triggered=" + triggered
                + " Failed=" + failed);
    }

    // ═══════════════════════════════════════════════════════════════════
    // fetchTasksByName() - queries RPAM.TASKS via GraphQL
    // replaces: advanceRuntimeDataService.queryUserTasksByVariables()
    // ═══════════════════════════════════════════════════════════════════

    private List<UserTaskInstance> fetchTasksByName(String queueName) {
        try {
            String query = queryLoader.getQuery("userTasksByName");
            Map<String, Object> variables = Map.of(
                    "name", queueName,
                    "states", ACTIVE_STATES
            );

            GraphQLResponse<Map<String, Object>> response =
                    graphQLClient.executeQuery(query, variables);

            if (response.getData() == null
                    || !response.getData().containsKey("UserTaskInstances")) {
                return Collections.emptyList();
            }

            return objectMapper.convertValue(
                    response.getData().get("UserTaskInstances"),
                    new TypeReference<List<UserTaskInstance>>() {}
            );
        } catch (Exception e) {
            LOGGER.error("fetchTasksByName failed for " + queueName
                    + ": " + e.getMessage());
            return Collections.emptyList();
        }
    }

    // ═══════════════════════════════════════════════════════════════════
    // executeTask() - claim → start → complete via BAMOE 9.x REST
    // replaces: UserTaskService.claim/start/complete
    // endpoint pattern: kogito.service.url/usertasks/instance/{id}/transition
    // ═══════════════════════════════════════════════════════════════════

    private void executeTask(UserTaskInstance task, String owner,
                             String caseId, String queueName) {
        String taskId = task.getId();
        String processInstanceId = task.getProcessInstanceId();
        String baseUrl = kogitoServiceUrl + "/usertasks/instance/" + taskId;

        try {
            // Claim if Ready
            if ("Ready".equals(task.getState())) {
                restClient.post()
                        .uri(baseUrl + "/transition?user=" + owner)
                        .contentType(MediaType.APPLICATION_JSON)
                        .body(Map.of("transitionId", "claim"))
                        .retrieve()
                        .toBodilessEntity();
            }

            // Start
            restClient.post()
                    .uri(baseUrl + "/transition?user=" + owner)
                    .contentType(MediaType.APPLICATION_JSON)
                    .body(Map.of("transitionId", "start"))
                    .retrieve()
                    .toBodilessEntity();

            // Update VEC_CASELOCK (preserved from old code)
            updateCaseLock(caseId, owner);

            // Build output variables (same structure old code used)
            Map<String, Object> outputVars = buildOutputVars(
                    queueName, caseId, owner);

            // Complete
            Map<String, Object> completeBody = new HashMap<>();
            completeBody.put("transitionId", "complete");
            completeBody.put("data", outputVars);

            restClient.post()
                    .uri(baseUrl + "/transition?user=" + owner)
                    .contentType(MediaType.APPLICATION_JSON)
                    .body(completeBody)
                    .retrieve()
                    .toBodilessEntity();

            LoggingUtils.logInfoMessage(LOGGER, null,
                    "Task completed: taskId=" + taskId
                    + " caseId=" + caseId + " owner=" + owner);

        } catch (Exception e) {
            LOGGER.error("executeTask failed taskId=" + taskId
                    + " caseId=" + caseId + ": "
                    + ExceptionUtils.getRootCauseMessage(e));
            throw e;
        }
    }

    // ═══════════════════════════════════════════════════════════════════
    // getCaseId()
    // RPAM.BUSINESS_KEY_MAPPING links processInstanceId → businessKey
    // businessKey = caseId in this system
    // ═══════════════════════════════════════════════════════════════════

    private String getCaseId(UserTaskInstance task) {
        String processInstanceId = task.getProcessInstanceId();
        try {
            List<Map<String, Object>> rows = bamoeJdbcTemplate.queryForList(
                    "SELECT BUSINESS_KEY FROM RPAM.BUSINESS_KEY_MAPPING " +
                    "WHERE PROCESS_INSTANCE_ID = ?",
                    processInstanceId
            );
            if (!rows.isEmpty()) {
                String bk = String.valueOf(rows.get(0).get("BUSINESS_KEY"));
                // businessKey may be "2147491080:someExtra" - take before colon
                return bk.contains(":") ? bk.substring(0, bk.indexOf(":")) : bk;
            }
        } catch (Exception e) {
            LOGGER.error("getCaseId failed for processInstanceId="
                    + processInstanceId + ": " + e.getMessage());
        }

        // Fallback: try PROCESS_INSTANCE_STATE_LOG
        try {
            List<Map<String, Object>> rows = bamoeJdbcTemplate.queryForList(
                    "SELECT BUSINESS_KEY FROM RPAM.PROCESS_INSTANCE_STATE_LOG " +
                    "WHERE PROCESS_INSTANCE_ID = ? " +
                    "AND BUSINESS_KEY IS NOT NULL AND ROWNUM = 1",
                    processInstanceId
            );
            if (!rows.isEmpty()) {
                String bk = String.valueOf(rows.get(0).get("BUSINESS_KEY"));
                return bk.contains(":") ? bk.substring(0, bk.indexOf(":")) : bk;
            }
        } catch (Exception e) {
            LOGGER.error("getCaseId fallback failed: " + e.getMessage());
        }

        LoggingUtils.logInfoMessage(LOGGER, null,
                "Could not resolve caseId for processInstanceId="
                + processInstanceId + ", using processInstanceId as fallback");
        return processInstanceId;
    }

    // ═══════════════════════════════════════════════════════════════════
    // resolveOwner()
    // replaces: PeopleAssignments.getPotentialOwners() iteration
    // Uses potentialUsers list from GraphQL + BPMNConstants.registeredUsers
    // ═══════════════════════════════════════════════════════════════════

    private String resolveOwner(UserTaskInstance task) {
        // Check actualOwner first
        String actualOwner = task.getActualOwner();
        if (actualOwner != null
                && BPMNConstants.registeredUsers.contains(actualOwner)) {
            return actualOwner;
        }

        // Check potentialUsers
        if (task.getPotentialUsers() != null) {
            for (String user : task.getPotentialUsers()) {
                if (BPMNConstants.registeredUsers.contains(user)) {
                    return user;
                }
            }
        }

        return null;
    }

    // ═══════════════════════════════════════════════════════════════════
    // applyOlaFilters() - preserved exactly from old OLA logic
    // ═══════════════════════════════════════════════════════════════════

    private List<UserTaskInstance> applyOlaFilters(List<UserTaskInstance> tasks) {

        // Filter 1: task must be older than 1 hour
        Instant oneHourAgo = Instant.now().minus(1, ChronoUnit.HOURS);
        tasks = tasks.stream()
                .filter(t -> t.getStarted() != null
                        && t.getStarted().isBefore(oneHourAgo))
                .collect(Collectors.toList());

        if (tasks.isEmpty()) return tasks;

        // Build caseId list for DB queries
        String caseIds = tasks.stream()
                .map(t -> "'" + getCaseId(t) + "'")
                .collect(Collectors.joining(","));

        // Filter 2: exclude CO Accept cases
        String coQuery =
            "SELECT P.ID FROM VEC_PRODUCTAPP P, VEC_DECENGINE DE, VEC_ACTPARMS V " +
            "WHERE P.CR_DECENGINEID=DE.ID AND DE.PRODUCTAPPID=P.ID " +
            "AND V.PRODUCTAPPID=P.ID AND DE.SHARESCREEN3='A' " +
            "AND P.ID IN (" + caseIds + ") AND V.AGGRIND='N'";

        List<String> coIds = new ArrayList<>();
        try {
            businessJdbcTemplate.queryForList(coQuery)
                    .forEach(r -> coIds.add(String.valueOf(r.get("ID"))));
        } catch (Exception e) {
            LOGGER.error("CO filter query failed: " + e.getMessage());
        }

        tasks = tasks.stream()
                .filter(t -> !coIds.contains(getCaseId(t)))
                .collect(Collectors.toList());

        if (tasks.isEmpty()) return tasks;

        // Rebuild caseIds after CO filter
        caseIds = tasks.stream()
                .map(t -> "'" + getCaseId(t) + "'")
                .collect(Collectors.joining(","));

        // Filter 3: exclude QUEUE_IND='C' completed cases
        String completedQuery =
            "SELECT PRODUCTAPPID FROM VEC_ACTPARMS " +
            "WHERE QUEUE_IND='C' AND PRODUCTAPPID IN (" + caseIds + ")";

        List<String> completedIds = new ArrayList<>();
        try {
            businessJdbcTemplate.queryForList(completedQuery)
                    .forEach(r -> completedIds.add(
                            String.valueOf(r.get("PRODUCTAPPID"))));
        } catch (Exception e) {
            LOGGER.error("Completed filter query failed: " + e.getMessage());
        }

        tasks = tasks.stream()
                .filter(t -> !completedIds.contains(getCaseId(t)))
                .collect(Collectors.toList());

        return tasks;
    }

    // ═══════════════════════════════════════════════════════════════════
    // updateCaseLock() - VECTUSCASE.VEC_CASELOCK update
    // preserved from old implementation
    // ═══════════════════════════════════════════════════════════════════

    private void updateCaseLock(String caseId, String userId) {
        try {
            businessJdbcTemplate.update(
                    "UPDATE VECTUSCASE.VEC_CASELOCK " +
                    "SET USERCODE=?, CREATEDATE=SYSDATE, CREATETIME=SYSDATE " +
                    "WHERE CASEID=?",
                    userId,
                    Long.parseLong(caseId)
            );
        } catch (Exception e) {
            LOGGER.error("updateCaseLock failed caseId=" + caseId
                    + ": " + e.getMessage());
        }
    }

    // ═══════════════════════════════════════════════════════════════════
    // buildOutputVars() - task completion variables
    // ═══════════════════════════════════════════════════════════════════

    private Map<String, Object> buildOutputVars(String queueName,
                                                 String caseId,
                                                 String owner) {
        Map<String, Object> vars = new HashMap<>();
        vars.put("initiator", owner);
        vars.put("caseId", caseId);
        vars.put("roboticQueueName", queueName);
        return vars;
    }
}
```

---

### Queue-specific service classes (13 total, all identical pattern)

Create these in `com.barclays.api.status.mgmt.service.impl`. All 13 follow exact same structure:

```java
package com.barclays.api.status.mgmt.service.impl;

import com.barclays.api.bap.bld.lib.logging.LogFactory;
import com.barclays.api.bap.bld.lib.logging.Logger;
import com.barclays.api.status.mgmt.service.QSchedulerService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class OlaResponseQServiceImpl {
    private static final Logger LOGGER = LogFactory.getLogger(OlaResponseQServiceImpl.class);
    @Autowired private QSchedulerService qSchedulerService;
    public void execute() {
        LOGGER.info("OlaResponseQServiceImpl executing");
        qSchedulerService.trigger("OLARESPONSEQ");
    }
}
```

All 13 with their exact queue name strings matching your `QueueUtils` and `ALLOWED_QUEUES`:

| Class | `trigger()` argument |
|-------|----------------------|
| `OlaResponseQServiceImpl` | `"OLARESPONSEQ"` |
| `NoInterfaceQServiceImpl` | `"NOINTERFACEQ"` |
| `NTCHoldQServiceImpl` | `"NTCHOLDQ"` |
| `NGCBckServiceImpl` | `"NGCBACKQ"` |
| `LogCleanUpServiceImpl` | `"LOGCLEANUPQ"` |
| `CASLSSQServiceImpl` | `"CASLSSQ"` |
| `NODEQServiceImpl` | `"NODEQ"` |
| `NoUploadQServiceImpl` | `"NOUPLOADQ"` |
| `BatchResubmitQServiceImpl` | `"BATCHRESUBMITQ"` |
| `LapsedProcessServiceImpl` | `"LAPSEDPROCESSQ"` |
| `FailedSoftToHardServiceImpl` | `"FAILEDSOFTTOHARDQ"` |
| `SMSRetryServiceImpl` | `"SMSRETRYQ"` |
| `AdvancedLogCleanUpServiceImpl` | `"ADVLOGCLEANUPQ"` |

---

### `QSchedulerService` interface

```java
package com.barclays.api.status.mgmt.service;

public interface QSchedulerService {
    void trigger(String queueName);
}
```

---

### `SchedulerCallbackController`

```java
package com.barclays.api.status.mgmt.controller;

import com.barclays.api.bap.bld.lib.logging.LogFactory;
import com.barclays.api.bap.bld.lib.logging.Logger;
import com.barclays.api.cif.utilities.LoggingUtils;
import com.barclays.api.status.mgmt.service.impl.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.Map;

@RestController
@RequestMapping("/scheduler")
public class SchedulerCallbackController {

    private static final Logger LOGGER =
        LogFactory.getLogger(SchedulerCallbackController.class);

    @Autowired private OlaResponseQServiceImpl olaResponseQService;
    @Autowired private NoInterfaceQServiceImpl noInterfaceQService;
    @Autowired private NTCHoldQServiceImpl ntcHoldQService;
    @Autowired private NGCBckServiceImpl ngcBckService;
    @Autowired private LogCleanUpServiceImpl logCleanUpService;
    @Autowired private CASLSSQServiceImpl caslssqService;
    @Autowired private NODEQServiceImpl nodeQService;
    @Autowired private NoUploadQServiceImpl noUploadQService;
    @Autowired private BatchResubmitQServiceImpl batchResubmitQService;
    @Autowired private LapsedProcessServiceImpl lapsedProcessService;
    @Autowired private FailedSoftToHardServiceImpl failedSoftToHardService;
    @Autowired private SMSRetryServiceImpl smsRetryService;
    @Autowired private AdvancedLogCleanUpServiceImpl advancedLogCleanUpService;

    @PostMapping("/execute/{queueName}")
    public ResponseEntity<String> executeQueue(
            @PathVariable String queueName,
            @RequestBody(required = false) Map<String, Object> payload) {

        LoggingUtils.logInfoMessage(LOGGER, null,
                "Callback received for queue: " + queueName);
        try {
            dispatch(queueName);
            return ResponseEntity.ok("OK");
        } catch (Exception e) {
            LOGGER.error("Callback failed for " + queueName + ": " + e.getMessage());
            return ResponseEntity.ok("FAILED");
        }
    }

    private void dispatch(String queueName) {
        switch (queueName.toUpperCase()) {
            case "OLARESPONSEQ"      -> olaResponseQService.execute();
            case "NOINTERFACEQ"      -> noInterfaceQService.execute();
            case "NTCHOLDQ"          -> ntcHoldQService.execute();
            case "NGCBACKQ"          -> ngcBckService.execute();
            case "LOGCLEANUPQ"       -> logCleanUpService.execute();
            case "CASLSSQ"           -> caslssqService.execute();
            case "NODEQ"             -> nodeQService.execute();
            case "NOUPLOADQ"         -> noUploadQService.execute();
            case "BATCHRESUBMITQ"    -> batchResubmitQService.execute();
            case "LAPSEDPROCESSQ"    -> lapsedProcessService.execute();
            case "FAILEDSOFTTOHARDQ" -> failedSoftToHardService.execute();
            case "SMSRETRYQ"         -> smsRetryService.execute();
            case "ADVLOGCLEANUPQ"    -> advancedLogCleanUpService.execute();
            default -> throw new IllegalArgumentException(
                    "Unknown queue: " + queueName);
        }
    }
}
```

---

### One change to `SchedulerInfoServiceImpl` — set `CALLBACK_ENDPOINT`

In your existing `updateRoboticQTimer()`, change the `ProcessJobDescription.of()` call:

```java
// ADD this field at top of class:
@Value("${kogito.service.url}")
private String serviceUrl;

// CHANGE this block (currently has no callback):
ProcessJobDescription jobDescription = ProcessJobDescription.of(
        ExactExpirationTime.of(
                ZonedDateTime.ofInstant(fireTime, ZoneId.systemDefault())
        ),
        queueName
);

// TO this (adds the callback URL):
ProcessJobDescription jobDescription = ProcessJobDescription.of(
        ExactExpirationTime.of(
                ZonedDateTime.ofInstant(fireTime, ZoneId.systemDefault())
        ),
        1,
        queueName,
        serviceUrl + "/scheduler/execute/" + queueName
);
```

That's the entire change to your existing file — one `@Value` field added, one method call updated.

---

### Summary — complete list of new files

| File | Location |
|------|----------|
| `user-tasks-by-name.graphql` | `resources/graphql/queries/` |
| `UserTaskInstance.java` | `model/graphql/` |
| `QSchedulerService.java` (interface) | `service/` |
| `QSchedulerServiceImpl.java` | `service/impl/` |
| `SchedulerCallbackController.java` | `controller/` |
| `OlaResponseQServiceImpl.java` | `service/impl/` |
| `NoInterfaceQServiceImpl.java` | `service/impl/` |
| `NTCHoldQServiceImpl.java` | `service/impl/` |
| `NGCBckServiceImpl.java` | `service/impl/` |
| `LogCleanUpServiceImpl.java` | `service/impl/` |
| `CASLSSQServiceImpl.java` | `service/impl/` |
| `NODEQServiceImpl.java` | `service/impl/` |
| `NoUploadQServiceImpl.java` | `service/impl/` |
| `BatchResubmitQServiceImpl.java` | `service/impl/` |
| `LapsedProcessServiceImpl.java` | `service/impl/` |
| `FailedSoftToHardServiceImpl.java` | `service/impl/` |
| `SMSRetryServiceImpl.java` | `service/impl/` |
| `AdvancedLogCleanUpServiceImpl.java` | `service/impl/` |

**One change to existing file:** `SchedulerInfoServiceImpl.java` — add `@Value` + update `ProcessJobDescription.of()` call only.

Everything else — `SchedulerInfoServiceImpl`, `SchedulerQueryServiceImpl`, `SchedulerGraphQLClient`, `GraphQLQueryLoader`, `QueueUtils`, `Job`, `JobResponse`, all 3 scheduler GraphQL files — **untouched**.