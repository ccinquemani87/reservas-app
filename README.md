## Arquitectura
navegador ──► reservas-frontend ──────► reservas-api ──────► reservas-db
React + Vite Node 20 + Express MySQL 8.4
nginx :8080 :3000 :3306
(host 3000) (host 3001, solo (sin puerto
depuración) publicado)

| Capa | Imagen | Puerto interno | Puerto publicado |
| --- | --- | --- | --- |
| `reservas-frontend` | React + Vite servido por nginx | 8080 | 3000 |
| `reservas-api` | Node 20 + Express | 3000 | 3001 (solo depuración) |
| `reservas-db` | MySQL 8.4 | 3306 | — |
El frontend es el único punto de entrada: nadie le habla a la base
directamente, y a la API le habla el frontend.


## TP 2 - Optimización de Imágenes y Multi-stage Builds 

### Tabla Comparativa de Tamaños 

| Imagen | Tag | Estrategia / Base | Tamaño Final | 
| :--- | :--- | :--- | :--- | 
| \`reservas-api\` | \`ingenuo\` | Monolítica (\`node:20\`) | 402 MB | 
| \`reservas-api\` | \`v1\` | Multi-etapa (\`node:20-alpine\` + \`npm ci --omit=dev\`) | 50 MB | 
| \`reservas-frontend\` | \`v1\` | Multi-etapa (\`node:20-alpine\` -&gt; \`nginx-unprivileged\`) | 21 MB |
