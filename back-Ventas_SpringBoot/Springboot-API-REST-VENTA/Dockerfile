# ============================================================
# Stage 1: BUILD
# Usa imagen Maven completa para compilar el proyecto
# ============================================================
FROM maven:3.9.6-eclipse-temurin-17-alpine AS builder

WORKDIR /app

# Copiar pom.xml primero para cachear descarga de dependencias
# Si el pom no cambia, Docker reutiliza esta capa aunque cambie el código
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copiar código fuente
COPY src ./src

# Compilar y empaquetar (sin tests para agilizar pipeline)
RUN mvn clean package -DskipTests -B

# ============================================================
# Stage 2: PRODUCTION
# Imagen final solo con JRE (no JDK) — mucho más liviana y segura
# JDK: ~300MB | JRE alpine: ~80MB
# ============================================================
FROM eclipse-temurin:17-jre-alpine AS production

# Instalar curl para healthcheck (y luego limpiar caché)
RUN apk add --no-cache curl

# Crear grupo y usuario sin privilegios (principio de mínimo privilegio)
# Si el proceso es comprometido, el atacante no tiene acceso root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copiar SOLO el JAR generado en stage anterior
COPY --from=builder /app/target/*.jar app.jar

# Asignar propietario al usuario sin privilegios
RUN chown appuser:appgroup app.jar

# Exponer el puerto de la API
EXPOSE 8080

# Healthcheck: verifica que el endpoint de Spring Boot responde
HEALTHCHECK --interval=30s --timeout=10s --start-period=45s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

# Ejecutar como usuario sin privilegios (NO root)
USER appuser

ENTRYPOINT ["java", "-jar", "app.jar"]
