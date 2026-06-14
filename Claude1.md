This is the critical clarification. Now I understand the full picture:

**From the screenshots:**

`VECTUSCASE.VEC_CASELOCK` columns:
- `CASEID` — NUMBER(10) NOT NULL
- `USERID` — NUMBER(10)
- `CREATEDATE`, `CREATETIME`, `CEXPIREDATE`, `EXPIRETIME` — DATE
- `USERCODE` — VARCHAR2(50) ← this is the user code string

`RPAM.JBPM_USER_TASKS` columns:
- `ID`, `USER_TASK_ID`, `TASK_NAME`, `STATUS`, `ACTUAL_OWNER`
- `PROCESS_INSTANCE_ID`, `PROCESS_ID`, `ROOT_PROCESS_INSTANCE_ID`
- **No `NAME` or `STATE` column** — they are `TASK_NAME` and `STATUS`

**And your critical point:** Queue names like `OLARESPONSEQ` are **not stored in `JBPM_USER_TASKS`** — they are stored in the old `TASK` table which is jBPM 8. In BAMOE 9.x, GraphQL `UserTaskInstance.name` comes from the Data Index (`kogito-addons-springboot-data-index-jpa`) which stores it differently.

---

**Run these to find where queue names actually live in 9.x:**

```sql
-- Check what TASK_NAME values look like in JBPM_USER_TASKS
SELECT TASK_NAME, STATUS, ACTUAL_OWNER, 
       PROCESS_INSTANCE_ID
FROM RPAM.JBPM_USER_TASKS
WHERE ROWNUM <= 10;

-- Check Data Index user task table
SELECT TABLE_NAME FROM ALL_TABLES 
WHERE OWNER = 'RPAM'
AND TABLE_NAME LIKE '%USER_TASK%'
AND TABLE_NAME NOT LIKE 'JBPM%'
AND TABLE_NAME NOT LIKE 'HTE%'
AND TABLE_NAME NOT LIKE 'TASK_%'
ORDER BY TABLE_NAME;

-- Check if there's a separate data index table
SELECT TABLE_NAME FROM ALL_TABLES
WHERE OWNER = 'RPAM'
AND TABLE_NAME IN (
    'USER_TASK_INSTANCES',
    'USERTASKINSTANCES', 
    'DATA_INDEX_USER_TASKS',
    'KIE_USER_TASK'
);

-- Check PROCESS_INSTANCES for businessKey (caseId)
DESC RPAM.PROCESS_INSTANCES;
```

Also run:
```sql
-- See what's in TASKS table (may be Data Index)
DESC RPAM.TASKS;

-- Sample data from TASKS
SELECT * FROM RPAM.TASKS 
WHERE ROWNUM <= 5;
```

Share results. This will tell us:
1. Where `name`/`state` fields that GraphQL returns actually come from in the DB
2. How `caseId` maps to `PROCESS_INSTANCE_ID` via `PROCESS_INSTANCES.businessKey`

The GraphQL query for `UserTaskInstances` works at the API level — but understanding the underlying storage confirms what `referenceName` and `name` actually contain so `getCaseId()` is correct.