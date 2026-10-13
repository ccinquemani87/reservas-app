# Etapa 1: dependencias de producción 

FROM node:20-alpine AS deps 
WORKDIR /app 
COPY package.json package-lock.json ./ 
RUN npm ci --omit=dev 


# Etapa 2: imagen final mínima 
FROM node:20-alpine WORKDIR /app 
ENV NODE\_ENV=production 
ARG APP\_VERSION=v1 
ENV APP\_VERSION=${APP\_VERSION} 

COPY --from=deps /app/node\_modules ./node\_modules 
COPY --chown=node:node . . 

USER node 
EXPOSE 3000 
CMD ["node", "src/index.js"]

