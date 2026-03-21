# Stage 1 — Build stage
# Use full JDK to compile and package the application
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app

# Copy maven wrapper first (layer caching — only re-runs if these change)
COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .

# Download all dependencies (cached layer — only re-runs if pom.xml changes)
RUN ./mvnw dependency:go-offline -B

# Copy source code (this layer re-runs on every code change)
COPY src src

# Build the jar, skip tests (tests run in Jenkins pipeline separately)
RUN ./mvnw package -DskipTests -B

# Stage 2 — Runtime stage
# Use smaller JRE image — no build tools, smaller attack surface
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Security best practice — never run as root in production
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy only the built jar from build stage
# Nothing else — no source code, no Maven, no JDK
COPY --from=build /app/target/accounts-0.0.1-SNAPSHOT.jar app.jar

# Give ownership to non-root user
RUN chown appuser:appgroup app.jar

# Switch to non-root user
USER appuser

# Document which port the app listens on
EXPOSE 8080

# Run the application
ENTRYPOINT ["java", "-jar", "app.jar"]
