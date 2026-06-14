If your goal is only to prove Quartz works in your BAMOE 9.3 environment, don't start with the full migration code that Claude generated.

Create a minimal POC exactly like you did for JobsService.

The POC should answer only these questions:

1. Can Quartz persist jobs in QRTZ_* tables?


2. Can Quartz fire a cron trigger?


3. Can Quartz invoke a Spring bean?


4. Can Quartz call QSchedulerService.trigger()?




---

POC Class 1 - Quartz Job

package com.barclays.poc.quartz;

import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.springframework.stereotype.Component;

@Component
public class QuartzPocJob implements Job {

    @Override
    public void execute(JobExecutionContext context)
            throws JobExecutionException {

        System.out.println("=== QUARTZ POC JOB EXECUTED ===");
        System.out.println("Time : " + new java.util.Date());

    }
}


---

POC Class 2 - Scheduler Service

package com.barclays.poc.quartz;

import java.util.Date;

import org.quartz.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class QuartzPocService {

    @Autowired
    private Scheduler scheduler;

    public void scheduleJob() throws Exception {

        JobDetail job =
                JobBuilder.newJob(QuartzPocJob.class)
                        .withIdentity("POC_JOB", "POC")
                        .build();

        Trigger trigger =
                TriggerBuilder.newTrigger()
                        .withIdentity("POC_TRIGGER", "POC")
                        .startAt(new Date(System.currentTimeMillis() + 60000))
                        .build();

        scheduler.scheduleJob(job, trigger);
    }
}


---

POC Class 3 - Controller

package com.barclays.poc.quartz;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/quartz")
public class QuartzPocController {

    @Autowired
    private QuartzPocService quartzPocService;

    @PostMapping("/schedule")
    public String schedule() throws Exception {

        quartzPocService.scheduleJob();

        return "Quartz Job Scheduled";
    }
}


---

Dependency

implementation 'org.springframework.boot:spring-boot-starter-quartz'


---

Properties

For initial validation:

spring.quartz.job-store-type=jdbc
spring.quartz.jdbc.initialize-schema=never

Add full Oracle cluster settings later.


---

Test

Call:

POST /quartz/schedule

Expected:

Response:

Quartz Job Scheduled


---

Verify Database

After scheduling:

SELECT *
FROM QRTZ_JOB_DETAILS;

Should contain:

POC_JOB


---

SELECT *
FROM QRTZ_TRIGGERS;

Should contain:

POC_TRIGGER


---

After one minute:

Application log:

=== QUARTZ POC JOB EXECUTED ===


---

Phase 2

Once the above works, replace the log statement with:

@Autowired
private QSchedulerService qSchedulerService;

and

qSchedulerService.trigger(
        SchedulerConstants.OLARESPONSEQ);

Then verify:

1. Quartz fires.


2. QSchedulerServiceImpl.trigger() executes.


3. Existing queue processing starts.


4. QRTZ_* tables contain scheduler metadata.



That gives you a clean Quartz POC equivalent to the JobsService POC you already completed, without implementing the full migration architecture first.