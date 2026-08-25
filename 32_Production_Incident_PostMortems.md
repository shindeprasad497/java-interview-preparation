# 32. Production Incident Post-Mortems & Live RCA

> **Navigation**: [Master Index](README.md) | [Previous: System Design Scenarios](31_System_Design_High_Scale_Scenarios.md) | [Next: Enterprise Security (OWASP)](33_Enterprise_Security_OWASP_Java.md)

---

## 📌 Chapter Overview
Real-world production incident case studies tested in **Senior & Lead Architecture Rounds**, covering **HikariCP connection pool starvation**, **100% CPU thread dump diagnosis**, **Cascading Microservice failure**, and **Netty off-heap memory leaks**.

---

## Incident 1: 504 Gateway Timeouts & HikariCP Pool Starvation

### 1. Incident Symptoms:
- Production API Gateway reported widespread **HTTP 504 Gateway Timeouts**.
- Spring Boot Actuator showed all 200 Tomcat worker threads in **`TIMED_WAITING`** inside `HikariDataSource.getConnection()`.
- Database server CPU was sitting idle at $<5\%$ utilization.

### 2. Root Cause Analysis (RCA):
A developer placed a slow external third-party Credit Check HTTP call ($1.5\text{s}$ latency) **inside a Spring `@Transactional` method**:

```java
// ❌ INCIDENT ROOT CAUSE: Holds DB connection idle for 1.5s while waiting for HTTP response!
@Service
public class LoanService {

    @Transactional
    public LoanResponse applyLoan(LoanRequest req) {
        User user = userRepo.findById(req.userId()).orElseThrow(); // Claims Hikari DB Connection!
        
        // 1.5s external network call while HOLDING DB connection lock!
        CreditScore score = creditBureauClient.getScore(user.getSsn());
        
        return loanRepo.save(new Loan(user, score));
    }
}
```
*At 150 requests/sec, all 10 HikariCP connections were claimed within 70ms and held idle while waiting for external network I/O, stalling all other database queries across the entire application!*

### 3. Production Fix:
Refactored the external HTTP call **outside** the transaction boundary:

```java
// ✅ PRODUCTION RESOLUTION: Database connection held for only ~2ms during write!
@Service
public class LoanService {

    public LoanResponse applyLoan(LoanRequest req) {
        // Step 1: External HTTP call runs with ZERO database connection held
        CreditScore score = creditBureauClient.getScore(req.ssn());
        
        // Step 2: Database transaction started ONLY for immediate DB persistence
        return persistLoanTransaction(req.userId(), score);
    }

    @Transactional
    protected LoanResponse persistLoanTransaction(Long userId, CreditScore score) {
        User user = userRepo.findById(userId).orElseThrow();
        return loanRepo.save(new Loan(user, score));
    }
}
```

---

## Incident 2: 100% CPU Spike & Thread Dump Diagnosis

### 1. Diagnosis Steps on Linux:
```bash
# Step 1: Find process ID with high CPU
top

# Step 2: Find the specific Linux thread consuming 100% CPU
top -H -p <PID>
# Output: PID 4598 consuming 99.8% CPU

# Step 3: Convert Linux Thread PID (4598) to Hexadecimal
printf '%x\n' 4598
# Output: 0x11f6 (This is the 'nid' in Java thread dumps!)

# Step 4: Capture Thread Dump and grep for the nid
jcmd <PID> Thread.print > threads.tdump
grep -A 20 "nid=0x11f6" threads.tdump
```

### 2. Thread Dump Output:
```
"pool-2-thread-4" #32 prio=5 os_prio=0 tid=0x00007f nid=0x11f6 runnable [0x00007f]
   java.lang.Thread.State: RUNNABLE
        at java.util.regex.Pattern$Loop.match(Pattern.java:4785)
        at java.util.regex.Pattern$Curly.match0(Pattern.java:4260)
        at com.example.service.SanitizationService.cleanInput(SanitizationService.java:28)
```
- **Root Cause**: Catastrophic Backtracking in a complex regular expression (`(a+)+$`) spinning in an exponential CPU loop.

---

> **Next Chapter**: [33 Enterprise Security, OWASP Top 10 for Java & Zero-Trust](33_Enterprise_Security_OWASP_Java.md)

