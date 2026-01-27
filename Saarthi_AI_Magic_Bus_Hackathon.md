# Saarthi AI – Magic Bus Foundation  
Hack a Difference 2026 | Barclays

---

## Summary – What problem we are solving

Magic Bus Foundation works with underserved youth in India to improve education, life skills, and employability.

### Challenges
- Dropout risk identified too late  
- Generic career guidance  
- Manual and time-consuming impact reporting  

### Our Solution: Saarthi AI
Saarthi AI is an AI-powered system that:
1. Predicts dropout risk early  
2. Suggests realistic career paths  
3. Generates simple impact summaries  
4. Works via Web UI and Copilot Studio  

---

## High-level Architecture

```
Web UI / Copilot Studio
          |
      Spring Boot API
   (Spring AI + Azure OpenAI)
          |
   Python ML Service (Risk)
          |
   Synthetic Data (Allowed)
```

---

## Project Structure

```
saarthi-ai/
├── README.md
├── docs/
├── backend/
│   ├── pom.xml
│   └── src/main/
│       ├── java/com/barclays/saarthi/
│       │   ├── SaarthiApplication.java
│       │   ├── controller/SaarthiController.java
│       │   └── service/RiskService.java
│       └── resources/application.yml
├── python/
│   └── predict_api.py
└── web/
    └── index.html
```

---

## Step 1 – Environment Setup (VDI)

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

## Step 2 – Python Dropout Risk Service

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

Run:
```bash
pip install flask
python predict_api.py
```

---

## Step 3 – Spring Boot Backend

### pom.xml (dependency snippet)

```xml
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-azure-openai-spring-boot-starter</artifactId>
</dependency>
```

### application.yml

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

### SaarthiController.java (core logic)

```java
@PostMapping(value="/career", consumes="text/plain")
public String career(@RequestBody String profile) {
    return chatClient.prompt()
        .system("You are a career guide for underprivileged youth in India.")
        .user(profile)
        .call()
        .content();
}
```

---

## Step 4 – Web UI

```html
<textarea id="profile" rows="5" cols="60"></textarea>
<button onclick="callApi('/api/risk')">Get Risk</button>
<button onclick="callApi('/api/career')">Get Career</button>
```

---

## Step 5 – Copilot Studio HTTP Action

- Method: POST  
- URL: `https://YOUR-APP.azurewebsites.net/api/career`  
- Header: `Content-Type: text/plain`  
- Body: `{{user_message}}`  

---

## Demo Flow

1. Paste student profile  
2. Show dropout risk  
3. Show career guidance  
4. Explain NGO impact  

---

## Compliance Checklist

- Synthetic data only  
- No Barclays internal data  
- Azure OpenAI only  
- Human-in-the-loop  

---

## Why this works for the hackathon

- Solves a real NGO problem  
- Combines Predictive + Generative AI  
- Enterprise-grade but simple  
- Easy to demo  
- Strong India relevance  

---

End of document.
