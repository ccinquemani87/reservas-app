# --- Etapa 1: Compilación de los archivos estáticos --- 
FROM node:20-alpine AS build 

WORKDIR /app 

COPY package.json package-lock.json ./ 

RUN npm ci 

COPY . . 

RUN npm run build 



# --- Etapa 2: Servidor web liviano y no raíz --- 
FROM nginxinc/nginx-unprivileged:1.27-alpine 

WORKDIR /usr/share/nginx/html 

COPY --from=build /app/dist . 

EXPOSE 8080 

CMD ["nginx", "-g", "daemon off;"]
