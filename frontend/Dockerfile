FROM nginx:alpine

LABEL org.opencontainers.image.title="taskflow-frontend" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.authors="AMOUSSA-hub" \
      org.opencontainers.image.source="https://github.com/AMOUSSA-hub/TaskFlow"

# Copier la configuration Nginx
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copier les fichiers statiques
COPY index.html /usr/share/nginx/html/
COPY app.js     /usr/share/nginx/html/
COPY styles.css /usr/share/nginx/html/

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost/nginx-health || exit 1

CMD ["nginx", "-g", "daemon off;"]