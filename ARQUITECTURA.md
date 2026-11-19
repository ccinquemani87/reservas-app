# Documento de Arquitectura — Trabajo Final Integrador
**Asignatura:** Contenedores, Docker y Orquestación con Kubernetes  
**Proyecto Integrador (Variante B):** `reservas-app` (React + Vite, Node 20 + Express, MySQL 8.4)  

---

## 1. Preguntas Obligatorias del Curso

### 1.1. Configuración del Frontend: Tiempo de Ejecución vs. Tiempo de Construcción
**¿Por qué el frontend se configura con `API_HOST` en tiempo de ejecución y no con `VITE_API_URL` en tiempo de construcción? Si fuera al revés, ¿qué habría que rehacer para pasar de desarrollo a producción?**

* **Respuesta:**
  Las aplicaciones cliente (Single Page Applications - SPA) basadas en React + Vite se ejecutan del lado del navegador del usuario final. Si la dirección del backend se inyectara en tiempo de construcción mediante variables como `VITE_API_URL`, la URL quedaría incrustada estáticamente dentro de los archivos JavaScript minificados durante el comando `vite build`. 
  
  Si se configurara en tiempo de construcción, para promocionar el artefacto desde un entorno de desarrollo a un entorno de homologación o producción, **se violaría el principio de inmutabilidad del artefacto**: sería necesario volver a ejecutar la etapa de compilación (`build`) para generar una nueva imagen de Docker por cada entorno objetivo.

  Al externalizar la configuración a tiempo de ejecución mediante la variable `API_HOST` (leída dinámicamente por la plantilla del servidor Nginx que sirve la SPA mediante `envsubst` o scripts de arranque), se genera una **única imagen inmutable** (`reservas-frontend:v1`). La misma imagen se despliega en cualquier entorno simplemente cambiando la variable leída a través de un `ConfigMap` de Kubernetes o un archivo `.env` de Docker Compose.

---

### 1.2. Política de Origen Cruzado (CORS) y Proxies Inverso
**¿Por qué esta arquitectura no necesita CORS? ¿Qué habría que agregar si la aplicación de página única se comunicara directamente con la API?**

* **Respuesta:**
  Esta arquitectura no requiere configurar políticas de CORS (*Cross-Origin Resource Sharing*) en la API REST porque el navegador del cliente interactúa con un **único punto de origen** expuesto mediante el Ingress Controller (en Kubernetes) o Nginx (en Docker Compose). 

  Tanto las peticiones del frontend (HTML/JS/CSS) como las llamadas a la API REST (`/api/*`) viajan hacia el mismo nombre de dominio y puerto (`http://reservas.local`). El servidor Nginx/Ingress actúa como un *Reverse Proxy*, enrutando internamente el tráfico `/api` hacia el servicio `reservas-api`. Desde la perspectiva de la política de mismo origen del navegador (*Same-Origin Policy*), el esquema, host y puerto son idénticos.

  Si la SPA se comunicara directamente con el contenedor/Pod de la API sin pasar por el proxy (por ejemplo, si el cliente llamara a `http://api.reservas.local:3000` desde `http://frontend.reservas.local`), habría que agregar los encabezados de CORS en Express (`Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`) y gestionar las peticiones pre-flight (`OPTIONS`) enviadas por el navegador antes de cada consulta HTTP.

---

### 1.3. Criterio de Autoescalado y Selección de HPA
**¿Por qué hay un solo autoescalador (HPA) entre los doce objetos? ¿Qué ocurriría si se le asignara uno a `reservas-db`? ¿Y por qué tampoco lo tiene el frontend?**

* **Respuesta:**
  El escalado horizontal (*Horizontal Pod Autoscaler*) es una decisión de diseño ligada a la naturaleza de cada capa:
  
  1. **API Express (`reservas-api`):** Es una capa totalmente **sin estado** (*stateless*). Su función es puramente computacional (validar datos, procesar lógica de negocios, ejecutar consultas a la base de datos). Puede multiplicarse dinámicamente de 2 a 6 réplicas ante incrementos de carga de CPU sin riesgo de desincronización o corrupción de datos.
  2. **Base de Datos (`reservas-db`):** Es una capa **con estado** (*stateful*). Al utilizar un motor relacional como MySQL montado sobre un volumen persistente (`PersistentVolumeClaim` con modo de acceso `ReadWriteOnce`), no se puede escalar horizontalmente simplemente aumentando la cantidad de réplicas en un `Deployment`. Si el HPA creara múltiples Pods de MySQL compitiendo por escribir en la misma carpeta del PVC, se provocaría corrupción de datos o fallos de bloqueo en el sistema de archivos. Para escalar una base de datos se requiere una topología con estado (*StatefulSet*) utilizando replicación Maestro-Esclavo / *ReplicaSets*.
  3. **Frontend (`reservas-frontend`):** Es un servidor de archivos estáticos (Nginx sirviendo HTML, JS y CSS compilados). Su consumo de recursos computacionales (CPU/Memoria) es despreciable. Con una cantidad fija de 2 réplicas puede atender miles de solicitudes concurrentes por segundo sin resentir su rendimiento.

---

## 2. Pregunta Específica de la Variante B (`reservas-app`)

### 2.1. Concurrencia de Réplicas y Garantías de Integridad en Reservas
**¿Qué garantiza que una sala no se reserve dos veces cuando la API Express corre en tres réplicas concurrentes?**

* **Respuesta:**
  Cuando la API `reservas-api` corre escalada en múltiples réplicas (por ejemplo, 3 réplicas), las peticiones HTTP entrantes se distribuyen aleatoriamente entre los Pods a través del Service de Kubernetes. 

  Si dos usuarios intentan reservar exactamente la misma sala en la misma fecha y franja horaria simultáneamente, la validación previa realizada por el código Node.js (`SELECT * FROM reservas WHERE sala_id = X AND fecha = Y AND franja = Z`) no es suficiente para evitar condiciones de carrera (*Race Conditions*), ya que dos procesos independientes de Express pueden leer "disponible" exactamente en el mismo instante de tiempo.

  Lo que **garantiza empíricamente la invariante** de que no exista una doble reserva es la restricción de unicidad nivel motor de base de datos MySQL:
  ```sql
  UNIQUE KEY unique_reserva (sala_id, fecha, franja)
  ```
  Al intentar insertar la segunda reserva simultánea, el motor MySQL rechaza la operación a nivel transaccional lanzando un error de clave duplicada (código de error `1062` / `ER_DUP_ENTRY`). La API captura esta excepción del driver de MySQL y responde al cliente con un código de estado HTTP `409 Conflict`.

---

## 3. Dificultades Técnicas Presentadas durante el Proyecto

1. **Requisito de `resources.requests` para el HPA:**
   * **Síntoma:** Al crear el recurso `HorizontalPodAutoscaler`, la columna `TARGETS` en `kubectl get hpa` se mantenía en estado `<unknown>`.
   * **Diagnóstico:** El `metrics-server` requiere una referencia base asignada al contenedor para poder calcular el porcentaje de utilización de CPU.
   * **Solución:** Se declaró formalmente el bloque `resources.requests.cpu: 100m` en el contenedor del Deployment `reservas-api.yaml`.

2. **Orden de Arranque y Fallos de Resolución DNS (`host not found in upstream`):**
   * **Síntoma:** Al arrancar el Pod de Nginx (`reservas-frontend`), el contenedor fallaba inmediatamente y entraba en `CrashLoopBackOff`.
   * **Diagnóstico:** Nginx valida los nombres de host del bloque `upstream` (`reservas-api`) al momento del inicio. Si el Service de Kubernetes no existe previamente en el servidor DNS interno de CoreDNS, la verificación de sintaxis de Nginx falla.
   * **Solución:** Se estableció la buena práctica de aplicar siempre los objetos de red (`Services`) antes de aplicar los Deployments que dependen de ellos. En Helm, este orden es gestionado automáticamente por el motor de plantillas según el `Kind` del objeto.

---

## 4. Pendientes y Consideraciones para Entornos de Producción Real

Para llevar la infraestructura de `reservas-app` a un entorno de producción de misión crítica, quedarían pendientes las siguientes mejoras operativas:

1. **Gestión de Bases de Datos con Estado:** Sustituir el `Deployment` monolítico de MySQL por un `StatefulSet` utilizando un operador oficial de MySQL (como *MySQL InnoDB Cluster* o *Bitnami MySQL Operator*) que gestione réplicas de lectura y conmutación por error (*Failover*) automática.
2. **Gestión Externa de Secretos:** Implementar soluciones como HashiCorp Vault, AWS Secrets Manager o *Sealed Secrets* para evitar almacenar cadenas codificadas en Base64 en el repositorio Git.
3. **Pila de Observabilidad Centralizada:** Desplegar *Prometheus* y *Grafana* para la recolección de métricas de rendimiento y la pila *LOKI* o *EFK* para el procesamiento distribuido de logs.
4. **Estrategia de Backup y Desastre (DR):** Automatizar respaldos periódicos del volumen persistente (`PVC`) mediante herramientas de instantáneas (*Snapshots*) nativas como *Velero*.
