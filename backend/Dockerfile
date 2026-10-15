# Etapa 1: dependencias de producción 

FROM node:20-alpine AS deps 
WORKDIR /app 
COPY package.json package-lock.json ./ 
RUN npm ci --omit=dev 


# Etapa 2: imagen final mínima 
FROM node:20-alpine 
WORKDIR /app 
ENV NODE_ENV=production 
ARG APP_VERSION=v1 
ENV APP_VERSION=${APP_VERSION} 

COPY --from=deps /app/node_modules ./node_modules 
COPY --chown=node:node . . 

USER node 
EXPOSE 3000 
CMD ["node", "server.js"]

