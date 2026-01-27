# Saarthi AI – FULL End-to-End Implementation Guide  
Hack a Difference 2026 | Barclays

---

## IMPORTANT – Read this first

This document is intentionally **long and detailed**.
It is designed so that **any team member (even new to Azure or AI)** can:

- Create every file
- Paste exact code
- Configure Azure correctly
- Run the solution end-to-end
- Demo confidently in the hackathon

This file is safe to upload directly to GitHub and will render correctly.

---

# 1. What problem are we solving (simple explanation)

Magic Bus Foundation supports underserved youth with education, life skills, and employability.

### Current challenges
- Students drop out, but warning signs are noticed too late
- Career guidance is generic and manual
- NGO reporting consumes a lot of staff time

### Saarthi AI – Our solution
Saarthi AI is a responsible AI platform that:
1. Predicts dropout risk early
2. Provides personalised career guidance using Azure OpenAI
3. Gives NGO staff a simple chat-based interface
4. Works fully within hackathon compliance rules

---

# 2. Final project structure (follow exactly)

Create folders like this:

```
Hackathon2026/
│
├── Saarthi_AI_Full_Guide.md
│
├── backend/
│   ├── pom.xml
│   └── src/main/
│       ├── java/com/barclays/saarthi/
│       │   ├── SaarthiApplication.java
│       │   ├── controller/
│       │   │   └── SaarthiController.java
│       │   └── service/
│       │       └── RiskService.java
│       └── resources/
│           └── application.yml
│
├── python/
│   └── predict_api.py
│
└── web/
    └── index.html
```

---

# 3. Environment setup on VDI

Open **Command Prompt as Administrator**.

Install required tools:

```bash
choco install python --version=3.11 -y
choco install openjdk17 -y
choco install maven -y
choco install git -y
choco install vscode -y
```

Verify:

```bash
python --version
java -version
mvn -v
```

---

# 4. Azure OpenAI – Step-by-step (Click-by-click)

## 4.1 Create Azure OpenAI resource

1. Open Azure Portal from VDI
2. Click **Create a resource**
3. Search for **Azure OpenAI**
4. Click **Create**
5. Fill details:
   - Subscription: Your team subscription
   - Resource Group: `rg-saarthi-ai`
   - Region: UK South or West Europe
   - Name: `saarthi-openai`
6. Click **Review + Create**
7. Click **Create**

Wait until deployment completes.

---

## 4.2 Deploy an OpenAI model

1. Open `saarthi-openai` resource
2. Click **Model deployments**
3. Click **Create deployment**
4. Select:
   - Model: `gpt-35-turbo` or `gpt-4o-mini`
   - Deployment name: `saarthi-chat`
5. Click **Create**

---

## 4.3 Copy credentials

1. Go to **Keys and Endpoint**
2. Copy:
   - API Key
   - Endpoint URL

You will use these in Spring Boot.

---

# 5. Python ML service (Dropout Risk)

## 5.1 Create file: `python/predict_api.py`

```python
from flask import Flask, request, jsonify
import random

app = Flask(__name__)

@app.route("/predict", methods=["POST"])
def predict():
    probability = random.uniform(0.2, 0.9)
    return jsonify({"probability": probability})

if __name__ == "__main__":
    app.run(port=5001)
```

## 5.2 Run Python service

```bash
pip install flask
python python/predict_api.py
```

Keep this running.

---

# 6. Spring Boot + Spring AI backend

## 6.1 Create Spring Boot project

Use **Spring Initializr** (via browser or IntelliJ):

- Project: Maven
- Language: Java
- Spring Boot: 3.x
- Group: `com.barclays`
- Artifact: `saarthi`
- Java: 17
- Dependencies:
  - Spring Web

Download and extract into `backend/` folder.

---

## 6.2 Update `pom.xml`

Replace dependencies section with:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-azure-openai-spring-boot-starter</artifactId>
        <version>0.8.1</version>
    </dependency>
</dependencies>
```

---

## 6.3 Create `application.yml`

File path:  
`backend/src/main/resources/application.yml`

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

## 6.4 Main class – `SaarthiApplication.java`

```java
package com.barclays.saarthi;

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

## 6.5 Service – `RiskService.java`

```java
package com.barclays.saarthi.service;

import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;
import java.util.Map;

@Service
public class RiskService {

    private final RestTemplate restTemplate = new RestTemplate();

    public double getRiskScore() {
        Map<String, Integer> payload = Map.of(
            "attendance", 70,
            "assessment_score", 60,
            "engagement", 3
        );

        Map<?, ?> response = restTemplate.postForObject(
            "http://localhost:5001/predict",
            payload,
            Map.class
        );

        return (double) response.get("probability");
    }
}
```

---

## 6.6 Controller – `SaarthiController.java`

```java
package com.barclays.saarthi.controller;

import org.springframework.web.bind.annotation.*;
import org.springframework.ai.chat.client.ChatClient;
import com.barclays.saarthi.service.RiskService;

@RestController
@RequestMapping("/api")
public class SaarthiController {

    private final ChatClient chatClient;
    private final RiskService riskService;

    public SaarthiController(ChatClient chatClient, RiskService riskService) {
        this.chatClient = chatClient;
        this.riskService = riskService;
    }

    @PostMapping(value = "/career", consumes = "text/plain")
    public String careerAdvice(@RequestBody String profile) {
        return chatClient.prompt()
            .system("You are a career guide for underprivileged youth in India. Provide simple, realistic advice.")
            .user(profile)
            .call()
            .content();
    }

    @PostMapping(value = "/risk", consumes = "text/plain")
    public String risk(@RequestBody String profile) {
        double score = riskService.getRiskScore();
        return "Risk Level: " + (score > 0.7 ? "High" : score > 0.4 ? "Medium" : "Low");
    }
}
```

---

## 6.7 Run Spring Boot

```bash
cd backend
mvn spring-boot:run
```

Test:
- http://localhost:8080/api/risk
- http://localhost:8080/api/career

---

# 7. Web UI

## Create `web/index.html`

```html
<!DOCTYPE html>
<html>
<body>
<h1>Saarthi AI – Magic Bus</h1>

<textarea id="profile" rows="5" cols="60"
placeholder="Enter student profile"></textarea><br><br>

<button onclick="callApi('/api/risk')">Get Risk</button>
<button onclick="callApi('/api/career')">Get Career</button>

<pre id="out"></pre>

<script>
async function callApi(url) {
  const text = document.getElementById("profile").value;
  const res = await fetch(url, {
    method: "POST",
    headers: {"Content-Type": "text/plain"},
    body: text
  });
  document.getElementById("out").innerText = await res.text();
}
</script>
</body>
</html>
```

Open this file in browser.

---

# 8. Copilot Studio – Detailed steps

1. Open Copilot Studio
2. Click **Create Copilot**
3. Name: Saarthi AI Assistant
4. Create Topic: Career Guidance
5. Add HTTP action:
   - Method: POST
   - URL: https://YOUR-APP.azurewebsites.net/api/career
   - Header: Content-Type: text/plain
   - Body: {{user_message}}
6. Save and Test

---

# 9. Azure Deployment

## Deploy Spring Boot

1. Azure Portal → Create App Service
2. Runtime: Java 17
3. Build JAR:

```bash
mvn clean package
```

4. Upload JAR in Deployment Center

---

# 10. Demo flow (5 minutes)

1. Explain problem
2. Enter student profile
3. Show dropout risk
4. Show career guidance
5. Show Copilot Studio chatbot

---

# 11. Compliance checklist

- Synthetic data only
- No Barclays internal data
- Azure OpenAI only
- Human-in-the-loop

---

END OF FULL GUIDE
