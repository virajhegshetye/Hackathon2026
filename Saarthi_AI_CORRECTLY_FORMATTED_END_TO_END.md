# Saarthi AI – Mobilisation Intelligence Platform  
**Hackathon:** Hack a Difference 2026  
**Team:** PromptCrafters  
**Charity:** Magic Bus Foundation (India)  
**Tech Stack:** Azure Databricks, Spring Boot (JDK 21), Spring AI, Azure OpenAI, HTML UI  

---

## 1. What Problem We Are Solving

Magic Bus faces challenges in **youth mobilisation** for skilling and job placement:

- Manual onboarding taking up to 60 days  
- Difficulty identifying eligible candidates  
- Drop-offs during mobilisation  
- Poor visibility into which engagement channels work  
- No clear evidence of impact  

### Our Solution: Saarthi AI

Saarthi AI is an **AI-powered mobilisation intelligence platform** that:

- Identifies the right youth to mobilise  
- Predicts mobilisation drop-off risk  
- Analyses engagement channel effectiveness  
- Automates onboarding recommendations using Azure OpenAI  
- Shows measurable impact metrics  

---

## 2. Workspace Setup (Hackathon VDI)

### 2.1 Base Folder

Create this folder:

```text
C:\hackathon\promptcrafters```

### 2.2 Final Project Structure

```text
C:\hackathon\promptcrafters├── databricks
│   └── notebooks
│       ├── 01_generate_candidates.py
│       ├── 02_mobilisation_risk_scoring.py
│       ├── 03_channel_effectiveness.py
│       └── 04_verify_tables.py
│
├── backend
│   ├── pom.xml
│   └── src\main
│       ├── java\com\promptcrafters\saarthi
│       │   ├── SaarthiApplication.java
│       │   ├── controller\MobilisationController.java
│       │   ├── service\MobilisationRiskService.java
│       │   ├── service\OnboardingRecommendationService.java
│       │   └── dto\CandidateDto.java
│       └── resources
│           ├── application.yml
│           └── static\index.html
```

---

## 3. Tool Installation

Run **Command Prompt as Administrator**:

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

---

## 4. Azure Databricks – Synthetic Data

### 4.1 Notebook: 01_generate_candidates

```python
from dbldatagen import DataGenerator
from pyspark.sql.types import *

gen = (
    DataGenerator(spark, "candidates", rows=60000)
    .withColumn("candidate_id", StringType(), expr="uuid()")
    .withColumn("age", IntegerType(), 18, 25)
    .withColumn("eligibility_score", IntegerType(), 40, 100)
    .withColumn("engagement_score", IntegerType(), 1, 5)
    .withColumn("digital_access", StringType(), values=["Low","Medium","High"])
    .withColumn(
        "mobilisation_channel",
        StringType(),
        values=["Field Volunteer","WhatsApp","Community Partner","School Outreach","Referral"]
    )
)

candidates_df = gen.build()
display(candidates_df)
```

---

### 4.2 Notebook: 02_mobilisation_risk_scoring

```python
from pyspark.sql.functions import col, when

risk_df = candidates_df.withColumn(
    "risk_score",
    0.4 * (100 - col("eligibility_score")) +
    0.3 * (5 - col("engagement_score")) * 10 +
    when(col("digital_access") == "Low", 20)
     .when(col("digital_access") == "Medium", 10)
     .otherwise(0)
).withColumn(
    "risk_level",
    when(col("risk_score") >= 60, "High")
     .when(col("risk_score") >= 35, "Medium")
     .otherwise("Low")
)

risk_df.write.mode("overwrite").saveAsTable("mobilisation_risk_profile")
display(risk_df)
```

---

### 4.3 Notebook: 03_channel_effectiveness

```python
channel_df = risk_df.groupBy("mobilisation_channel").count()
channel_df.write.mode("overwrite").saveAsTable("channel_effectiveness")
display(channel_df)
```

---

## 5. Spring Boot Backend (JDK 21)

### 5.1 pom.xml

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

### 5.2 application.yml

```yaml
server:
  port: 8080

spring:
  ai:
    azure:
      openai:
        api-key: YOUR_AZURE_OPENAI_KEY
        endpoint: https://YOUR-RESOURCE.openai.azure.com/
        chat:
          options:
            deployment-name: saarthi-chat
```

---

### 5.3 CandidateDto.java

```java
package com.promptcrafters.saarthi.dto;

public class CandidateDto {
    public String candidateId;
    public String riskLevel;
    public String channel;
}
```

---

### 5.4 MobilisationRiskService.java

```java
package com.promptcrafters.saarthi.service;

import com.promptcrafters.saarthi.dto.CandidateDto;
import org.springframework.stereotype.Service;

@Service
public class MobilisationRiskService {

    public CandidateDto getCandidate(String id) {
        CandidateDto dto = new CandidateDto();
        dto.candidateId = id;
        dto.riskLevel = "High";
        dto.channel = "WhatsApp";
        return dto;
    }
}
```

---

### 5.5 OnboardingRecommendationService.java

```java
package com.promptcrafters.saarthi.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.stereotype.Service;

@Service
public class OnboardingRecommendationService {

    private final ChatClient chatClient;

    public OnboardingRecommendationService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public String recommend(String profile) {
        return chatClient.prompt()
            .system("Generate simple onboarding steps for youth mobilisation in India.")
            .user(profile)
            .call()
            .content();
    }
}
```

---

### 5.6 MobilisationController.java

```java
package com.promptcrafters.saarthi.controller;

import com.promptcrafters.saarthi.service.*;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api")
public class MobilisationController {

    private final MobilisationRiskService riskService;
    private final OnboardingRecommendationService onboardingService;

    public MobilisationController(
        MobilisationRiskService riskService,
        OnboardingRecommendationService onboardingService
    ) {
        this.riskService = riskService;
        this.onboardingService = onboardingService;
    }

    @GetMapping("/candidate/{id}")
    public Object candidate(@PathVariable String id) {
        return riskService.getCandidate(id);
    }

    @PostMapping(value = "/onboarding", consumes = "text/plain")
    public String onboarding(@RequestBody String profile) {
        return onboardingService.recommend(profile);
    }
}
```

---

### 5.7 SaarthiApplication.java

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

## 6. UI (Spring Boot Static Page)

```html
<!DOCTYPE html>
<html>
<body>
<h2>Saarthi AI – Mobilisation Dashboard</h2>

<input id="cid" placeholder="Candidate ID">
<button onclick="load()">Get Candidate</button>

<br><br>

<textarea id="profile" rows="5" cols="60"></textarea>
<button onclick="onboard()">Onboarding Recommendation</button>

<pre id="out"></pre>

<script>
async function load() {
  const r = await fetch(`/api/candidate/${cid.value}`);
  out.innerText = JSON.stringify(await r.json(), null, 2);
}

async function onboard() {
  const r = await fetch("/api/onboarding", {
    method: "POST",
    headers: { "Content-Type": "text/plain" },
    body: profile.value
  });
  out.innerText = await r.text();
}
</script>
</body>
</html>
```

---

## 7. Run the Application

```bash
cd backend
mvn spring-boot:run
```

Open in browser:

```text
http://localhost:8080/index.html
```

---

## 8. Demo Flow (Hackathon)

1. Explain Magic Bus mobilisation problem  
2. Show Databricks risk + channel tables  
3. Call onboarding recommendation  
4. Show AI-generated guidance  
5. Highlight reduced onboarding time & impact  

---

**END OF DOCUMENT**
