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


# Trabajo Práctico N° 3 — Redes y Almacenamiento en Docker

Se ejecutó el frontend omitiendo el parámetro `--network`:

```
docker run -d --name reservas-frontend -p 3000:8080 -e API_HOST=reservas-api -e API_PORT=3000 reservas-frontend:v1

```

* Salida de `docker ps -a`: Contenedor en estado `Exited (1)`.
* Salida de `docker logs reservas-frontend`:

```
nginx: [emerg] host not found in upstream "reservas-api" in /etc/nginx/conf.d/default.conf:24

```

### Explicación del fallo:

Al omitir `--network`, el contenedor se conectó a la red `bridge` por defecto, la cual no cuenta con resolución DNS interna por nombre de contenedor. Nginx resuelve las direcciones de sus directivas `proxy_pass` una sola vez durante el arranque; al no poder resolver el nombre `reservas-api`, aborta el proceso de inmediato en lugar de iniciar de forma degradada.

## 8. Orden estricto de arranque, análisis de fallos por inversión y limitaciones 

El orden de arranque obligatorio para garantizar la disponibilidad del sistema es: **Red (`reservas-net`) ➔ Volumen (\`reservas-db-data\`) ➔ Base de datos (\`reservas-db\`) ➔ API (\`reservas-api`) ➔ Frontend (`reservas-frontend`)** 

--- 

### ¿Qué falla exactamente al invertir cada paso? 

1. **Invertir Red ➔ Base de datos (o cualquier contenedor):** 
  * **Fallo:** Si se intenta crear un contenedor asignándole `--network reservas-net` antes de haber creado la red, el comando aborta inmediatamente con: `docker: Error response from daemon: network reservas-net not found`. 
  * Si en su lugar se omite el parámetro `--network` para arrancarlo primero, el contenedor cae en la red predeterminada (`bridge`), donde no existe resolución DNS por nombre de contenedor, impidiendo que el resto del sistema pueda localizarlo. 

2. **Invertir Volumen ➔ Base de datos:** 
  * **Fallo:** Si se inicia la base de datos omitiendo el montaje del volumen persistente (`-v reservas-db-data:/var/lib/mysql`), MySQL escribirá sus tablas directamente en la capa de lectura/escritura efímera del contenedor. 
  * Al eliminar el contenedor con `docker rm -f reservas-db`, se destruyen todas las tablas y datos creados, perdiendo la persistencia del sistema. 

3. **Invertir Base de datos ➔ API:**

  * **Fallo:** Al arrancar con la variable `SEED_DEMO=true`, la API intenta conectarse de inmediato con el motor MySQL para ejecutar las migraciones y sembrar los datos. 
  * Si la base no existe o aún se encuentra en su fase preliminar de inicialización (servidor temporal en `port: 0`), la API no logra conectarse y registra un error crítico **`ECONNREFUSED`** en sus logs, interrumpiendo o degradando el servicio. 

4. **Invertir API ➔ Frontend:**
  * **Fallo:** Nginx procesa la plantilla y sustituye la directiva `proxy_pass http://${API_HOST}:${API_PORT}/;` al momento de inicializar el contenedor. 
  * Nginx resuelve el nombre de host del \*upstream\* (\`reservas-api\`) **una única vez durante el arranque**; si el contenedor de la API aún no está creado y registrado en el DNS interno de la red, Nginx no reintenta ni arranca en estado degradado: aborta de forma inmediata (PID 1 finaliza) y el contenedor queda en estado **`Exited (1)`** con el error textual: 
`nginx: [emerg] host not found in upstream "reservas-api" in /etc/nginx/conf.d/default.conf:24`. 

--- 

### Limitación actual y resolución en la Semana 4

* **Limitación actual:** Toda esta secuencia de dependencias, tiempos de espera y parámetros reside actualmente **en la memoria del operador que tipea los comandos en la terminal**, y no en ningún archivo ejecutable ni declarativo. Si un operador invierte un paso, olvida un flag o no aguarda a que MySQL informe `port: 3306`, la aplicación falla.

* **Resolución (Semana 4):** Esta problemática se resolverá de forma automatizada mediante **Docker Compose**, donde la totalidad de la infraestructura (redes, volúmenes, variables de entorno y orden de inicio con `depends_on` y condiciones de salud `condition: service_healthy`) quedará formalizada en un único archivo declarativo `docker-compose.yml`.
