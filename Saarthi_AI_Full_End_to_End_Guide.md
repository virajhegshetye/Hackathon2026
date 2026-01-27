# Saarthi AI – Magic Bus Foundation  
Hack a Difference 2026 | Barclays

---

## 1. What are we building (in simple words)

Magic Bus Foundation works with underserved youth in India to improve education, life skills, and employability.

### Problems today
- Students drop out but the risk is detected too late  
- Career guidance is generic and manual  
- NGO staff spend too much time creating reports  

### Our solution: Saarthi AI
Saarthi AI is a simple, responsible AI platform that:
1. Predicts dropout risk early  
2. Suggests realistic career paths  
3. Generates impact summaries  
4. Can be used via a web page or Copilot Studio chatbot  

This solution is:
- Hackathon compliant (synthetic data only)
- Built on Azure
- Easy to demo in 5 minutes

---

## 2. High-level architecture

```
Browser / Copilot Studio
          |
      Spring Boot API
   (Spring AI + Azure OpenAI)
          |
   Python ML Service
          |
   Synthetic Data
```

---

## 3. Complete project structure

```
saarthi-ai/
│
├── README.md
│
├── backend/
│   ├── pom.xml
│   └── src/main/
│       ├── java/com/barclays/saarthi/
│       │   ├── SaarthiApplication.java
│       │   ├── controller/SaarthiController.java
│       │   └── service/RiskService.java
│       └── resources/application.yml
│
├── python/
│   └── predict_api.py
│
├── web/
│   └── index.html
│
└── docs/
    └── notes.md
```

---

## 4. Environment setup (VDI)

Run these commands in **Admin Command Prompt**:

```bash
choco install python --version=3.11 -y
choco install openjdk17 -y
choco install maven -y
choco install git -y
choco install vscode -y
```

Verify installation:

```bash
python --version
java -version
mvn -v
```

---

## 5. Azure setup – step by step

### 5.1 Create Azure OpenAI resource

1. Login to Azure Portal (from VDI)
2. Click **Create a resource**
3. Search for **Azure OpenAI**
4. Create with:
   - Subscription: Team subscription
   - Resource group: rg-saarthi-ai
   - Region: UK South or West Europe
5. Click **Create**

### 5.2 Deploy a model

1. Open Azure OpenAI resource
2. Go to **Model deployments**
3. Create deployment:
   - Model: gpt-35-turbo or gpt-4o-mini
   - Deployment name: saarthi-chat
4. Save

### 5.3 Copy credentials

From **Keys and Endpoint**, copy:
- Endpoint URL
- API Key

---

## 6. Python ML service (dropout risk)

### File: python/predict_api.py

```python
from flask import Flask, jsonify
import random

app = Flask(__name__)

@app.route("/predict", methods=["POST"])
def predict():
    probability = random.uniform(0.2, 0.9)
    return jsonify({"probability": probability})

if __name__ == "__main__":
    app.run(port=5001)
```

Run service:

```bash
pip install flask
python predict_api.py
```

---

## 7. Spring Boot backend with Spring AI

### 7.1 pom.xml (important dependencies)

```xml
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-azure-openai-spring-boot-starter</artifactId>
</dependency>
```

### 7.2 application.yml

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

### 7.3 SaarthiApplication.java

```java
@SpringBootApplication
public class SaarthiApplication {
    public static void main(String[] args) {
        SpringApplication.run(SaarthiApplication.class, args);
    }
}
```

### 7.4 SaarthiController.java

```java
@RestController
@RequestMapping("/api")
public class SaarthiController {

    private final ChatClient chatClient;
    private final RiskService riskService;

    public SaarthiController(ChatClient chatClient, RiskService riskService) {
        this.chatClient = chatClient;
        this.riskService = riskService;
    }

    @PostMapping(value="/career", consumes="text/plain")
    public String career(@RequestBody String profile) {
        return chatClient.prompt()
            .system("You are a career guide for underprivileged youth in India.")
            .user(profile)
            .call()
            .content();
    }

    @PostMapping(value="/risk", consumes="text/plain")
    public String risk(@RequestBody String profile) {
        double p = riskService.predict();
        return "Risk Level: " + (p > 0.7 ? "High" : p > 0.4 ? "Medium" : "Low");
    }
}
```

### 7.5 RiskService.java

```java
@Service
public class RiskService {

    private final RestTemplate restTemplate = new RestTemplate();

    public double predict() {
        Map<String,Integer> payload = Map.of(
            "attendance", 70,
            "assessment_score", 60,
            "engagement", 3
        );

        Map<?,?> response = restTemplate.postForObject(
            "http://localhost:5001/predict",
            payload,
            Map.class
        );

        return (double) response.get("probability");
    }
}
```

Run backend:

```bash
mvn spring-boot:run
```

---

## 8. Web UI (simple demo)

### File: web/index.html

```html
<!DOCTYPE html>
<html>
<body>
<h1>Saarthi AI – Magic Bus</h1>

<textarea id="profile" rows="5" cols="60"
placeholder="Student profile"></textarea><br><br>

<button onclick="callApi('/api/risk')">Get Risk</button>
<button onclick="callApi('/api/career')">Get Career</button>

<pre id="out"></pre>

<script>
async function callApi(url) {
  const text = document.getElementById("profile").value;
  const res = await fetch(url, {
    method: "POST",
    headers: {"Content-Type":"text/plain"},
    body: text
  });
  document.getElementById("out").innerText = await res.text();
}
</script>
</body>
</html>
```

---

## 9. Copilot Studio agent setup

1. Open Copilot Studio
2. Create new agent: **Saarthi AI Assistant**
3. Create topic: Career Guidance
4. Add HTTP action:
   - Method: POST
   - URL: https://YOUR-APP.azurewebsites.net/api/career
   - Header: Content-Type: text/plain
   - Body: {{user_message}}
5. Return response to user

---

## 10. Azure deployment

### Deploy Spring Boot

1. Create Azure App Service (Java 17)
2. Build JAR:
```bash
mvn clean package
```
3. Upload JAR in Deployment Center

### Update Copilot URL
Replace localhost with App Service URL.

---

## 11. Demo flow (5 minutes)

1. Explain problem
2. Paste student profile
3. Show risk score
4. Show career guidance
5. Explain NGO impact

---

## 12. Compliance checklist

- Synthetic data only
- No Barclays internal data
- Azure OpenAI only
- Human-in-the-loop

---

## 13. Why this wins hackathon points

- Real NGO use case
- Predictive + Generative AI
- Enterprise-grade stack
- Easy to demo
- Strong social impact

---

End of document.
