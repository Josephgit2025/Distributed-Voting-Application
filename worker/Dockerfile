### Stage 1 : Builder ###
FROM maven:3.9.6-eclipse-temurin-21-alpine AS builder

WORKDIR /build

COPY pom.xml .
RUN mvn dependency:resolve

COPY src ./src
RUN mvn package


### Stage 2 : Runtime ###
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

COPY --from=builder /build/target/worker-jar-with-dependencies.jar .

CMD ["java", "-jar", "worker-jar-with-dependencies.jar"]
