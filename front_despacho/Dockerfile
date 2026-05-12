# ============================================================
# Stage 1: BUILD
# Compila la aplicación React con Vite
# ============================================================
FROM node:20-alpine AS builder

WORKDIR /app

# Copiar package.json primero para cachear instalación de dependencias
# Si package.json no cambia, Docker reutiliza esta capa
COPY package*.json ./
RUN npm ci

# Copiar el resto del código y construir para producción
COPY . .
RUN npm run build

# ============================================================
# Stage 2: PRODUCTION
# Imagen final liviana con Nginx para servir archivos estáticos
# Node completo: ~350MB | nginx:alpine: ~25MB
# ============================================================
FROM nginx:1.25-alpine AS production

# Instalar curl para healthcheck
RUN apk add --no-cache curl

# Eliminar configuración por defecto de Nginx
RUN rm /etc/nginx/conf.d/default.conf

# Copiar configuración personalizada de Nginx
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copiar el build generado en Stage 1
COPY --from=builder /app/dist /usr/share/nginx/html

# Crear usuario sin privilegios y asignar permisos
RUN addgroup -S appgroup && adduser -S appuser -G appgroup \
    && chown -R appuser:appgroup /usr/share/nginx/html \
    && chown -R appuser:appgroup /var/cache/nginx \
    && chown -R appuser:appgroup /var/log/nginx \
    && touch /var/run/nginx.pid \
    && chown appuser:appgroup /var/run/nginx.pid

EXPOSE 80

# Healthcheck
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD curl -f http://localhost/ || exit 1

# Ejecutar como usuario sin privilegios
USER appuser

CMD ["nginx", "-g", "daemon off;"]
