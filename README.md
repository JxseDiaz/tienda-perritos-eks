# Tienda de Perritos — CI/CD en AWS EKS

Proyecto de la Evaluación Final Transversal (ISY1101 - Introducción a Herramientas DevOps, Duoc UC).
Automatiza la integración, empaquetado y despliegue continuo de una tienda online de productos
para perros, compuesta por frontend, backend y base de datos relacional.

## Arquitectura

```
Usuario → [Frontend: Nginx] → /api/* → [Backend: Node/Express] → [MySQL]
```

- **Frontend**: HTML/JS estático servido por Nginx (imagen `nginx:alpine`). Actúa además como
  proxy reverso hacia el backend en la ruta `/api/`.
- **Backend**: API REST en Node.js/Express (imagen `node:18-alpine`) que expone operaciones
  CRUD sobre productos, conectándose a MySQL mediante `mysql2/promise`.
- **Base de datos**: MySQL 8, inicializada con `db/init.sql` (tabla `productos` con datos de
  ejemplo).

En Kubernetes, la comunicación entre componentes se resuelve por DNS interno del clúster
usando los `Service` de cada componente (`tienda-backend`, `tienda-db`), por eso el nginx del
frontend y el backend usan esos mismos nombres como hostname tanto en local (docker-compose)
como en EKS.

## Cómo levantarlo en local

Requiere Docker y Docker Compose.

```bash
docker compose up --build
```

- Frontend: http://localhost:8080
- Backend (health check): http://localhost:3001/api/health

Para apagar todo:

```bash
docker compose down       # mantiene los datos
docker compose down -v    # borra también los datos de la BD
```

## Cómo se despliega (CI/CD)

Cada `push` a `main` dispara el workflow `.github/workflows/deploy-eks.yml`, que:

1. Construye y sube (`docker build` + `docker push`) las 3 imágenes a Amazon ECR, etiquetadas
   con el hash corto del commit.
2. Configura `kubectl` contra el clúster EKS (`tienda-cluster`).
3. Aplica los manifests de `k8s/` (namespace, secret, deployments, services) y actualiza las
   imágenes de los Deployments con `kubectl set image`.
4. Aplica los HPA de backend y frontend.
5. Muestra el estado final (`pods`, `svc`, `hpa`) como evidencia.

Las credenciales de AWS se gestionan como **GitHub Secrets** (`AWS_ACCESS_KEY_ID`,
`AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, `AWS_REGION`, `EKS_CLUSTER_NAME`,
`EKS_NAMESPACE`), ya que AWS Academy entrega credenciales temporales que deben renovarse
en cada sesión de laboratorio. La contraseña de MySQL en el clúster se gestiona como un
`Secret` de Kubernetes (`k8s/mysql-secret.yaml`), nunca en texto plano en los manifests de
Deployment.

## Estructura del repositorio

```
.
├── backend/            # API Node/Express + Dockerfile
├── frontend/            # Estático + Nginx + Dockerfile
├── db/                  # MySQL + script de inicialización + Dockerfile
├── k8s/                 # Manifests de Kubernetes (namespace, secret, deployments, services, HPA)
├── docker-compose.yml    # Orquestación local para desarrollo
└── .github/workflows/    # Pipeline de CI/CD (GitHub Actions)
```

## Escalabilidad

Backend y frontend cuentan con `HorizontalPodAutoscaler` (`k8s/backend-hpa.yaml`,
`k8s/frontend-hpa.yaml`) que escala el número de pods según el uso de CPU, usando
Metrics Server como fuente de métricas.
