# --- Etapa 1: Compilación de los archivos estáticos --- 
FROM node:20-alpine AS build 

WORKDIR /app 

COPY package.json package-lock.json ./ 

RUN npm ci 

COPY . . 

RUN npm run build 



# --- Etapa 2: Servidor web liviano y no raíz --- 
FROM nginxinc/nginx-unprivileged:1.27-alpine

ENV API_HOST=reservas-api \
    API_PORT=3000 \
    NGNIX_ENVSUBST_FILTER=^API_

WORKDIR /usr/share/nginx/html 

COPY --from=build /app/dist . 

COPY nginx/default.conf.template /etc/nginx/templates/default.conf.template

EXPOSE 8080 

CMD ["nginx", "-g", "daemon off;"]


# --- Etapa 2: Servidor web liviano y no raíz --- 

FROM nginxinc/nginx-unprivileged:1.27-alpine 

ENV API_HOST=reservas-api
ENV API_PORT=3000
ENV NGINX_ENVSUBST_FILTER=^API_ 

WORKDIR /usr/share/nginx/html 

COPY --from=build /app/dist . 

COPY nginx/templates/ /etc/nginx/templates 

EXPOSE 8080 

CMD ["nginx", "-g", "daemon off;"]
