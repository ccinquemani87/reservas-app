# Proyecto Integrador: `reservas-app`
**Curso:** Contenedores, Docker y Orquestación con Kubernetes  
**Variante B:** Reserva de salas por franja horaria (React + Vite, Node 20 + Express, MySQL 8.4)  

---

## 📌 Descripción del Proyecto
`reservas-app` es una aplicación de arquitectura distribuida en 3 capas diseñada para la gestión y reserva de salas por franja horaria:
* **Frontend:** Aplicación SPA en React + Vite servida por Nginx (`:8080` interno, `:3000` publicado en Compose).
* **Backend (API):** API REST desarrollada en Node.js + Express (`:3000` interno).
* **Base de Datos:** Motor relacional MySQL 8.4 (`:3306` interno).

---

## 📁 Estructura del Repositorio
```text
reservas-app/
├── backend/                # Código fuente de la API Express
├── frontend/               # Código fuente de la SPA React
├── k8s/                    # Manifiestos estáticos de los 12 objetos de Kubernetes
├── reservas-chart/         # Chart de Helm con los 12 objetos parametrizados
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
├── docker-compose.yaml     # Orquestación para entorno local/desarrollo
├── ARQUITECTURA.md         # Documento con las respuestas teóricas y justificaciones técnicas
├── INFORME.md              # Evidencia de despliegue, pruebas de HPA y ciclo de vida de Helm
└── README.md               # Guía rápida de inicio y despliegue
```

---

## 🚀 Cómo levantar el proyecto

### **Camino 1: Con Docker Compose (Entorno Local en una sola máquina)**
Para desplegar la aplicación completa localmente con Compose:

1. **Construir y levantar los 3 servicios:**
   ```bash
   docker compose up --build -d
   ```

2. **Verificar el estado de las comprobaciones de salud (Healthchecks):**
   ```bash
   docker compose ps
   ```

3. **Acceder a la aplicación:**
   * **Navegador Web:** `http://localhost:3000`
   * **API REST (solo depuración):** `http://localhost:3001/api/reservas`

4. **Detener la aplicación:**
   * Conservando volumen de datos: `docker compose down`
   * Eliminando volumen de datos: `docker compose down -v`

---

### **Camino 2: Con Helm (Sobre el clúster de Kubernetes)**
Para desplegar el Chart de Helm empaquetado en un clúster de Kubernetes:

1. **Simular el renderizado de los 12 objetos para validar sintaxis:**
   ```bash
   helm template reservas-app ./reservas-chart
   ```

2. **Instalar el Chart de Helm en el namespace `reservas-app`:**
   ```bash
   helm install reservas-app ./reservas-chart -n reservas-app --create-namespace
   ```

3. **Verificar el despliegue de los 12 objetos en el clúster:**
   ```bash
   kubectl get all,cm,secret,pvc,ingress,hpa -n reservas-app
   ```

4. **Acceder a la aplicación:**
   * Configurar `/etc/hosts`: `<IP-del-nodo> reservas.local`
   * Abrir en el navegador: `http://reservas.local`

5. **Actualizar la versión de la API (Upgrade):**
   ```bash
   helm upgrade reservas-app ./reservas-chart --set api.image.tag=v2 -n reservas-app
   ```

6. **Consultar historial y hacer Rollback:**
   ```bash
   helm history reservas-app -n reservas-app
   helm rollback reservas-app 1 -n reservas-app
   ```

---

## 📑 Documentación Adicional
* 📘 **[ARQUITECTURA.md](./ARQUITECTURA.md):** Respuestas a las consignas de arquitectura, seguridad, CORS, persistencia y concurrencia.
* 📗 **[INFORME.md](./INFORME.md):** Demostración completa del funcionamiento en el clúster, prueba bajo carga de HPA y evidencias de comandos.
