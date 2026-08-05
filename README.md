# Demo de Métricas DORA con Apache DevLake

## Descripción General

Este repositorio contiene un ambiente de demostración para recolectar y visualizar **métricas DORA** utilizando:

* GitHub Actions
* Apache DevLake
* MySQL
* Grafana
* Kubernetes mediante Helm

El objetivo es generar automáticamente eventos del ciclo de desarrollo de software:

* Commits
* Pull Requests
* Deployments
* Fallos de despliegue
* Recuperaciones después de fallos

y utilizar Apache DevLake para calcular las métricas DORA.

Los resultados son visualizados mediante dashboards de Grafana.

---

# Arquitectura

```text
GitHub Repository
        |
        |
        v
GitHub Actions
        |
        |
        +----------------+
        |                |
        v                v
 Commits             Deployments
 Pull Requests       DEV/STAGING/PRODUCTION
        |
        |
        v
Apache DevLake
        |
        |
        +----------------+
        |
        v
MySQL Database
        |
        |
        v
Grafana DORA Dashboard
```

---

# Componentes

| Componente     | Propósito                                |
| -------------- | ---------------------------------------- |
| GitHub Actions | Genera eventos del ciclo de entrega      |
| Apache DevLake | Recolecta, transforma y calcula métricas |
| MySQL          | Base de datos de DevLake                 |
| Grafana        | Visualización de métricas DORA           |
| Kubernetes     | Plataforma donde se ejecuta DevLake      |

---

# Estructura del Repositorio

```text
.
├── .github
│   └── workflows
│       ├── dora-demo-generator.yml
│       └── deploy-demo.yml
│
├── README.md
└── dora-demo.txt
```

---

# Requisitos Previos

Instalar:

* kubectl
* Helm
* GitHub CLI (`gh`)

Validar instalación:

```bash
kubectl version

helm version

gh version
```

---

# Instalación de Apache DevLake

## Crear namespace

```bash
kubectl create namespace devlake
```

---

## Agregar repositorio Helm

```bash
helm repo add devlake https://apache.github.io/incubator-devlake-helm-chart

helm repo update
```

---

## Instalar DevLake

```bash
helm install devlake devlake/devlake \
  -n devlake
```

---

## Validar instalación

```bash
kubectl get pods -n devlake
```

Debe mostrar:

```text
devlake-grafana
devlake-lake
devlake-ui
devlake-mysql
```

---

# Acceso a DevLake UI

Obtener el servicio:

```bash
kubectl get svc -n devlake
```

Ejemplo:

```text
devlake-ui   NodePort   4000:32001
```

Ingresar:

```text
http://<IP-NODE>:32001
```

---

# Acceso a Grafana

Crear port-forward:

```bash
kubectl port-forward \
svc/devlake-grafana \
-n devlake 3000:80
```

Abrir:

```text
http://localhost:3000
```

---

# Configuración de GitHub

Apache DevLake recolecta:

* Commits
* Pull Requests
* GitHub Actions Runs
* Deployments

Crear un token de GitHub con permisos:

```text
Repository:
- Contents Read/Write
- Pull Requests Read/Write
- Actions Read/Write
```

Configurar la conexión desde DevLake UI.

---

# Generador Automático de Datos DORA

Este repositorio incluye workflows que generan datos automáticamente para alimentar los dashboards DORA.

Workflows:

```text
.github/workflows/

├── dora-demo-generator.yml
└── deploy-demo.yml
```

---

# Flujo generado

El workflow crea automáticamente:

1. Rama de funcionalidad
2. Commit
3. Pull Request
4. Merge del Pull Request
5. Deployment DEV
6. Deployment STAGING
7. Deployment PRODUCTION exitoso
8. Deployment PRODUCTION fallido
9. Deployment de recuperación

Flujo:

```text
Feature Branch
      |
      v
Commit
      |
      v
Pull Request
      |
      v
Merge
      |
      v
DEV
      |
      v
STAGING
      |
      v
PRODUCTION SUCCESS
      |
      v
PRODUCTION FAILURE
      |
      v
RECOVERY
```

---

# Configuración del Secret de GitHub

Crear un secret:

```text
Repository
 |
 Settings
 |
 Secrets and variables
 |
 Actions
 |
 New repository secret
```

Nombre:

```text
DORA_DEMO_TOKEN
```

Permisos requeridos:

```text
Contents       Read/Write
Pull Requests  Read/Write
Actions        Read/Write
```

---

# Ejecutar Generador DORA

Ir a:

```text
GitHub
 |
Actions
 |
DORA Demo Generator
 |
Run workflow
```

Seleccionar:

```text
iterations: 10
```

Ejemplo:

```text
iterations = 10
```

Esto genera aproximadamente:

| Evento               | Cantidad |
| -------------------- | -------: |
| Pull Requests        |       10 |
| Deployments DEV      |       10 |
| Deployments STAGING  |       10 |
| Production Success   |       10 |
| Production Failure   |       10 |
| Recovery Deployments |       10 |

---

# Convención de Ambientes

Para compatibilidad con los dashboards de DevLake utilizar:

```text
DEV
STAGING
PRODUCTION
```

Evitar:

```text
dev
staging
production
```

porque algunas consultas SQL de los dashboards distinguen mayúsculas y minúsculas.

---

# Recolección de Datos en DevLake

Después de generar actividad en GitHub:

Ingresar a DevLake:

```text
Projects
   |
   Proyecto DORA
   |
   Collect Data
```

Esperar a que termine el pipeline.

---

# Validación en MySQL

Entrar al contenedor:

```bash
kubectl exec -it \
-n devlake \
statefulset/devlake-mysql \
-- bash
```

Conectar:

```bash
mysql -uroot -padmin lake
```

---

## Validar Deployments

```sql
SELECT
 environment,
 result,
 COUNT(*) AS total
FROM cicd_deployment_commits
GROUP BY environment,result;
```

Resultado esperado:

```text
DEV          SUCCESS
STAGING      SUCCESS
PRODUCTION   SUCCESS
PRODUCTION   FAILURE
```

---

## Validar Pull Requests

```sql
SELECT
 project_name,
 COUNT(*) AS total_prs
FROM project_pr_metrics
GROUP BY project_name;
```

---

## Validar Lead Time

```sql
SELECT
 project_name,
 pr_cycle_time,
 pr_deployed_date
FROM project_pr_metrics;
```

Debe existir:

```text
pr_deployed_date
```

para que DevLake pueda calcular Lead Time for Changes.

---

# Métricas DORA

## Deployment Frequency

Mide:

```text
Frecuencia de despliegues a producción
```

Fuente:

```text
cicd_deployment_commits
```

---

## Change Failure Rate

Mide:

```text
Deployments fallidos /
Total deployments productivos
```

Ejemplo:

```text
3 fallos / 10 deployments
= 30%
```

Fuente:

```text
cicd_deployment_commits
```

---

## Lead Time for Changes

Mide:

```text
Tiempo desde el cambio de código
hasta producción
```

Flujo:

```text
Commit
  |
  v
Pull Request
  |
  v
Merge
  |
  v
Deployment
```

Fuente:

```text
project_pr_metrics
```

---

## Mean Time To Recovery (MTTR)

Mide:

```text
Tiempo desde un fallo
hasta la recuperación
```

Flujo:

```text
Production Failure
        |
        v
Recovery Deployment
```

---

# Solución de Problemas

## Grafana sin datos

Validar tablas:

```sql
SELECT COUNT(*) FROM commits;

SELECT COUNT(*) FROM cicd_tasks;

SELECT COUNT(*) FROM cicd_deployments;
```

Ejecutar nuevamente la recolección de DevLake.

---

## Deployment Frequency sin datos

Validar ambientes:

```sql
SELECT DISTINCT environment,result
FROM cicd_deployment_commits;
```

Debe existir:

```text
PRODUCTION SUCCESS
```

---

## Datos históricos con nombres diferentes

Si existen:

```text
production
dev
```

y también:

```text
PRODUCTION
DEV
```

los datos antiguos pueden permanecer.

Los nuevos eventos deben utilizar:

```text
DEV
STAGING
PRODUCTION
```

---

# Versiones Utilizadas

| Componente     | Versión             |
| -------------- | ------------------- |
| Apache DevLake | v1.0.2              |
| Helm Chart     | devlake-1.0.2       |
| Grafana        | Incluido en DevLake |
| Kubernetes     | Instalación Helm    |

---

# Mejoras Futuras

Posibles extensiones:

* Integración con Jira
* Integración con SonarQube
* Métricas DevSecOps
* Dashboards por equipo
* Generación automática de cientos de eventos DORA
* Integración con múltiples repositorios

---

# Objetivo del Proyecto

Este ambiente sirve como plataforma reutilizable para demostrar:

* Adopción de métricas DORA
* Madurez DevOps
* Optimización de CI/CD
* Ingeniería basada en datos
* Mejora continua del ciclo de entrega
