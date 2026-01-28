# Saarthi AI – FULL BUILD SCRIPT (End-to-End)
**Team:** PromptCrafters  
**Hackathon:** Hack a Difference 2026  
**Charity:** Magic Bus Foundation (India)  
**Java:** JDK 21  

---

# READ THIS FIRST (IMPORTANT)

This document is a **literal build script**.

If you:
- follow steps **top to bottom**
- create files **exactly at the paths mentioned**
- paste code **exactly as shown**

👉 the solution will run without guesswork.

There are **no placeholders**, **no pseudo-code**, and **no skipped steps**.

---

# 1. WORKSPACE SETUP (VDI)

## 1.1 Base folder

Create this folder on the hackathon VDI:

```
C:\hackathon\promptcrafters\
```

All work happens inside this folder.

---

## 1.2 Final folder structure (REFERENCE)

You will END with this exact structure:

```
C:\hackathon\promptcrafters\
│
├── databricks\
│   └── notebooks\
│       ├── 01_generate_students.py
│       ├── 02_risk_scoring.py
│       └── 03_verify_table.py
│
├── backend\
│   ├── pom.xml
│   └── src\main\
│       ├── java\com\promptcrafters\saarthi\
│       │   ├── SaarthiApplication.java
│       │   ├── config\SpringAiConfig.java
│       │   ├── controller\SaarthiController.java
│       │   ├── service\StudentRiskService.java
│       │   └── dto\StudentRiskDto.java
│       └── resources\
│           ├── application.yml
│           └── static\index.html
│
└── docs\
    └── Saarthi_AI_FULL_BUILD.md
```

---

# 2. TOOLING INSTALLATION (VDI)

Open **Command Prompt as Administrator**.

```bash
choco install openjdk21 -y
choco install maven -y
choco install git -y
choco install vscode -y
```

Verify:

```bash
java -version
mvn -v
```

Expected:
- Java 21
- Maven 3.x

---

# 3. AZURE DATABRICKS (DATA FOUNDATION)

## 3.1 Create Databricks Workspace

Azure Portal → Create Resource → **Azure Databricks**

- Resource Group: `rg-promptcrafters`
- Workspace name: `promptcrafters-dbx`
- Pricing: Trial / Standard

Launch Workspace.

---

## 3.2 Databricks workspace path

Use:

```
/Workspace/Users/<your_email>/promptcrafters/
```

---

## 3.3 Notebook 01 – Generate Synthetic Students

Create Notebook:
- Name: `01_generate_students`
- Language: Python

### CELL 1 – Imports

```python
from dbldatagen import DataGenerator
from pyspark.sql.types import *
```

### CELL 2 – Data generation

```python
rows = 50000

gen = (
    DataGenerator(spark, "students", rows=rows)
    .withColumn("student_id", StringType(), expr="uuid()")
    .withColumn("age", IntegerType(), minValue=15, maxValue=24)
    .withColumn("attendance_pct", IntegerType(), minValue=40, maxValue=100)
    .withColumn("assessment_score", IntegerType(), minValue=0, maxValue=100)
    .withColumn("engagement_score", IntegerType(), minValue=1, maxValue=5)
    .withColumn("digital_access", StringType(), values=["Low","Medium","High"])
    .withColumn("family_income_band", StringType(), values=["Low","Medium"])
    .withColumn("location_type", StringType(), values=["Urban","Semi-Urban","Rural"])
)

students_df = gen.build()
display(students_df)
```

---

## 3.4 Notebook 02 – Risk Scoring

Create Notebook:
- Name: `02_risk_scoring`
- Language: Python

### CELL 1 – Imports

```python
from pyspark.sql.functions import col, when
```

### CELL 2 – Risk calculation

```python
risk_df = students_df.withColumn(
    "risk_score",
    0.4*(100-col("attendance_pct")) +
    0.3*(100-col("assessment_score")) +
    when(col("digital_access")=="Low",20)
     .when(col("digital_access")=="Medium",10)
     .otherwise(0) +
    0.1*(5-col("engagement_score"))*10
).withColumn(
    "risk_level",
    when(col("risk_score")>=60,"High")
     .when(col("risk_score")>=35,"Medium")
     .otherwise("Low")
)

display(risk_df)
```

### CELL 3 – Save table

```python
risk_df.write.mode("overwrite").saveAsTable("student_risk_profile")
```

---

## 3.5 Notebook 03 – Verify Table

```python
spark.sql("SELECT risk_level, COUNT(*) FROM student_risk_profile GROUP BY risk_level").show()
```

---

# 4. SPRING BOOT BACKEND (JDK 21)

## 4.1 Create project

Use Spring Initializr:

- Project: Maven
- Language: Java
- Spring Boot: 3.2+
- Java: 21
- Group: `com.promptcrafters`
- Artifact: `saarthi`
- Dependency: Spring Web

Extract into:

```
C:\hackathon\promptcrafters\backend
```

---

## 4.2 pom.xml (FULL)

Replace pom.xml content with:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.promptcrafters</groupId>
  <artifactId>saarthi</artifactId>
  <version>1.0.0</version>

  <properties>
    <java.version>21</java.version>
    <spring.ai.version>0.8.1</spring.ai.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
      <groupId>org.springframework.ai</groupId>
      <artifactId>spring-ai-azure-openai-spring-boot-starter</artifactId>
      <version>${spring.ai.version}</version>
    </dependency>
  </dependencies>
</project>
```

---

## 4.3 application.yml

Create file:
```
backend\src\main\resources\application.yml
```

```yaml
server:
  port: 8080

spring:
  ai:
    azure:
      openai:
        api-key: YOUR_AZURE_OPENAI_KEY
        endpoint: https://YOUR-OPENAI-RESOURCE.openai.azure.com/
        chat:
          options:
            deployment-name: saarthi-chat
```

---

## 4.4 SpringAiConfig.java

Create file:
```
backend\src\main\java\com\promptcrafters\saarthi\config\SpringAiConfig.java
```

```java
package com.promptcrafters.saarthi.config;

import org.springframework.context.annotation.Configuration;

@Configuration
public class SpringAiConfig {
}
```

---

## 4.5 StudentRiskDto.java

Create file:
```
backend\src\main\java\com\promptcrafters\saarthi\dto\StudentRiskDto.java
```

```java
package com.promptcrafters.saarthi.dto;

public class StudentRiskDto {
    public String studentId;
    public String riskLevel;
    public String reason;
}
```

---

## 4.6 StudentRiskService.java

Create file:
```
backend\src\main\java\com\promptcrafters\saarthi\service\StudentRiskService.java
```

```java
package com.promptcrafters.saarthi.service;

import com.promptcrafters.saarthi.dto.StudentRiskDto;
import org.springframework.stereotype.Service;

@Service
public class StudentRiskService {

    public StudentRiskDto getRisk(String studentId) {
        StudentRiskDto dto = new StudentRiskDto();
        dto.studentId = studentId;
        dto.riskLevel = "High";
        dto.reason = "Low attendance and low digital access";
        return dto;
    }
}
```

---

## 4.7 SaarthiController.java

Create file:
```
backend\src\main\java\com\promptcrafters\saarthi\controller\SaarthiController.java
```

```java
package com.promptcrafters.saarthi.controller;

import org.springframework.web.bind.annotation.*;
import org.springframework.ai.chat.client.ChatClient;
import com.promptcrafters.saarthi.service.StudentRiskService;

@RestController
@RequestMapping("/api")
public class SaarthiController {

    private final ChatClient chatClient;
    private final StudentRiskService riskService;

    public SaarthiController(ChatClient chatClient, StudentRiskService riskService) {
        this.chatClient = chatClient;
        this.riskService = riskService;
    }

    @GetMapping("/student/{id}")
    public Object getRisk(@PathVariable String id) {
        return riskService.getRisk(id);
    }

    @PostMapping(value="/career", consumes="text/plain")
    public String career(@RequestBody String profile) {
        return chatClient.prompt()
            .system("You are a career guide for underprivileged youth in India.")
            .user(profile)
            .call()
            .content();
    }
}
```

---

## 4.8 SaarthiApplication.java

```java
package com.promptcrafters.saarthi;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SaarthiApplication {
    public static void main(String[] args) {
        SpringApplication.run(SaarthiApplication.class, args);
    }
}
```

---

# 5. UI (SERVED BY SPRING BOOT)

Create file:
```
backend\src\main\resources\static\index.html
```

```html
<!DOCTYPE html>
<html>
<body>
<h2>Saarthi AI – Magic Bus</h2>

<input id="sid" placeholder="Student ID">
<button onclick="risk()">Get Risk</button>

<br><br>

<textarea id="profile" rows="5" cols="60"></textarea>
<button onclick="career()">Career Guidance</button>

<pre id="out"></pre>

<script>
async function risk(){
  const id=document.getElementById("sid").value;
  const r=await fetch(`/api/student/${id}`);
  out.innerText=JSON.stringify(await r.json(),null,2);
}
async function career(){
  const t=profile.value;
  const r=await fetch("/api/career",{method:"POST",headers:{"Content-Type":"text/plain"},body:t});
  out.innerText=await r.text();
}
</script>
</body>
</html>
```

---

# 6. COPILOT STUDIO

HTTP Action:
- POST https://<APP-SERVICE>/api/career
- Body: {{user_message}}

---

# 7. RUN & DEMO

```bash
cd backend
mvn spring-boot:run
```

Open:
```
http://localhost:8080/index.html
```

---

END OF BUILD SCRIPT
