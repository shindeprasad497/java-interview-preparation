# 34. Build Tools, Maven POM Anatomy, Multi-Module Projects & DevOps

> **Navigation**: [Master Index](README.md) | [Previous: Enterprise Security](33_Enterprise_Security_OWASP_Java.md) | [Next: Engineering Leadership](35_Engineering_Leadership_ADRs_Clean_Code.md)

---

## 📌 Chapter Overview
This module explores **Maven POM.xml Anatomy**, designing **Multi-Module Enterprise Maven Projects**, dependency convergence with Bill of Materials (BOM), resolving diamond dependency conflicts, and modern deployment strategies (**Blue-Green, Canary, Feature Toggles**).

---

## 1. Anatomy of a Maven `pom.xml` File

```
+-----------------------------------------------------------------------------------+
|                            MAVEN POM.XML CORE ANATOMY                             |
+-----------------------------------------------------------------------------------+
|  1. Coordinates:          <groupId>, <artifactId>, <version>, <packaging>         |
|  2. Inheritance:          <parent> (Points to spring-boot-starter-parent / root)  |
|  3. Modules:              <modules> (Lists sub-projects in a multi-module build)  |
|  4. Properties:           <properties> (Centralized versions & Java version)      |
|  5. Dependency Control:   <dependencyManagement> (Declares versions, NO JAR pull) |
|  6. Direct Dependencies:  <dependencies> (Actually pulls JARs onto classpath)     |
|  7. Build & Plugins:      <build> -> <plugins> (Compiler, Spring Boot Repackage)  |
+-----------------------------------------------------------------------------------+
```

### Q1. What is the difference between `<dependencyManagement>` and `<dependencies>`?
**Answer:**
- **`<dependencyManagement>`**: Declares dependency versions, configurations, and exclusions **centrally in the parent POM**, but does **NOT** actually download the JAR or add it to the child project classpath.
- **`<dependencies>`**: Directly pulls the declared JAR onto the project's build classpath.
  - In a child module, when you declare a dependency that exists in the parent's `<dependencyManagement>`, you **omit the `<version>` tag**! The child automatically inherits the version from the parent.

---

## 2. Multi-Module Enterprise Maven Project Architecture

```
 enterprise-application/ (Root Parent POM: packaging=pom)
 │
 ├── pom.xml                     <-- Parent POM (dependencyManagement, plugins, modules)
 │
 ├── common-domain/              <-- Module 1 (Entities, DTOs, Enums)
 │   └── pom.xml
 │
 ├── data-persistence/          <-- Module 2 (Spring Data JPA Repositories, Flyway)
 │   └── pom.xml (depends on: common-domain)
 │
 ├── business-service/           <-- Module 3 (Business Logic, Services, Kafka)
 │   └── pom.xml (depends on: data-persistence)
 │
 └── web-api-gateway/            <-- Module 4 (REST Controllers, Security, Actuator)
     └── pom.xml (depends on: business-service -> Generates executable JAR)
```

### Q2. How do you configure Root Parent and Child POMs in a Multi-Module Project?
**Answer:**

#### 1. Root Parent `pom.xml`:
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.enterprise.app</groupId>
    <artifactId>enterprise-application</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging> <!-- Must be 'pom' for multi-module parent! -->

    <modules>
        <module>common-domain</module>
        <module>data-persistence</module>
        <module>business-service</module>
        <module>web-api-gateway</module>
    </modules>

    <properties>
        <java.version>21</java.version>
        <spring.boot.version>3.3.0</spring.boot.version>
        <lombok.version>1.18.32</lombok.version>
    </properties>

    <!-- Centralized Dependency Management across all modules -->
    <dependencyManagement>
        <dependencies>
            <!-- Spring Boot BOM -->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring.boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <!-- Internal Sub-Module Coordinates -->
            <dependency>
                <groupId>com.enterprise.app</groupId>
                <artifactId>common-domain</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>com.enterprise.app</groupId>
                <artifactId>data-persistence</artifactId>
                <version>${project.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

#### 2. Child Module `business-service/pom.xml`:
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.enterprise.app</groupId>
        <artifactId>enterprise-application</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>

    <artifactId>business-service</artifactId>
    <packaging>jar</packaging>

    <dependencies>
        <!-- Inter-module sibling dependency (Version omitted; inherited from parent!) -->
        <dependency>
            <groupId>com.enterprise.app</groupId>
            <artifactId>data-persistence</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
    </dependencies>
</project>
```

---

## 3. Diamond Dependency Conflict Resolution

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

### Q3. How do you resolve Maven diamond dependency conflicts in enterprise projects?
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

## 4. Deployment Strategies: Blue-Green vs. Canary

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

### Q4. Compare Blue-Green vs. Canary Deployments.
**Answer:**

| Strategy | Mechanism | Rollback Speed | Infrastructure Cost |
| :--- | :--- | :--- | :--- |
| **Blue-Green** | Two identical production environments. Router instantly flips traffic from Blue to Green. | **Instant (Seconds)** via router flip | High (Requires $2\times$ hardware capacity) |
| **Canary** | Deploys new version to small subset of pods (5%). Shifts traffic gradually if error rates remain nominal. | Fast (Kill canary pods) | Low (Incremental resource allocation) |
| **Rolling Update**| Progressively replaces old pods with new pods one by one in Kubernetes. | Moderate (Rollout undo) | Lowest (Zero extra idle hardware) |

---

## 5. Feature Flags & Trunk-Based Development

### Q5. How do Feature Toggles (Togglz / LaunchDarkly) enable continuous delivery?
**Answer:**
Feature flags allow engineering teams to merge incomplete features directly into `main` without exposing unready logic to end users:
- **Decouples Deployment from Release**: Code is deployed to production continuously, but activated via feature flags dynamically without redeployment.
- **Circuit Breaker Kill-Switch**: If a new feature causes high latency, it can be instantly disabled via web dashboard in $<1\text{s}$.

---

> **Next Chapter**: [35 Engineering Leadership, ADRs & Monolith Modernization](35_Engineering_Leadership_ADRs_Clean_Code.md)
