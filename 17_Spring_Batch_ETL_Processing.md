# 17. Spring Batch & High-Throughput ETL Processing

> **Navigation**: [Master Index](README.md) | [Previous: Spring Security & OAuth2](16_Spring_Security_6_OAuth2.md) | [Next: Spring Modulith](18_Spring_Modulith_Events.md)

---

## 📌 Chapter Overview
This module explores **Spring Batch 5 (Spring Boot 3)** architecture, chunk-oriented processing (`ItemReader`, `ItemProcessor`, `ItemWriter`), transaction boundaries, fault-tolerant skip/retry policies, and **parallel step partitioning**.

---

## 1. Spring Batch Architecture & Component Hierarchy

```
                                +-------------------+
                                |    JobLauncher    |
                                +-------------------+
                                          |
                                          v
                                +-------------------+ <======> [ JobRepository ]
                                |        Job        |          (Stores JobExecution,
                                +-------------------+           StepExecution state in DB)
                                          |
                        +-----------------+-----------------+
                        |                                   |
                        v                                   v
                +---------------+                   +---------------+
                |    Step 1     |                   |    Step 2     |
                +---------------+                   +---------------+
                        |
            +-----------+-----------+
            | Chunk-Oriented Model  |
            +-----------------------+
            |  [ ItemReader ]       | ---> Reads 1 item at a time
            |  [ ItemProcessor ]    | ---> Transforms 1 item at a time (Returns null to filter)
            |  [ ItemWriter ]       | ---> Writes Chunk of N items in a SINGLE DB Transaction
            +-----------------------+
```

---

## 2. Chunk-Oriented Processing in Action

### Q1. How does Chunk-Oriented processing handle transactions and memory?
**Answer:**
Instead of reading 1,000,000 records into JVM memory at once (which causes `OutOfMemoryError`), Spring Batch processes records in small, deterministic **Chunks** (e.g., `chunk(100)`):
1. The **`ItemReader`** reads items sequentially until the chunk size (100) is reached.
2. The **`ItemProcessor`** transforms/validates each item individually.
3. The **`ItemWriter`** receives the entire list of 100 transformed items and commits them to the database in a **single database transaction**.
4. If a failure occurs, only the current chunk rolls back, and state is recorded in `JobRepository` for safe restart.

```java
@Configuration
public class UserBatchConfig {

    @Bean
    public Job userImportJob(JobRepository jobRepository, Step processUserChunkStep) {
        return new JobBuilder("userImportJob", jobRepository)
            .start(processUserChunkStep)
            .build();
    }

    @Bean
    public Step processUserChunkStep(
            JobRepository jobRepository,
            PlatformTransactionManager txManager,
            ItemReader<UserCsvDto> csvReader,
            ItemProcessor<UserCsvDto, UserEntity> userProcessor,
            ItemWriter<UserEntity> jpaWriter) {

        return new StepBuilder("processUserChunkStep", jobRepository)
            .<UserCsvDto, UserEntity>chunk(100, txManager) // Process in chunks of 100 per transaction
            .reader(csvReader)
            .processor(userProcessor)
            .writer(jpaWriter)
            // Fault tolerance: Skip malformed CSV lines up to 50 times
            .faultTolerant()
            .skip(FlatFileParseException.class)
            .skipLimit(50)
            // Retry transient DB lock conflicts up to 3 times
            .retry(DeadlockLoserDataAccessException.class)
            .retryLimit(3)
            .build();
    }
}
```

---

## 3. High-Throughput Scaling: Partitioned Steps

### Q2. How do you scale Spring Batch to process millions of records in parallel?
**Answer:**
Using **Partitioned Steps**, a master step divides the dataset into discrete slices (partitions) using a `Partitioner` (e.g., by range of IDs: `1-100k`, `100k-200k`, `200k-300k`). Worker threads process each partition concurrently.

```java
@Bean
public Step masterStep(JobRepository jobRepository, Step workerStep, Partitioner partitioner) {
    return new StepBuilder("masterStep", jobRepository)
        .partitioner("workerStep", partitioner)
        .step(workerStep)
        .gridSize(8) // 8 parallel worker threads
        .taskExecutor(batchTaskExecutor())
        .build();
}

@Bean
public TaskExecutor batchTaskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(8);
    executor.setMaxPoolSize(16);
    executor.setThreadNamePrefix("batch-worker-");
    executor.initialize();
    return executor;
}
```

---

> **Next Chapter**: [18 Spring Modulith & Event-Driven Architecture](18_Spring_Modulith_Events.md)

