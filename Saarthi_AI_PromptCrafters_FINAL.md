# Saarthi AI – Full End-to-End Hackathon Guide  
**Team:** PromptCrafters  
**Hackathon:** Hack a Difference 2026  
**Charity:** Magic Bus Foundation (India)

---

## 0. How to use this document

This is a complete build manual.
Follow sections in order and create files exactly at the paths mentioned.

---

## 1. Problem & Solution

Magic Bus Foundation supports underserved youth with education and employability.

Challenges:
- Dropout risk detected late
- Generic career guidance
- Manual reporting

Saarthi AI:
- Detects dropout risk early
- Uses Azure OpenAI for career guidance
- Provides UI and Copilot Studio access

---

## 2. Workspace Layout

Base path:
```
C:\hackathon\promptcrafters\
```

Project structure:
```
C:\hackathon\promptcrafters\
├── databricks\
│   └── notebooks\
│       ├── 01_generate_student_data.py
│       └── 02_student_risk_scoring.py
├── backend\
│   ├── pom.xml
│   └── src\main\
│       ├── java\com\promptcrafters\saarthi\
│       │   ├── SaarthiApplication.java
│       │   ├── controller\SaarthiController.java
│       │   └── service\StudentRiskService.java
│       └── resources\application.yml
├── ui\
│   └── index.html
└── docs\
    └── Saarthi_AI_Full_Guide.md
```

---

## 3. Environment Setup

```bash
choco install python --version=3.11 -y
choco install openjdk21 -y
choco install maven -y
choco install git -y
```

Verify:
```bash
java -version
```

---

## 4. Databricks Synthetic Data

Notebook: `01_generate_student_data`

```python
from dbldatagen import DataGenerator
from pyspark.sql.types import *

gen = (
    DataGenerator(spark,"students",rows=50000)
    .withColumn("student_id",StringType(),expr="uuid()")
    .withColumn("attendance_pct",IntegerType(),40,100)
    .withColumn("assessment_score",IntegerType(),0,100)
    .withColumn("engagement_score",IntegerType(),1,5)
    .withColumn("digital_access",StringType(),values=["Low","Medium","High"])
)

students_df = gen.build()
```

Notebook: `02_student_risk_scoring`

```python
from pyspark.sql.functions import when,col

risk_df = students_df.withColumn(
    "risk_score",
    0.4*(100-col("attendance_pct")) +
    0.3*(100-col("assessment_score")) +
    when(col("digital_access")=="Low",20)
    .when(col("digital_access")=="Medium",10)
    .otherwise(0)
)

risk_df.write.mode("overwrite").saveAsTable("student_risk_profile")
```

---

## 5. Spring Boot Backend (JDK 21)

pom.xml snippet:

```xml
<properties>
  <java.version>21</java.version>
</properties>
```

application.yml:

```yaml
spring:
  ai:
    azure:
      openai:
        api-key: YOUR_KEY
        endpoint: https://YOUR_ENDPOINT.openai.azure.com/
        chat:
          options:
            deployment-name: saarthi-chat
```

Controller:

```java
@RestController
@RequestMapping("/api")
public class SaarthiController {
  @PostMapping("/career")
  public String career(@RequestBody String profile) {
    return chatClient.prompt().user(profile).call().content();
  }
}
```

---

## 6. UI

```html
<textarea id="profile"></textarea>
<button onclick="career()">Career</button>
```

---

## 7. Copilot Studio

HTTP Action:
POST → /api/career
Body → {{user_message}}

---

## 8. Deployment

Build:
```bash
mvn clean package
```

Deploy JAR to Azure App Service (Java 21).

---

END
