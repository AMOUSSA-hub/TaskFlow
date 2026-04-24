# ── Stage 1 : Build ──────────────────────────────────────────────────
FROM node:20-alpine AS build

WORKDIR /app

# Copier package*.json AVANT le code → cache npm ci si seul le code change
COPY package*.json ./
RUN npm ci --omit=dev

# ── Stage 2 : Production ────────────────────────────────────────────
FROM node:20-alpine

# Labels OCI — traçabilité en production
LABEL org.opencontainers.image.title="taskflow-backend" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.authors="AMOUSSA-hub" \
      org.opencontainers.image.source="https://github.com/AMOUSSA-hub/TaskFlow" \
      org.opencontainers.image.description="TaskFlow API — Backend Node.js"

# Utilisateur non-root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copier les dépendances depuis le stage build
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/package.json ./package.json

# Copier le code source
COPY server.js ./

# Changer le propriétaire des fichiers
RUN chown -R appuser:appgroup /app

# Passer à l'utilisateur non-root
USER appuser

EXPOSE 3001

# Healthcheck — suit l'image partout où elle est déployée
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3001/health || exit 1

CMD ["node", "server.js"]
