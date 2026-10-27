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
| `reservas-api` | `ingenuo` | Monolítica (`node:20`) | 402 MB | 
| `reservas-api` | `v1` | Multi-etapa (`node:20-alpine` + `npm ci --omit=dev`) | 50 MB | 
| `reservas-frontend` | `v1` | Multi-etapa (`node:20-alpine` -&gt; `nginx-unprivileged`) | 21 MB |


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

El orden de arranque obligatorio para garantizar la disponibilidad del sistema es: **Red (`reservas-net`) ➔ Volumen (`reservas-db-data`) ➔ Base de datos (`reservas-db`) ➔ API (`reservas-api`) ➔ Frontend (`reservas-frontend`)** 

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
  * Nginx resuelve el nombre de host del *upstream* (`reservas-api`) **una única vez durante el arranque**; si el contenedor de la API aún no está creado y registrado en el DNS interno de la red, Nginx no reintenta ni arranca en estado degradado: aborta de forma inmediata (PID 1 finaliza) y el contenedor queda en estado **`Exited (1)`** con el error textual: 
`nginx: [emerg] host not found in upstream "reservas-api" in /etc/nginx/conf.d/default.conf:24`. 

--- 

### Limitación actual y resolución en la Semana 4

* **Limitación actual:** Toda esta secuencia de dependencias, tiempos de espera y parámetros reside actualmente **en la memoria del operador que tipea los comandos en la terminal**, y no en ningún archivo ejecutable ni declarativo. Si un operador invierte un paso, olvida un flag o no aguarda a que MySQL informe `port: 3306`, la aplicación falla.

* **Resolución (Semana 4):** Esta problemática se resolverá de forma automatizada mediante **Docker Compose**, donde la totalidad de la infraestructura (redes, volúmenes, variables de entorno y orden de inicio con `depends_on` y condiciones de salud `condition: service_healthy`) quedará formalizada en un único archivo declarativo `docker-compose.yml`.



## Trabajo Práctico N° 4 — Docker Compose y Seguridad Básica
### 1. Healthchecks sin dependencias añadidas 
* **Evitar herramientas externas:** No se debe instalar `curl` ni herramientas de diagnóstico dentro de las imágenes solo para evaluar el estado del servicio, ya que agregaría peso innecesario y ampliaría la superficie de ataque con paquetes ajenos a producción [1]. 
* **Implementación nativa:** 
    * **`reservas-api`:** Emplea el método `fetch()` incorporado de forma nativa en Node.js (disponible a partir de la versión 18) [1]. 
    * **`reservas-frontend`:** Utiliza `wget`, el cual ya viene provisto por BusyBox dentro de la imagen base Alpine [1].
---
### 2. Orquestación y orden de arranque (`depends_on`) 
El uso de `depends_on` con `condition: service_healthy` resuelve dos problemáticas diferentes según la capa [1]: 
* **`reservas-api` sobre `reservas-db`:** La API requiere que el motor MySQL esté listo para aceptar conexiones TCP, algo que la forma simple `depends_on: [reservas-db]` no garantiza (solo espera a que el contenedor se inicie, no a que complete su proceso interno) [1]. Además, la API cuenta con reintentos durante 60 segundos [1]. 
* **`reservas-frontend` sobre `reservas-api`:** Nginx resuelve el nombre del *upstream* (`reservas-api`) una sola vez al arrancar [2]. La condición `service_healthy` asegura que el proxy inicie con el backend completamente disponible, evitando errores `host not found in upstream` y respuestas HTTP 502 al inicio [1]. 

---
### 3. Ciclo de vida y persistencia de datos 
* **`docker compose down`:** Detiene y elimina contenedores y redes, pero **preserva intactos los volúmenes nombrados** y la persistencia de las reservas [3, 4]. 
* **`docker compose down -v`:** Elimina los contenedores, redes y **destruye de forma irreversible los volúmenes**, eliminando los datos almacenados en MySQL [3, 4]. 

---
### 4. Variables de entorno y gestión de credenciales 
* **Riesgo en producción:** Definir contraseñas literales en `docker-compose.yml` expone secretos a cualquier usuario con acceso al repositorio o al host mediante comandos como `docker inspect` [3, 5, 6].* **Solución aplicada:** Se desacoplaron las claves en un archivo `.env` excluido del control de versiones mediante `.gitignore`, documentando el contrato de variables en `.env.example` [3]. En entornos productivos de orquestación, esto se administra mediante objetos nativos como los *Secrets* de Kubernetes [3, 6]. 

--- 
### 5. Auditoría de seguridad 

#### A. Usuarios de ejecución y privilegios mínimos 
* **`reservas-api`:** Se ejecuta bajo el usuario sin privilegios `node` (`uid=1000`) [7]. 
* **`reservas-frontend`:** Corre bajo el usuario sin privilegios `nginx` (`uid=101`) [7]. 
* **`reservas-db`:** El demonio principal opera con el usuario `mysql`, sin acceso administrativo en el host [7, 8]. 

#### B. Exposición de puertos 
* **`reservas-db`:** No publica puertos al anfitrión; su comunicación permanece estrictamente aislada dentro de `reservas-net` [7, 9]. 
* **`reservas-api`:** Expone el puerto `3001` únicamente para fines de depuración directa durante el desarrollo (será puramente interno en Kubernetes) [7]. 
* **`reservas-frontend`:** Publica `3000:8080`, constituyendo el único punto de entrada público de la aplicación [7, 10]. 

#### C. Escaneo de vulnerabilidades (Trivy) 
| Imagen | Origen de los hallazgos | Total | Severidad (C / H / M / L) | Con versión corregida (*Fixed Version*) | | :--- | :--- | :---: | :---: | :---: | 
| **`reservas-api:v1`** | Sistema base (`Alpine 3.23.4`) | **51** | 0 CRIT / 4 HIGH / 15 MED / 32 LOW | Muy pocos parches disponibles en el SO | 
| **`reservas-api:v1`** | Dependencias de app (`node-pkg`) | **40** | 2 CRIT / 22 HIGH / 13 MED / 3 LOW | La gran mayoría cuenta con parche en npm | 
| **`reservas-frontend:v1`** | Sistema base (`Alpine 3.21.3`) | **159** | 2 CRIT / 39 HIGH / 76 MED / 42 LOW | Prácticamente todas tienen parche | 
| **`reservas-frontend:v1`** | Dependencias de app | **0** | 0 CRIT / 0 HIGH / 0 MED / 0 LOW | No aplica (construcción multi-etapa) | 

* **Conclusión de dependencias:** La construcción multi-etapa del frontend eliminó exitosamente Node, npm y `node_modules` de la imagen de producción, resultando en 0 vulnerabilidades de aplicación [11]. Los hallazgos de sistema corresponden a parches pendientes que se solucionan actualizando la imagen base de Alpine. 

--- 
### 6. Principio de mínimo privilegio con Linux Capabilities 
Al aplicar `cap_drop: [ALL]` a todos los servicios [12]: 
* **`reservas-api` y `reservas-frontend`:** Continúan operando con normalidad sin requerir ninguna capability adicional, ya que corren como usuarios no-root [8, 12]. 
* **`reservas-db`:** Requiere capabilities puntuales agregadas mediante `cap_add` [12]: 
* **`SETGID`:** Necesaria para cambiar el ID de grupo del proceso inicial a `mysql` [12, 13]. 
* **`SETUID`:** Necesaria para que el script de inicio descienda los privilegios de `root` al usuario `mysql` [8, 12]. 
* **`DAC_OVERRIDE`:** Necesaria para inspeccionar y preparar archivos en el directorio de datos ignorando temporalmente las restricciones POSIX de permisos [8, 12].

# Trabajo Práctico N°5 - Kubernetes

## Diagnóstico de Errores e Incidencias 

### 1. Error de descarga de imagen: `ImagePullBackOff` / `ErrImagePull` 

* **Comando ejecutado:** 
  ``bash 
  kubectl run test-pull --image=reservas-api:v1

NAME        READY   STATUS              RESTARTS    AGE 
test-pull   0/1     ImagePullBackOff        0       15s
Events: 
Type            Reason          Age                 From                 Message 
----            ------          ----                ----                 ------- 
Normal          Scheduled       30s                 default-scheduler    Successfully assigned reservas-app/test-pull to node-01 
Normal          Pulling         12s (x2 over 28s)   kubelet              Pulling image "reservas-api:v1" 
Warning         Failed          11s (x2 over 27s)   kubelet              Failed to pull image "reservas-api:v1": rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/reservas-api:v1": not found 
Warning         Failed          11s (x2 over 27s)   kubelet              Error: ErrImagePull 
Normal          BackOff         2s (x2 over 26s)    kubelet              Back-off pulling image "reservas-api:v1"

* **Causa Raíz:** Al especificar la imagen como `reservas-api:v1` sin indicar un prefijo de usuario, Kubernetes/cri-o intenta descargarla del registro público oficial (`docker.io/library/reservas-api:v1`), el cual no existe. Dado que el nodo del clúster no tiene la imagen construida localmente, la descarga falla.

* **Solución Aplicada:**
1. Generar un tag indicando el usuario de Docker Hub: docker tag reservas-api:v1 ccinquemani87/reservas-api:v1
2. Publicar la imagen en el registry público: docker push ccinquemani87/reservas-api:v1
3. Hacer referencia a la imagen completa en el manifiesto YAML image: ccinquemani87/reservas-api:v1


### 2. Fallo de resolución DNS y CrashLoopBackOff en la API 

* **Estado observado:** `reservas-api` cambia de estado a `Error` y luego a `CrashLoopBackOff` tras múltiples reinicios. 

* **Evidencia en los registros (`kubectl logs reservas-api`): ** Error de conexión a la base de datos por fallo de resolución de nombre de host (`getaddrinfo ENOTFOUND reservas-db`). 

* **Causa Raíz:** En Kubernetes, desplegar Pods independientes (*naked Pods*) no crea automáticamente entradas DNS en el clúster. La variable `DB_HOST=reservas-db` no puede ser resuelta por la API al no existir un objeto de tipo `Service` asociado a la base de datos. 

* **Conclusión:** Demuestra la imposibilidad de comunicación por nombre entre Pods aislados sin la abstracción de red que proveen los `Services` en Kubernetes.


### 3. Diagnóstico de Errores y Comportamientos Observados

#### A. Diagnóstico de `ImagePullBackOff` / `ErrImagePull`

Se provocó de manera deliberada el error al intentar desplegar un Pod haciendo referencia a una imagen sin prefijo de usuario del registry (`reservas-api:v1`):

```
kubectl run pod-test --image=reservas-api:v1

```

* **Comando de inspección:** `kubectl describe pod pod-test`

* **Causa raíz identificada en** **Events** **:** El `kubelet` intentó descargar la imagen desde Docker Hub oficial en lugar del repositorio personal. Al no existir la imagen en la ruta oficial, el clúster falló con `ErrImagePull` y pasó al estado de reintento `ImagePullBackOff`.


### B. Diagnóstico de `CrashLoopBackOff` en la API (`DB_HOST=reservas-db`)

Al aplicar los manifiestos con la variable `DB_HOST=reservas-db`, el Pod `reservas-api` falló repetidamente quedando en estado `CrashLoopBackOff`.

* **Salida de** **kubectl logs reservas-api** **:** `connect EHOSTUNREACH 10.244.0.10:3306` (o `getaddrinfo ENOTFOUND reservas-db`)

* **Causa raíz:** En Kubernetes, un Pod individual **no proporciona resolución DNS** por nombre a otros Pods. El nombre corto `reservas-db` no es resoluble dentro del clúster sin la presencia de un recurso tipo **Service**.

### C. Prueba con Dirección IP y Volatilidad de las IPs 

Para comprobar la conectividad entre componentes, se reemplazó temporalmente la variable `DB_HOST` en `pod-reservas-api.yaml` por la IP directa asignada al Pod de la base de datos (`10.244.0.21`). Al aplicar el cambio, el Pod `reservas-api` pasó correctamente a estado **`Running`**. 

* **Prueba de volatilidad:** Se eliminó y recreó el Pod de la base de datos (`kubectl delete pod reservas-db`), provocando que Kubernetes le asignara una nueva IP dinámica. Como consecuencia, la API volvió a fallar. 

* **Conclusión:** Las IPs de los Pods son efímeras y descartables, lo que hace imprescindible el uso de **Services** para disponer de un punto de acceso estable. 

--- 

### D. Diagnóstico en el Frontend (`host not found in upstream`) Al desplegar `pod-reservas-frontend.yaml`, el contenedor de Nginx falló indicando `host not found in upstream`. 

* **Causa raíz:** Nginx intentó resolver el nombre DNS del backend (`reservas-api`), pero al no existir un recurso tipo **Service** en el clúster, la resolución falló durante la inicialización del servidor web. 

--- 

### E. Acceso de Diagnóstico mediante `port-forward` Para verificar el funcionamiento de la API de forma aislada, se utilizó la redirección de puertos temporal: ```bash kubectl port-forward pod/reservas-api 3001:3000

**Comprobaciones efectuadas:**
* `curl -s localhost:3001/health` → Responde con el estado de salud de la aplicación.
* `curl -s localhost:3001/reservas` → Devuelve la respuesta de la base de datos.

### F. Falta de Supervisión ("Un Pod es mortal")

Al eliminar manualmente el Pod con `kubectl delete pod reservas-api`, el recurso fue destruido definitivamente sin que el clúster intentara recrearlo. Esto confirma que los Pods sueltos carecen de mecanismos de autorreparación y alta disponibilidad

## 4. Comparativa: `docker-compose.yml` vs. Manifiestos K8s (TP5)

| Aspecto               | Docker Compose (TP4)                                    | K8s - Pods Sueltos (TP5)                                 |
| --------------------- | ------------------------------------------------------- | -------------------------------------------------------- |
| **Agrupación**        | Definición centralizada en `docker-compose.yml`.        | Manifiestos YAML independientes por Pod.                 |
| **Resolución DNS**    | Automática por nombre de servicio en la red de Compose. | **Sin DNS.** Requiere recursos de tipo `Service`.        |
| **Control de Inicio** | `depends_on` con `condition: service_healthy`.          | Sin orden de arranque; la app debe gestionar reintentos. |
| **Resiliencia**       | Reinicio de contenedores según política.                | Si el Pod se elimina, no se vuelve a crear.              |


## 5. Cheatsheet de Comandos `kubectl` Utilizados

1. `kubectl config current-context`: Muestra el clúster y contexto activo.
2. `kubectl create namespace reservas-app`: Crea el namespace de trabajo.
3. `kubectl config set-context --current --namespace=reservas-app`: Fija el namespace por defecto.
4. `kubectl run reservas-db --image=mysql:8.4 --dry-run=client -o yaml &gt; k8s/pod-reservas-db.yaml`: Genera la plantilla YAML del Pod.
5. `kubectl apply -f k8s/`: Aplica declarativamente todos los manifiestos de la carpeta.
6. `kubectl get pods -o wide`: Lista los Pods mostrando IP asignada y nodo.
7. `kubectl describe pod



