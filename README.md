# 🚀 devops-pipeline-github-actions-grafana

![Deploy](https://github.com/JoshuaSMC/devops-pipeline-github-actions-grafana/actions/workflows/deploy-render.yml/badge.svg)
![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-CI%2FCD-2088FF?logo=githubactions)
![Prometheus](https://img.shields.io/badge/Prometheus-monitoring-E6522C?logo=prometheus)
![Grafana](https://img.shields.io/badge/Grafana-dashboards-F46800?logo=grafana)
![Render](https://img.shields.io/badge/Deploy-Render-46E3B7?logo=render)
![AWS](https://img.shields.io/badge/AWS-ECS%20Fargate-FF9900?logo=amazonaws)

Pipeline CI/CD completo con **GitHub Actions** que despliega automáticamente en **AWS ECS** y **Render**, más stack de monitoreo con **Prometheus + Grafana**.

> **Repo 3 de 3 — Portfolio DevOps/Cloud**
> Este es el repo estrella de la trilogía. La app que se despliega está en [`tasks-api-spring-boot-docker`](https://github.com/JoshuaSMC/tasks-api-spring-boot-docker). La infraestructura donde corre se provisiona desde [`infrastructure-as-code-terraform-aws`](https://github.com/JoshuaSMC/infrastructure-as-code-terraform-aws).

> 🟢 **Live:** https://tasks-api-f4b9.onrender.com/api/tasks

---

## ⚙️ Tecnologías

| Capa | Tecnología |
|------|-----------|
| CI/CD | GitHub Actions |
| Registry origen | GHCR (GitHub Container Registry) |
| Registry destino | ECR (Amazon Elastic Container Registry) |
| Deploy | AWS ECS Fargate |
| Monitoreo | Prometheus 2.51 + Grafana 10.4 |
| Métricas app | Spring Boot Actuator + Micrometer |

---

## 🗺️ Narrativa del portfolio

| Repo | Qué muestra |
|------|------------|
| [1. tasks-api-spring-boot-docker](https://github.com/JoshuaSMC/tasks-api-spring-boot-docker) | App dockerizada, publicada en GHCR con CI/CD |
| [2. infrastructure-as-code-terraform-aws](https://github.com/JoshuaSMC/infrastructure-as-code-terraform-aws) | Infraestructura como código con Terraform y CloudFormation |
| **3. devops-pipeline-github-actions-grafana** ← estás acá | Pipeline CI/CD completo: deploy automático + monitoreo con Grafana |

---

## 🏗️ Arquitectura del pipeline

```
  Repo 1: tasks-api                     Repo 3: este repo
  ┌─────────────────┐                  ┌──────────────────────────────────────┐
  │  push → main    │                  │  GitHub Actions                      │
  │                 │                  │                                      │
  │  CI pipeline:   │   imagen en      │  ci-cd.yml                           │
  │  build → test   │──── GHCR ───────▶│  1. Pull imagen de GHCR              │
  │  → push GHCR    │                  │  2. Push a ECR                       │
  └─────────────────┘                  │  3. Actualizar task definition        │
                                       │  4. Deploy ECS (wait for stability)  │
  Repo 2: infrastructure               │  5. Smoke test post-deploy            │
  ┌─────────────────┐                  └──────────────────┬───────────────────┘
  │  terraform apply│                                     │
  │                 │   infra lista                       │ deploy
  │  VPC + ECS      │─────────────────────────────────────▼
  │  + ALB + ECR    │                  ┌──────────────────────────────────────┐
  └─────────────────┘                  │  AWS ECS Fargate                     │
                                       │  tasks-api :8080                     │
                                       │  (detrás del ALB)                    │
                                       └──────────────────┬───────────────────┘
                                                          │ métricas
                                                          ▼
                                       ┌──────────────────────────────────────┐
                                       │  Monitoreo local (docker compose)    │
                                       │                                      │
                                       │  Prometheus → scrape /actuator/      │
                                       │               prometheus             │
                                       │  Grafana    → dashboards en :3000    │
                                       └──────────────────────────────────────┘
```

---

## 📁 Estructura del proyecto

```
.
├── .github/
│   └── workflows/
│       ├── ci-cd.yml              # Pipeline principal: build → push ECR → deploy ECS
│       └── reusable-deploy.yml    # Workflow reutilizable para deploy desde otros repos
├── monitoring/
│   ├── prometheus.yml             # Config Prometheus: scrape tasks-api cada 15s
│   └── grafana/
│       ├── provisioning/
│       │   ├── datasources/       # Auto-configura Prometheus como datasource
│       │   └── dashboards/        # Auto-carga los dashboards al iniciar
│       └── dashboards/
│           └── tasks-api.json     # Dashboard con HTTP rate, latencia, JVM, CPU
└── docker-compose.monitoring.yml  # Stack completo: app + Prometheus + Grafana
```

---

## 🔄 Pipeline CI/CD

### Flujo completo

```
push → main
    │
    ▼
Job 1: build-and-push
    ├── Configura credenciales AWS
    ├── Login a ECR
    ├── Login a GHCR
    ├── Pull imagen ghcr.io/joshuasmc/tasks-api:latest
    └── Push a ECR (tag :latest y :sha-<commit>)
    │
    ▼
Job 2: deploy (environment: production)
    ├── Descarga task definition actual de ECS
    ├── Reemplaza la imagen con la nueva versión
    ├── Registra nueva task definition
    ├── Actualiza el ECS service (wait-for-stability: true)
    └── Smoke test: curl /actuator/health con retry
```

### Secrets requeridos

| Secret | Descripción |
|--------|------------|
| `AWS_ACCESS_KEY_ID` | Credencial IAM con permisos ECR + ECS |
| `AWS_SECRET_ACCESS_KEY` | Secret key correspondiente |

### Workflow reutilizable

`reusable-deploy.yml` puede ser llamado desde cualquier repo del portfolio:

```yaml
jobs:
  deploy:
    uses: JoshuaSMC/devops-pipeline-github-actions-grafana/.github/workflows/reusable-deploy.yml@main
    with:
      image: "123456789.dkr.ecr.us-east-1.amazonaws.com/tasks-api-prod:latest"
      ecs_cluster: "tasks-api-prod-cluster"
      ecs_service: "tasks-api-prod-service"
      container_name: "tasks-api"
    secrets:
      aws_access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws_secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

---

## 📊 Monitoreo local

### Prerrequisito: habilitar métricas Prometheus en tasks-api

Agregar al `pom.xml` de [`tasks-api-spring-boot-docker`](https://github.com/JoshuaSMC/tasks-api-spring-boot-docker):

```xml
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Y en `application.properties`:

```properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus
```

### Levantar el stack

```bash
# Levanta tasks-api + Prometheus + Grafana
docker compose -f docker-compose.monitoring.yml up
```

| Servicio | URL |
|---------|-----|
| tasks-api | http://localhost:8080 |
| Prometheus | http://localhost:9090 |
| Grafana | http://localhost:3000 (admin / admin) |

> Grafana carga el dashboard automáticamente al iniciar.

### Dashboard incluido

El dashboard `tasks-api.json` muestra:

| Panel | Métrica |
|-------|---------|
| HTTP Requests/s | Tasa de requests por endpoint y método |
| Latencia p95 | Percentil 95 de tiempo de respuesta (ms) |
| Tasa de errores | Requests con status 4xx o 5xx |
| JVM Heap | Memoria heap usada vs máxima (MB) |
| Threads activos | Threads vivos en la JVM |
| Uptime | Tiempo en línea de la instancia |
| Total requests | Requests en los últimos 30 minutos |
| CPU usage | % de CPU del proceso (gauge con thresholds) |

---

## 🔒 Decisiones técnicas

- **Separación de jobs**: build-and-push y deploy son jobs independientes — si el push a ECR falla, el deploy no corre
- **Image promotion GHCR → ECR**: la imagen se construye una sola vez en Repo 1, se promueve a ECR sin reconstruir
- **Doble tag en ECR**: `:latest` para referencia fácil y `:sha-<commit>` para trazabilidad exacta
- **`wait-for-service-stability: true`**: el job no termina hasta que ECS confirma que el nuevo task está healthy
- **Smoke test post-deploy**: verifica que la app responde vía ALB antes de reportar éxito
- **`environment: production`**: el job de deploy requiere aprobación manual si se configura en GitHub
- **Workflow reutilizable**: el deploy puede llamarse desde otros repos sin duplicar lógica
- **Provisioning automático en Grafana**: datasource y dashboards se cargan al iniciar el container, sin configuración manual

---

## 👤 Autor

- [@JoshuaSMC](https://github.com/JoshuaSMC)

---

## 📄 Licencia

MIT
