# 34. Build Tools, CI/CD Pipelines & Deployment Strategies

> **Navigation**: [Master Index](README.md) | [Previous: Enterprise Security](33_Enterprise_Security_OWASP_Java.md) | [Next: Engineering Leadership](35_Engineering_Leadership_ADRs_Clean_Code.md)

---

## 📌 Chapter Overview
This module explores **Maven & Gradle multi-module dependency convergence**, Bill of Materials (BOM), resolving diamond dependency conflicts, and modern deployment strategies (**Blue-Green, Canary, Feature Toggles**).

---

## 1. Maven Dependency Management & Conflict Resolution

```
+-------------------------------------------------------------------------------+
|                       DIAMOND DEPENDENCY CONFLICT RESOLUTION                  |
+-------------------------------------------------------------------------------+
|                       [ App: my-service ]                                     |
|                         /            \                                        |
|                        v              v                                       |
|             [ Module A v1.0 ]    [ Module B v2.0 ]                            |
|                     |                    |                                    |
|                     v                    v                                    |
|             [ Library X v1.2 ]   [ Library X v2.5 ] <--- CONFLICT!            |
|                                                                               |
|  Maven Rule 1: Nearest Definition in Dependency Tree Wins                     |
|  Maven Rule 2: First Declaration in POM Wins if depths are equal              |
+-------------------------------------------------------------------------------+
```

### Q1. How do you resolve Maven diamond dependency conflicts in enterprise projects?
**Answer:**
1. **Diagnosis**: Run `mvn dependency:tree -Dverbose -Dincludes=com.example:conflict-lib` to inspect the exact path of conflicting versions.
2. **Resolution A (Explicit `<dependencyManagement>` BOM)**: Override the conflicting version centrally in the root parent POM.
3. **Resolution B (Exclusions)**: Exclude the transitive dependency from the pulling module:
   ```xml
   <dependency>
       <groupId>com.example</groupId>
       <artifactId>module-a</artifactId>
       <exclusions>
           <exclusion>
               <groupId>com.example</groupId>
               <artifactId>conflict-lib</artifactId>
           </exclusion>
       </exclusions>
   </dependency>
   ```

---

## 2. Deployment Strategies: Blue-Green vs. Canary

```
 BLUE-GREEN DEPLOYMENT (Instant Switch):
 [ Load Balancer ] ---> [ Blue Environment (Active v1.0) ]  (100% Traffic)
                        [ Green Environment (Staged v2.0)]  (0% Traffic)
        | (Switch Router Pointer)
        v
 [ Load Balancer ] ---> [ Green Environment (Active v2.0)]  (100% Traffic)

 CANARY DEPLOYMENT (Progressive Traffic Shift):
 [ Load Balancer ] ---> 95% Traffic ---> [ Stable Pods v1.0 ]
                  `--->  5% Traffic ---> [ Canary Pods v2.0 ] (Monitor error metrics!)
```

### Q2. Compare Blue-Green vs. Canary Deployments.
**Answer:**

| Strategy | Mechanism | Rollback Speed | Infrastructure Cost |
| :--- | :--- | :--- | :--- |
| **Blue-Green** | Two identical production environments. Router instantly flips traffic from Blue to Green. | **Instant (Seconds)** via router flip | High (Requires $2\times$ hardware capacity) |
| **Canary** | Deploys new version to small subset of pods (5%). Shifts traffic gradually if error rates remain nominal. | Fast (Kill canary pods) | Low (Incremental resource allocation) |
| **Rolling Update**| Progressively replaces old pods with new pods one by one in Kubernetes. | Moderate (Rollout undo) | Lowest (Zero extra idle hardware) |

---

## 3. Feature Flags & Trunk-Based Development

### Q3. How do Feature Toggles (Togglz / LaunchDarkly) enable continuous delivery?
**Answer:**
Feature flags allow engineering teams to merge incomplete features directly into `main` without exposing unready logic to end users:
- **Decouples Deployment from Release**: Code is deployed to production continuously, but activated via feature flags dynamically without redeployment.
- **Circuit Breaker Kill-Switch**: If a new feature causes high latency, it can be instantly disabled via web dashboard in $<1\text{s}$.

---

> **Next Chapter**: [35 Engineering Leadership, ADRs & Monolith Modernization](35_Engineering_Leadership_ADRs_Clean_Code.md)

