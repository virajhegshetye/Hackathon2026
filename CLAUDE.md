# BAMOE Migration Workspace

## Folder Structure & Roles

- `rhpam-jdk8-old/` — **READ-ONLY REFERENCE.** Old RHPAM JDK8 codebase (jBPM-based).
  Never edit, commit, or run any git command inside this folder. Only read
  from it to understand old logic, old entity/table usage, old service
  implementations, and old configuration.

- `bamoe-9.4-jdk21/` — **ACTIVE TARGET.** The real project to edit.
  All code changes, migrations, and commits happen here only.
  Controllers are already migrated. Service/implementation classes,
  repositories, and entities are NOT yet migrated and need rework since
  tables and class structures changed between RHPAM and BAMOE.

- `BAMOE_RPAM.sql` — New BAMOE schema (target DB structure). Source of truth
  for table/column names in the new system.

- `ibamoe-9.4.x-documentation.pdf` — Official BAMOE 9.4 migration/reference
  documentation (local copy).

- `job_details.csv`, `job_execution_log.csv` — Sample data from **new BAMOE
  tables** (post-migration schema). Use to understand actual column
  names/values in the new system.

- `requestinfoData.csv`, `task.csv` — Sample data from **old RHPAM tables**
  (jBPM `RequestInfo`, `Task`). Use to understand old table shape before
  migration.

## Official Documentation Reference

When local documentation is insufficient or you need to confirm current
BAMOE 9.4.x API/behavior, check the official IBM docs:

- Base: https://www.ibm.com/docs/en/ibamoe/9.4.x
- Always prefer checking this site (via web fetch/search) over assuming
  behavior from RHPAM/jBPM knowledge, since BAMOE 9.4 has renamed/replaced
  many jBPM concepts (e.g. ExecutorService → Kogito Jobs Service).
- Cross-reference: local PDF (`ibamoe-9.4.x-documentation.pdf`) first for
  speed, then the live IBM docs site for anything version-specific,
  recently changed, or not covered in the PDF.
- If the two sources conflict, flag the conflict explicitly rather than
  picking one silently.

## Configuration Reference

- `rhpam-jdk8-old/.../application-dev.properties` (or equivalent env-specific
  properties file) — contains old scheduler-related configuration such as
  cron expressions, thread pool sizes, executor retry/interval settings,
  job service URLs, and any RHPAM ExecutorService-specific properties.

  Before migrating any scheduler-related endpoint (getRoboticQList,
  updateRoboticQTimer, or their service impls), read this file fully and:
  1. List every scheduler-related property found (name, value, purpose).
  2. Check whether an equivalent property already exists in
     `bamoe-9.4-jdk21/.../application*.properties` (BAMOE/Kogito naming
     will differ — e.g. old `executor.*` properties likely map to
     `kogito.jobs-service.*` or similar).
  3. Flag any old property that has NO equivalent yet in the new project —
     this may need to be added, and its correct BAMOE-side name/format
     should be confirmed via local PDF or
     https://www.ibm.com/docs/en/ibamoe/9.4.x before adding it.
  4. Never silently drop a property — if unsure whether it's still needed
     under Kogito Jobs Service, ask rather than omit it.

## Table Mapping (old → new)

| Old (RHPAM/jBPM) | Old data ref         | New (BAMOE)                          | New data ref            |
|-------------------|----------------------|----------------------------------------|----------------------------|
| RequestInfo table  | requestinfoData.csv  | (map via BAMOE_RPAM.sql)              | job_details.csv            |
| Task table         | task.csv             | (map via BAMOE_RPAM.sql)              | job_execution_log.csv      |

> Exact old→new table/column mapping must be confirmed against
> `BAMOE_RPAM.sql` and documentation (local PDF + IBM docs site) before
> writing code — do not assume names match old RHPAM tables.

## Current Migration Scope (active task)

Only migrate these 2 endpoints and their full dependency chain. Do not
touch, refactor, or "helpfully improve" anything outside this scope, even
if other migration gaps are noticed while working.

1. **GET** `/retrieveAppDetails/queueList/getRoboticQList`
   - Controller method: `HomeScreenController.getRoboticQList()`
   - Old file: `rhpam-jdk8-old/.../HomeScreenController.java`
   - Calls: `schedulerInfoService.getRoboticQList()`
   - Service: `SchedulerInfoService.java` (interface), `SchedulerInfoServiceImpl.java`

2. **POST** `/applicationDetails/execute/updateRoboticQTimer`
   - Controller method: `HomeScreenController.timerUpdateForNoInterfaceQ(WeblinkRequest)`
   - Old file: `rhpam-jdk8-old/.../HomeScreenController.java`
   - Calls: `schedulerInfoService.updateDate(requestData)`
   - Request DTO: `WeblinkRequest.java` (uses `getData().getAttributes().getApplicationDetails()`)
   - Service: `SchedulerInfoService.java` (interface), `SchedulerInfoServiceImpl.java`

Candidate shared dependencies to verify via Completeness Check below:
- `SchedulerConstants.java` — confirm if referenced by these 2 service methods
- `Response` / `ResponseAssembler` / `ResponseHttpStatus` — shared helper
  used across many other endpoints too. Reference only, never modify.

## Completeness Check (mandatory before migration starts)

Before writing any code, verify the dependency chain above is complete:

1. For every class listed in scope, search `rhpam-jdk8-old/` for all
   imports, `@Autowired`/injected fields, and method calls to other classes.
   Do this via actual grep/search — do not just trust the list already
   written above; re-derive it independently.
2. Recursively follow the chain: service → repo → entity → DTO → mapper →
   validator → constants/enums → exception classes → shared utils.
3. Cross-check: does anything in this chain touch a table NOT present in
   `requestinfoData.csv` / `task.csv`? Flag it — may indicate a missed
   old table.
4. Check reverse dependencies too: does anything else in `rhpam-jdk8-old/`
   call INTO these classes (not just what these classes call out to)?
   Flag any class shared with other endpoints outside this scope —
   migrating/changing a shared class can silently break something else.
5. Read `application-dev.properties` in `rhpam-jdk8-old/` and list every
   scheduler-related property. Check if equivalents exist in
   `bamoe-9.4-jdk21/`'s properties files; flag anything missing or needing
   renaming per Kogito Jobs Service conventions.
6. For any BAMOE-side behavior uncertainty (e.g. how Kogito Jobs Service
   replaces the old scheduler pattern), check local PDF, then IBM docs
   site (https://www.ibm.com/docs/en/ibamoe/9.4.x) before proceeding.
7. Report back:
   - Full final class list (including anything not in the original scope list)
   - Anything newly found, clearly marked **"MISSED — not in original scope"**
   - Any class shared with other endpoints, clearly flagged
   - Scheduler config properties found, and which are missing on the BAMOE side
   - Any doc conflicts found between local PDF and IBM docs site
8. **Wait for explicit confirmation before writing or migrating any code.**

## Rules

1. Never run `git add`, `git commit`, or `git push` unless explicitly asked,
   and only ever inside `bamoe-9.4-jdk21/`.
2. Never modify anything inside `rhpam-jdk8-old/`.
3. Work one class at a time. Show the diff/plan before moving to the next.
4. When migrating:
   - Read old Controller + Service/Impl in `rhpam-jdk8-old/`.
   - Read old scheduler config in `application-dev.properties` (see
     Configuration Reference section) if the endpoint is scheduler-related.
   - Cross-check old table usage against `requestinfoData.csv` / `task.csv`.
   - Cross-check new schema in `BAMOE_RPAM.sql` and new sample data in
     `job_details.csv` / `job_execution_log.csv`.
   - Confirm BAMOE-specific API/class/property changes against local PDF
     and IBM docs site (Kogito Jobs Service vs jBPM ExecutorService, etc).
   - Only then write/update the implementation in `bamoe-9.4-jdk21/`.
5. Controllers in `bamoe-9.4-jdk21/` are already migrated — only touch them
   if the endpoint signature itself must change to match new service logic.
6. If any table/column mapping, method behavior, property mapping, or
   migration path is ambiguous, stop and ask rather than guessing.