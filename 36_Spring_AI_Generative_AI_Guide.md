# 36. Spring AI, Model Context Protocol (MCP), RAG & Autonomous Agents

> **Navigation**: [Master Index](README.md) | [Previous: Engineering Leadership](35_Engineering_Leadership_ADRs_Clean_Code.md) | [Next: Quick Revision Checklist](37_Quick_Revision_Checklist.md)

---

## 📌 Chapter Overview
This module is a comprehensive master guide to **Spring AI**, the **Model Context Protocol (MCP)**, **Production Retrieval-Augmented Generation (RAG)**, and building **Autonomous AI Agents (ReAct Pattern, Memory, Multi-Tool Calling, Multi-Agent Orchestration)** in modern Java applications.

---

## 1. Spring AI Core Architecture & The `ChatClient` API

```
+-----------------------------------------------------------------------------------+
|                            SPRING AI CORE ARCHITECTURE                            |
+-----------------------------------------------------------------------------------+
|  [ Spring Application Business Layer ]                                            |
|          |                                                                        |
|          v                                                                        |
|  [ ChatClient (Fluent Builder API) ]                                              |
|          |                                                                        |
|          +-----> [ ChatMemoryAdvisor ] <=======> [ In-Memory / Redis Chat Memory ]|
|          |       (Multi-Turn Conversation State)                                  |
|          |                                                                        |
|          +-----> [ QuestionAnswerAdvisor ] <===> [ VectorStore (PgVector/Milvus) ]|
|          |       (RAG Semantic Context Injection)                                 |
|          |                                                                        |
|          +-----> [ Tool / Function Callback ] <===> [ Spring @Service @Tool Beans]|
|          |       (Dynamic Function Calling)                                       |
|          |                                                                        |
|          +-----> [ Model Context Protocol (MCP) ] <===> [ External MCP Servers ]  |
|          |                                                                        |
|          v                                                                        |
|  [ Model Client Adapter (OpenAI / Anthropic / Gemini / Ollama / Bedrock) ]        |
|          |                                                                        |
|          v                                                                        |
|  [ LLM Cloud Provider / Local Model ]                                             |
+-----------------------------------------------------------------------------------+
```

### Q1. What is Spring AI and how does `ChatClient` simplify LLM integration?
**Answer:**
**Spring AI** provides a portable, cloud-agnostic abstraction layer for building Generative AI applications in Java. It decouples business logic from specific model providers (OpenAI, Anthropic, Google Gemini, Ollama, Amazon Bedrock).

#### Core Features of `ChatClient`:
- **Fluent API**: Chainable prompts, system prompts, user messages, and parameter tuning (`temperature`, `topP`).
- **Modular Advisors**: Intercepts requests/responses for RAG context injection, conversation memory, and token observability.
- **Structured Outputs**: Type-safe entity deserialization into Java Records/DTOs via `BeanOutputConverter<T>`.
- **Multimodal Inputs**: Supports text, images (`Media`), audio, and PDF documents.

```java
@Service
public class CustomerSupportAiService {

    private final ChatClient chatClient;

    public CustomerSupportAiService(ChatClient.Builder chatClientBuilder, VectorStore vectorStore) {
        this.chatClient = chatClientBuilder
            .defaultSystem("You are a helpful banking customer support assistant. Be concise and professional.")
            .defaultAdvisors(
                new MessageChatMemoryAdvisor(new InMemoryChatMemory()), // Conversation Memory
                new QuestionAnswerAdvisor(vectorStore)                   // RAG Context Advisor
            )
            .build();
    }

    public String handleInquiry(String conversationId, String userMessage) {
        return chatClient.prompt()
            .user(userMessage)
            .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
            .call()
            .content();
    }
}
```

---

## 2. Model Context Protocol (MCP) in Spring AI

```
+-----------------------------------------------------------------------------------+
|                        MODEL CONTEXT PROTOCOL (MCP) ARCHITECTURE                  |
+-----------------------------------------------------------------------------------+
|                                                                                   |
|  [ Spring Boot Host App ]                                                         |
|         |                                                                         |
|         v                                                                         |
|  [ Spring AI MCP Client ]                                                         |
|         |                                                                         |
|         +--- (Transport: STDIO / SSE over HTTP / JSON-RPC 2.0) ---+               |
|         |                                                         |               |
|         v                                                         v               |
|  [ Local MCP Server ]                                    [ Remote MCP Server ]    |
|  - PostgreSQL MCP Server (Schema/Queries)                - GitHub MCP Server      |
|  - Filesystem MCP Server (Read/Write)                    - Brave Search MCP       |
|  - Custom Enterprise Spring Boot MCP Server              - Slack MCP Server       |
|                                                                                   |
+-----------------------------------------------------------------------------------+
```

### Q2. What is the Model Context Protocol (MCP) and how does Spring AI implement it?
**Answer:**
The **Model Context Protocol (MCP)** is an open standard developed by Anthropic that standardizes how AI applications securely connect to external tools, databases, and context providers using **JSON-RPC 2.0**.

#### 3 Core MCP Primitives:
1. **Tools**: Executable functions that the LLM can invoke (e.g., query database, execute shell command, create GitHub issue).
2. **Resources**: Read-only contextual data exposed to the LLM (e.g., file contents, API schemas, logs).
3. **Prompts**: Pre-engineered prompt templates exposed by the server.

#### Transport Modes:
- **`stdio`**: Communicates with a local process via standard input/output (low latency, secure).
- **`SSE (Server-Sent Events) / HTTP`**: Communicates with remote servers over network endpoints.

---

### Q3. How do you create an MCP Server in Spring Boot and connect an MCP Client?
**Answer:**

#### 1. Creating a Spring Boot MCP Server (`spring-ai-mcp-server`):
```java
@Configuration
public class PaymentMcpServerConfig {

    // Expose a Spring Service method as a standardized MCP Tool
    @Bean
    public ToolCallback paymentStatusTool(PaymentService paymentService) {
        return FunctionToolCallback.builder("checkPaymentStatus", paymentService::getStatus)
            .description("Query the live settlement status of a transaction by payment reference ID")
            .inputType(PaymentStatusRequest.class)
            .build();
    }
}
```

#### 2. Consuming External MCP Tools in Spring AI Client:
```java
@Configuration
public class McpClientConfig {

    @Bean
    public McpSyncClient postgresMcpClient() {
        // Connect to a local PostgreSQL MCP server via STDIO transport
        ServerParameters params = ServerParameters.builder("npx")
            .args("-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost:5432/mydb")
            .build();

        McpTransport transport = new StdioClientTransport(params);
        McpSyncClient client = McpClient.sync(transport).build();
        client.initialize();
        return client;
    }

    @Bean
    public ChatClient aiAssistant(ChatClient.Builder builder, McpSyncClient mcpClient) {
        // Register all MCP tools from the PostgreSQL server into the ChatClient!
        return builder
            .defaultTools(new McpFunctionCallback(mcpClient))
            .build();
    }
}
```

---

## 3. Advanced Retrieval-Augmented Generation (RAG) Deep Dive

```
+-----------------------------------------------------------------------------------+
|                        PRODUCTION RAG INGESTION & QUERY PIPELINE                  |
+-----------------------------------------------------------------------------------+
|  INGESTION ETL PIPELINE (Offline / Batch):                                        |
|  [ PDF / Docs / DB ] ---> DocumentReader                                          |
|                                |                                                  |
|                                v                                                  |
|                      TokenTextSplitter (Chunking: 500 tokens, 50 overlap)         |
|                                |                                                  |
|                                v                                                  |
|                      EmbeddingModel (e.g., text-embedding-3-small -> 1536 dims)  |
|                                |                                                  |
|                                v                                                  |
|                      VectorStore (PgVector / Milvus / Redis / Qdrant)             |
|                                                                                   |
|  QUERY RETRIEVAL PIPELINE (Real-Time):                                            |
|  [ User Question ] ---> EmbeddingModel ---> Query Vector (1536 dims)              |
|                                                  |                                |
|                                                  v                                |
|                             VectorStore.similaritySearch(Top-K = 5)               |
|                                                  |                                |
|                                                  v                                |
|                                  [ Re-Ranker / Cross-Encoder ]                    |
|                                                  |                                |
|                                                  v                                |
|                         System Prompt + Retrieved Document Chunks                 |
|                                                  |                                |
|                                                  v                                |
|                                      [ LLM Generates Answer ]                     |
+-----------------------------------------------------------------------------------+
```

### Q4. Detail the full RAG Ingestion Pipeline with PgVector in Spring AI.
**Answer:**

```java
@Service
public class DocumentRagIngestionService {

    private final VectorStore vectorStore;
    private final EmbeddingModel embeddingModel;

    public DocumentRagIngestionService(VectorStore vectorStore, EmbeddingModel embeddingModel) {
        this.vectorStore = vectorStore;
        this.embeddingModel = embeddingModel;
    }

    public void ingestKnowledgeBase(Resource pdfResource) {
        // Step 1: Read raw documents (PDF, Markdown, HTML, JSON)
        PagePdfDocumentReader reader = new PagePdfDocumentReader(pdfResource);
        List<Document> rawDocuments = reader.get();

        // Step 2: Chunk documents into overlapping token windows
        TokenTextSplitter splitter = new TokenTextSplitter(
            500,  // chunk size (tokens)
            50,   // chunk overlap (preserves semantic context across boundaries)
            5,    // min chunk size
            10000,// max characters
            true
        );
        List<Document> splitDocuments = splitter.apply(rawDocuments);

        // Step 3: Enrich with Metadata for filtered retrieval
        for (Document doc : splitDocuments) {
            doc.getMetadata().put("source", pdfResource.getFilename());
            doc.getMetadata().put("ingested_at", Instant.now().toString());
        }

        // Step 4: Generate dense vector embeddings and write to PgVector
        vectorStore.accept(splitDocuments);
    }
}
```

---

### Q5. What are Advanced RAG Techniques (Hybrid Search, Re-Ranking, Multi-Query)?
**Answer:**
1. **Hybrid Search**: Combines **Dense Vector Similarity Search** (captures semantic meaning/intent) with **Sparse Lexical Search (BM25)** (captures exact keywords, part numbers, and error codes). Results are combined using Reciprocal Rank Fusion (RRF).
2. **Context Re-Ranking**: Uses a Cross-Encoder model to score and re-order top 20 candidate vector chunks down to the top 3 most relevant chunks before passing to the LLM, reducing token cost and eliminating hallucinations.
3. **Metadata Filtering**: Scopes semantic queries to specific users, tenants, or document types:
   ```java
   FilterExpressionBuilder b = new FilterExpressionBuilder();
   List<Document> results = vectorStore.similaritySearch(
       SearchRequest.query("Refund policy")
           .withTopK(3)
           .withSimilarityThreshold(0.75)
           .withFilterExpression(b.and(b.eq("tenant_id", "tenant_a"), b.eq("category", "billing")).build())
   );
   ```

---

## 4. Autonomous AI Agent Creation using Spring AI

```
+-----------------------------------------------------------------------------------+
|                        REAct (REASON + ACT) AGENT WORKFLOW LOOP                   |
+-----------------------------------------------------------------------------------+
|                                                                                   |
|  [ User Goal ] ---> (1. THOUGHT: Reason about current state & pick tool)          |
|                               |                                                   |
|                               v                                                   |
|                    (2. ACTION: Execute Tool via Spring Bean / MCP)                |
|                               |                                                   |
|                               v                                                   |
|                    (3. OBSERVATION: Inspect tool output data)                     |
|                               |                                                   |
|                               +--- Goal Achieved? ---+                            |
|                               /                      \                            |
|                            (NO)                      (YES)                        |
|                             /                          \                          |
|                 (Repeat Reason + Act Loop)       [ Final Answer to User ]         |
|                                                                                   |
+-----------------------------------------------------------------------------------+
```

### Q6. How do you implement an Autonomous AI Agent with Tool Calling and Memory in Spring AI?
**Answer:**

```java
// 1. Define Business Tools annotated with @Tool
@Component
public class AirlineAgentTools {

    @Autowired private BookingRepository bookingRepo;
    @Autowired private FlightService flightService;

    @Tool(description = "Lookup active flight booking details including seat and status by booking reference code")
    public BookingDto getBookingDetails(String bookingCode) {
        return bookingRepo.findByCode(bookingCode)
            .orElseThrow(() -> new IllegalArgumentException("Booking code not found: " + bookingCode));
    }

    @Tool(description = "Change the assigned seat number for a passenger booking")
    public boolean changeSeat(String bookingCode, String newSeatNumber) {
        return flightService.reassignSeat(bookingCode, newSeatNumber);
    }
}

// 2. Build the Autonomous ReAct Agent
@Service
public class AutonomousCustomerServiceAgent {

    private final ChatClient agentClient;

    public AutonomousCustomerServiceAgent(ChatClient.Builder builder, AirlineAgentTools tools) {
        this.agentClient = builder
            .defaultSystem("""
                You are an autonomous airline customer support agent.
                You have access to tools for querying bookings and changing seats.
                Follow the ReAct loop:
                1. Reason about what tool is required based on the user's intent.
                2. Call the tool to execute the action.
                3. Observe the tool result and form your final response.
                Never make up information. If a booking is not found, politely inform the user.
                """)
            .defaultTools(tools) // Registers all @Tool methods automatically!
            .defaultAdvisors(new MessageChatMemoryAdvisor(new InMemoryChatMemory()))
            .build();
    }

    public String runAgentTurn(String sessionId, String userMessage) {
        return agentClient.prompt()
            .user(userMessage)
            .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, sessionId))
            .call()
            .content();
    }
}
```

---

## 5. Type-Safe Structured Output (`BeanOutputConverter`)

```java
public record FlightAnalysisReport(
    String flightNumber,
    String departureCity,
    String arrivalCity,
    boolean isDelayed,
    int delayMinutes,
    List<String> weatherAlerts
) {}

@Service
public class FlightReportService {
    @Autowired private ChatClient chatClient;

    public FlightAnalysisReport generateReport(String rawFlightLogs) {
        // BeanOutputConverter injects JSON schema instructions and parses LLM output into Java Record!
        BeanOutputConverter<FlightAnalysisReport> converter = new BeanOutputConverter<>(FlightAnalysisReport.class);

        String response = chatClient.prompt()
            .user(u -> u.text("""
                Analyze the following flight logs and return structured JSON matching the schema:
                {rawLogs}
                
                {format}
                """)
                .param("rawLogs", rawFlightLogs)
                .param("format", converter.getFormat()))
            .call()
            .content();

        return converter.convert(response); // Returns 100% type-safe FlightAnalysisReport instance!
    }
}
```

---

> **Next Chapter**: [37 Quick Revision Checklist & Interview Cheat Sheet](37_Quick_Revision_Checklist.md)
