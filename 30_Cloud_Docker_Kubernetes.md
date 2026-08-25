# 30. Cloud-Native Deployments: Docker & Kubernetes

> **Navigation**: [Master Index](README.md) | [Previous: Observability & Testing](29_Observability_Tracing_Testing.md) | [Next: System Design Scenarios](31_System_Design_High_Scale_Scenarios.md)

---

## 📌 Chapter Overview
This module covers containerizing Spring Boot applications with **Docker Multi-Stage Builds & LayerTools**, configuring **Kubernetes Liveness / Readiness Probes**, and managing **Graceful Shutdown**.

---

## 1. Multi-Stage Dockerfile with Spring Boot LayerTools

### Q1. Why is single-layer `COPY target/*.jar app.jar` an anti-pattern?
**Answer:**
A fat JAR mixes external dependencies (which rarely change) with application business classes (which change constantly). Copying the whole JAR in one Docker layer forces Docker to re-download the entire 100 MB+ layer on every deployment.

#### Production Solution: Spring Boot LayerTools
Splits the JAR into 4 distinct layers:
1. `dependencies` (Third-party JARs) $\rightarrow$ Cached indefinitely!
2. `spring-boot-loader`
3. `snapshot-dependencies`
4. `application` (Only your compiled `.class` files, ~200 KB) $\rightarrow$ Rebuilt in seconds!

```dockerfile
# Stage 1: Extract layers using Layertools
FROM eclipse-temurin:21-jre-alpine AS builder
WORKDIR /builder
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} application.jar
RUN java -Djarmode=layertools -jar application.jar extract

# Stage 2: Final Minimal Runtime Image
FROM eclipse-temurin:21-jre-alpine
WORKDIR /application

# Run as non-privileged security user
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

# Copy layers ordered from least-frequently to most-frequently changed
COPY --from=builder /builder/dependencies/ ./
COPY --from=builder /builder/spring-boot-loader/ ./
COPY --from=builder /builder/snapshot-dependencies/ ./
COPY --from=builder /builder/application/ ./

EXPOSE 8080
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "org.springframework.boot.loader.launch.JarLauncher"]
```

---

## 2. Kubernetes Deployment Manifest with Probes & Graceful Shutdown

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max pods created above target during deployment
      maxUnavailable: 0  # Zero downtime deployment guarantee
  template:
    spec:
      terminationGracePeriodSeconds: 45 # Give Spring Boot 30s to finish active requests
      containers:
        - name: order-service
          image: registry.example.com/order-service:v1.2.0
          resources:
            requests:
              memory: "1Gi"
              cpu: "500m"
            limits:
              memory: "1.5Gi"
              cpu: "1000m"
          # 1. Startup Probe: Allows slow startup without killing pod prematurely
          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            failureThreshold: 30
            periodSeconds: 2
          # 2. Liveness Probe: Restarts pod if deadlocked or JVM frozen
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            periodSeconds: 10
          # 3. Readiness Probe: Removes pod from K8s Service load balancer if busy
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            periodSeconds: 5
```

#### Graceful Shutdown Configuration (`application.yml`):
```yaml
server:
  shutdown: graceful # Rejects new incoming HTTP requests, finishes active in-flight requests

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

---

> **Next Chapter**: [31 System Design: High-Scale Scenarios & Protocols](31_System_Design_High_Scale_Scenarios.md)

