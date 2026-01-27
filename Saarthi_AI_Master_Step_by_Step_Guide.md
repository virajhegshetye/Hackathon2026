# Saarthi AI – Complete Step-by-Step Implementation Guide  
Hack a Difference 2026 | Barclays

---

## How to use this document (IMPORTANT)

This document is a MASTER BUILD GUIDE.
Follow sections in order. Create files exactly as instructed.

---

## 1. Problem and Solution

Magic Bus Foundation supports underserved youth.
Challenges:
- Dropout risk identified too late
- Generic career guidance
- Manual NGO reporting

Saarthi AI:
- Predicts dropout risk early
- Uses Azure OpenAI for career guidance
- Works via Web UI and Copilot Studio

---

## 2. Project Structure

```
Hackathon2026/
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

## 3. Environment Setup

```bash
choco install python --version=3.11 -y
choco install openjdk17 -y
choco install maven -y
choco install git -y
```

Verify:

```bash
python --version
java -version
mvn -v
```

---

## 4. Azure OpenAI Setup

1. Azure Portal → Create Azure OpenAI
2. Deploy model: gpt-35-turbo
3. Deployment name: saarthi-chat
4. Copy API key and endpoint

---

## 5. Python ML Service

Create `python/predict_api.py`

```python
from flask import Flask, jsonify
import random

app = Flask(__name__)

@app.route("/predict", methods=["POST"])
def predict():
    return jsonify({"probability": random.uniform(0.2, 0.9)})

if __name__ == "__main__":
    app.run(port=5001)
```

Run:

```bash
pip install flask
python python/predict_api.py
```

---

## 6. Spring Boot Backend

### pom.xml

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
        api-key: YOUR_KEY
        endpoint: https://YOUR_ENDPOINT.openai.azure.com/
        chat:
          options:
            deployment-name: saarthi-chat
```

### Controller

```java
@PostMapping("/career")
public String career(@RequestBody String profile) {
    return chatClient.prompt().user(profile).call().content();
}
```

---

## 7. Web UI

Create `web/index.html`

```html
<textarea id="profile"></textarea>
<button onclick="callApi('/api/career')">Career</button>
```

---

## 8. Copilot Studio

HTTP Action:
POST → /api/career
Body → {{user_message}}

---

## 9. Deployment

1. Build JAR
2. Create Azure App Service
3. Upload JAR

---

END
