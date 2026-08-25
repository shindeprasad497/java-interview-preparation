# 36. Spring AI & Enterprise Generative AI Engineering

> **Navigation**: [Master Index](README.md) | [Previous: Engineering Leadership](35_Engineering_Leadership_ADRs_Clean_Code.md) | [Next: Quick Revision Checklist](37_Quick_Revision_Checklist.md)

---

## 📌 Chapter Overview
This module explores **Spring AI**, integrating Large Language Models (LLMs) into Java applications, implementing **Retrieval Augmented Generation (RAG)** with Vector Stores, and enabling **Dynamic Function / Tool Calling**.

---

## 1. Spring AI Architecture & The `ChatClient` API

```
+-----------------------------------------------------------------------------------+
|                            SPRING AI ARCHITECTURE                                 |
+-----------------------------------------------------------------------------------+
|  [ Spring Application ]                                                           |
|          |                                                                        |
|          v                                                                        |
|  [ ChatClient (Fluent Builder API) ]                                              |
|          |                                                                        |
|          +-----> [ QuestionAnswerAdvisor ] <===> [ VectorStore (PgVector / Redis)]|
|          |       (RAG Context Injection)                                          |
|          |                                                                        |
|          +-----> [ Tool / Function Callback ] <===> [ Spring @Service Methods ]   |
|          |                                                                        |
|          v                                                                        |
|  [ Model Client Adapter (OpenAI / Anthropic / Gemini / Ollama) ]                  |
|          |                                                                        |
|          v                                                                        |
|  [ LLM Cloud Endpoint ]                                                           |
+-----------------------------------------------------------------------------------+
```

### Q1. How do you implement Retrieval Augmented Generation (RAG) with Spring AI?
**Answer:**

```java
@RestController
@RequestMapping("/api/v1/support")
public class SupportAiController {

    private final ChatClient chatClient;

    public SupportAiController(ChatClient.Builder builder, VectorStore vectorStore) {
        this.chatClient = builder
            // Automatically queries PgVector vector store to retrieve matching documentation!
            .defaultAdvisors(new QuestionAnswerAdvisor(vectorStore))
            .build();
    }

    @PostMapping("/ask")
    public String askAiSupport(@RequestBody String userQuestion) {
        return chatClient.prompt()
            .user(userQuestion)
            .call()
            .content();
    }
}
```

---

## 2. Dynamic Tool Calling / Function Calling

### Q2. How do you allow an LLM to dynamically invoke Java Spring methods?
**Answer:**

```java
@Configuration
public class AiToolConfig {

    // Define standard Java Function bean annotated with @Description for the LLM
    @Bean
    @Description("Fetch real-time flight booking status and gate info by booking reference code")
    public Function<FlightStatusRequest, FlightStatusResponse> flightStatusFunction(FlightService flightService) {
        return request -> flightService.lookupStatus(request.bookingCode());
    }
}

// In Service:
public String handleTravelerQuery(String query) {
    return chatClient.prompt()
        .user(query)
        .functions("flightStatusFunction") // LLM decides autonomously when to invoke this Java method!
        .call()
        .content();
}
```

---

> **Next Chapter**: [37 Quick Revision Checklist & Interview Cheat Sheet](37_Quick_Revision_Checklist.md)

