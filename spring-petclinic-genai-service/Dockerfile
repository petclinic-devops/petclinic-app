# Sử dụng JRE để chạy Spring Boot
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8084

ENV OPENAI_API_KEY=""
ENV AZURE_OPENAI_KEY=""
ENV AZURE_OPENAI_ENDPOINT=""

# Chạy Spring Boot
ENTRYPOINT ["java", "-jar", "app.jar"]