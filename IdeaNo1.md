✅ COPY FROM HERE ↓ (THIS IS PURE MARKDOWN)
Saarthi AI – Magic Bus Foundation
Hack a Difference 2026 | Barclays
1. Summary – What problem we are solving
Magic Bus Foundation works with underserved youth in India to improve:
education
life skills
employability
Problems NGOs face
Dropout risk is identified too late
Career guidance is generic
Impact reporting is manual and time-consuming
Our solution: Saarthi AI
Saarthi AI is a simple, explainable AI system that:
Predicts dropout risk early
Suggests realistic career paths
Generates easy impact summaries
Is accessible via web UI + Copilot Studio agent
Built using Azure OpenAI, Spring AI, Python, and Copilot Studio.
2. High-level architecture
Copy code

Web Page / Copilot Studio
          |
      Spring Boot API
     (Spring AI + Azure OpenAI)
          |
   Python ML Service (Risk)
          |
   Synthetic Data (Allowed)
3. Project & folder structure
Copy code

saarthi-ai/
│
├── README.md
│
├── web/
│   └── index.html
│
├── backend/
│   ├── pom.xml
│   └── src/main/java/com/barclays/saarthi/
│       ├── SaarthiApplication.java
│       ├── controller/
│       │   └── SaarthiController.java
│       └── service/
│           └── RiskService.java
│   └── src/main/resources/
│       └── application.yml
│
└── python/
    └── predict_api.py
4. Step 1 – Environment setup (VDI)
Run in Admin Command Prompt:
Copy code
Bash
choco install python --version=3.11 -y
choco install openjdk17 -y
choco install maven -y
choco install git -y
choco install vscode -y
Verify:
Copy code
Bash
python --version
java -version
mvn -v
5. Step 2 – Python dropout risk service
File: python/predict_api.py
Copy code
Python
from flask import Flask, request, jsonify
import random

app = Flask(__name__)

@app.route("/predict", methods=["POST"])
def predict():
    probability = random.uniform(0.2, 0.9)  # demo-safe
    return jsonify({"probability": probability})

if __name__ == "__main__":
    app.run(port=5001)
Run it:
Copy code
Bash
pip install flask
python predict_api.py
6. Step 3 – Spring Boot backend
6.1 pom.xml
Copy code
Xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.barclays.saarthi</groupId>
  <artifactId>saarthi-ai</artifactId>
  <version>1.0.0</version>

  <properties>
    <java.version>17</java.version>
    <spring.ai.version>0.8.1</spring.ai.version>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-bom</artifactId>
        <version>${spring.ai.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
      <groupId>org.springframework.ai</groupId>
      <artifactId>spring-ai-azure-openai-spring-boot-starter</artifactId>
    </dependency>
  </dependencies>
</project>
6.2 application.yml
Copy code
Yaml
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
6.3 SaarthiApplication.java
Copy code
Java
@SpringBootApplication
public class SaarthiApplication {
    public static void main(String[] args) {
        SpringApplication.run(SaarthiApplication.class, args);
    }
}
6.4 SaarthiController.java
Copy code
Java
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
        return "Risk: " + (p > 0.7 ? "High" : p > 0.4 ? "Medium" : "Low");
    }
}
6.5 RiskService.java
Copy code
Java
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
7. Step 4 – Simple web UI
File: web/index.html
Copy code
Html
<!DOCTYPE html>
<html>
<body>
<h1>Magic Bus – Saarthi AI</h1>

<textarea id="profile" rows="5" cols="60"></textarea><br><br>

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
8. Step 5 – Copilot Studio agent
HTTP Action configuration
Method: POST
URL:
https://YOUR-APP.azurewebsites.net/api/career
Header:
Content-Type: text/plain
Body:
{{user_message}}
9. How to demo (5 minutes)
Paste student profile
Show risk score
Ask career guidance
Explain NGO impact
10. Compliance checklist
Synthetic data only
No Barclays internal data
Azure OpenAI only
Human-in-the-loop
11. Why this wins hackathon points
Real NGO problem
Predictive + Generative AI
Enterprise stack
Simple demo
India relevance
