# Spring AI & Generative AI Engineering in Java

> **Navigation**: [Master Index](README.md) | [Previous: Microservices Architecture](15_Microservices_Architecture.md) | [Study Roadmap](00_Study_Roadmap.md)

---

## 1. Spring AI Architecture & Abstraction Layer

```
+-----------------------------------------------------------------------------------+
|                            SPRING AI ARCHITECTURE                                 |
+-----------------------------------------------------------------------------------+
|  [ Spring Boot Application Logic ]                                                |
|         |                                                                         |
|         v                                                                         |
|  [ ChatClient / StreamingChatClient ] (Fluent Java API)                           |
|         |                                                                         |
|         +---> [ Advisors Chain ] (Prompt Injection Guard, Context Injection)      |
|         |                                                                         |
|         +---> [ Prompt Templates & Structured Output Converters (JSON/Records) ]  |
|         |                                                                         |
|         +---> [ Model Clients: OpenAI | Anthropic | Gemini | Bedrock | Ollama ]  |
|         |                                                                         |
|         +---> [ Vector Store Abstraction: PGVector | Redis | Pinecone | Milvus ]  |
+-----------------------------------------------------------------------------------+
```

---

### Q1. How do you implement Retrieval Augmented Generation (RAG) using Spring AI?
**Answer:**
RAG enriches LLM queries by retrieving relevant enterprise context from a **Vector Database** before sending the prompt to the language model.

```java
@Service
public class CustomerSupportRagService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public CustomerSupportRagService(ChatClient.Builder chatClientBuilder, VectorStore vectorStore) {
        this.chatClient = chatClientBuilder.build();
        this.vectorStore = vectorStore;
    }

    public String answerInquiry(String userQuestion) {
        // 1. Semantic Similarity Search in Vector DB (PGVector / Redis)
        List<Document> similarDocuments = vectorStore.similaritySearch(
            SearchRequest.query(userQuestion).withTopK(3).withSimilarityThreshold(0.75)
        );

        String context = similarDocuments.stream()
            .map(Document::getContent)
            .collect(Collectors.joining("\n\n"));

        // 2. Prompt Template with Retrieved Context
        String systemPrompt = """
            You are an enterprise customer support assistant.
            Use ONLY the following retrieved context to answer the user inquiry.
            If the answer cannot be found in the context, reply "I cannot find this in our documentation."
            
            CONTEXT:
            {context}
            """;

        // 3. Execute LLM Call with Context Injection
        return chatClient.prompt()
            .system(sp -> sp.text(systemPrompt).param("context", context))
            .user(userQuestion)
            .call()
            .content();
    }
}
```

---

## 2. Function Calling / Tool Calling in Spring AI

### Q2. How do you enable an LLM to invoke your Java Spring Service methods dynamically?
**Answer:**
Spring AI allows you to register standard Java `@Bean` functional interfaces as **Tools** that the LLM can decide to execute when it requires external live data.

```java
// 1. Define Request / Response Records
public record WeatherRequest(String city) {}
public record WeatherResponse(String city, double temperatureCelsius, String condition) {}

// 2. Register Tool as a Spring Functional Bean
@Configuration
public class AiToolConfig {

    @Bean
    @Description("Fetch the real-time weather temperature and condition for a given city")
    public Function<WeatherRequest, WeatherResponse> weatherTool(WeatherService weatherService) {
        return request -> weatherService.getCurrentWeather(request.city());
    }
}

// 3. Bind Tool to ChatClient Invocation
@Service
public class AssistantService {
    private final ChatClient chatClient;

    public AssistantService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public String askAssistant(String prompt) {
        return chatClient.prompt()
            .user(prompt)
            // Model will automatically detect if it needs to call 'weatherTool'
            .functions("weatherTool")
            .call()
            .content();
    }
}
```

---

## 3. Structured Output & AI Safety

### Q3. How do you guarantee Structured JSON / Java Record outputs from an LLM?
**Answer:**
Use Spring AI's **`BeanOutputConverter`** to automatically configure schema definitions in the prompt and deserialize the LLM's response directly into a type-safe Java Record.

```java
public record OrderAnalysis(
    String orderId,
    boolean fraudRisk,
    double riskScore,
    List<String> flaggedReasons
) {}

@Service
public class FraudAnalysisService {
    private final ChatClient chatClient;

    public FraudAnalysisService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public OrderAnalysis analyzeOrder(Order order) {
        var outputConverter = new BeanOutputConverter<>(OrderAnalysis.class);

        String format = outputConverter.getFormat(); // Generates JSON schema instructions
        String userPrompt = "Analyze this transaction: " + order.toString() + "\n" + format;

        String response = chatClient.prompt()
            .user(userPrompt)
            .call()
            .content();

        return outputConverter.convert(response); // Type-safe Java Record conversion!
    }
}
```

---

> **Return to Master Index**: [README.md](README.md)
