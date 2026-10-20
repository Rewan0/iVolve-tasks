
FROM amazoncorretto:17-alpine3.21-full

WORKDIR /app

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --chown=appuser:appgroup target/calculator.jar app.jar

USER appuser

CMD ["java", "-jar", "calculator.jar"]

