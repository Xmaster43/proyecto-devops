================================================================================
  PROYECTO DEVOPS - INNOVATECH CHILE
  Evaluación Parcial N°3: Orquestación y Automatización en la Nube
================================================================================

Asignatura: ISY1101 - Introducción a Herramientas DevOps
Integrantes: Josefa Nuñez, Bastian Gomez, Ivan Hernandez
Fecha: Junio 2026

================================================================================
DESCRIPCIÓN GENERAL DEL PROYECTO
================================================================================

Este proyecto implementa una SOLUCIÓN DEVOPS COMPLETA EN AWS EKS (KUBERNETES) 
con automatización CI/CD mediante GitHub Actions. Gestiona una aplicación de 
microservicios para INNOVATECH CHILE (gestión de Despachos y Ventas).

OBJETIVOS CUMPLIDOS:

✓ Orquestación: Clúster EKS con Deployments y Services
✓ Automatización: Pipeline CI/CD completo (build → push → deploy)
✓ Escalabilidad: Replicas y preparación para autoscaling
✓ Seguridad: Secrets en GitHub, VPC privadas, Security Groups
✓ Monitoreo: Logs en CloudWatch, métricas en EKS

================================================================================
ARQUITECTURA
================================================================================

DIAGRAMA DE LA INFRAESTRUCTURA:

┌─────────────────────────────────────────────────────────────┐
│                   AWS Región: US-EAST-1                     │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                 INTERNET (Público)                    │   │
│  └──────────────────────┬───────────────────────────────┘   │
│                         │                                     │
│  ┌──────────────────────▼───────────────────────────────┐   │
│  │         Application Load Balancer (ALB)              │   │
│  │        IP Pública: a985749b1c2114b01bec...          │   │
│  └──────────────────────┬───────────────────────────────┘   │
│                         │                                     │
│  ┌──────────────────────▼───────────────────────────────┐   │
│  │              EKS KUBERNETES CLUSTER                  │   │
│  │                                                       │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │   │
│  │  │   Frontend  │  │  Despachos  │  │   Ventas    │ │   │
│  │  │   (Nginx)   │  │   Backend   │  │   Backend   │ │   │
│  │  │  Pod x2     │  │   Pod x2    │  │   Pod x2    │ │   │
│  │  │  Puerto:80  │  │  Puerto:    │  │   Puerto:   │ │   │
│  │  │             │  │   8081      │  │    8082     │ │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘ │   │
│  │        ↓                ↓                 ↓          │   │
│  │  ┌──────────────────────────────────────────┐       │   │
│  │  │        MySQL Database (Pod)              │       │   │
│  │  │     Puerto 3306 (Red Interna)            │       │   │
│  │  └──────────────────────────────────────────┘       │   │
│  │                                                       │   │
│  │  Red Interna: 172.20.0.0/16 (ServiceMesh)          │   │
│  └───────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │           AMAZON ECR (Container Registry)            │   │
│  │  - 637423587001.dkr.ecr.us-east-1.amazonaws.com    │   │
│  │  - innovatech-backend:despachos-v1                   │   │
│  │  - innovatech-backend:ventas-v1                      │   │
│  │  - innovatech-frontend:v1                            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │     CloudWatch (Logs & Métricas)                     │   │
│  │  - /aws/eks/cluster/innovatech-cluster              │   │
│  │  - CPU, Memory, Network I/O                          │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
└─────────────────────────────────────────────────────────────┘

              ↑
              │
    ┌─────────┴──────────┐
    │   GITHUB ACTIONS   │
    │   CI/CD Pipeline   │
    │                    │
    │ 1. Trigger: push   │
    │    a branch        │
    │ 2. Build imagen    │
    │ 3. Push a ECR      │
    │ 4. Deploy en EKS   │
    │    (kubectl)       │
    └────────────────────┘

================================================================================
COMPONENTES PRINCIPALES
================================================================================

1. CLÚSTER EKS (KUBERNETES)

Nombre: innovatech-cluster
Versión Kubernetes: 1.28+
Nodos: t3.medium (2-3 nodos)
VPC: Privada con subredes separadas
Autoscaling: Habilitado (1-5 nodos)


2. DEPLOYMENTS (K8S)

FRONTEND (Nginx + React):
- Nombre: frontend-despacho
- Replicas: 2 (Alta disponibilidad)
- Puerto expuesto: 80
- Imagen: innovatech-frontend:v1
- Service: LoadBalancer (acceso público)

BACKEND DESPACHOS:
- Nombre: despachos-api
- Replicas: 2
- Puerto: 8081
- Imagen: innovatech-backend:despachos-v1
- Service: ClusterIP (interno)

BACKEND VENTAS:
- Nombre: ventas-api
- Replicas: 2
- Puerto: 8082
- Imagen: innovatech-backend:ventas-v1
- Service: ClusterIP (interno)

MYSQL DATABASE:
- Nombre: mysql-deployment
- Replicas: 1
- Puerto: 3306
- Volumen: PersistentVolume (datos persistentes)
- Secretos: Credenciales en AWS Secrets Manager


3. ECR (ELASTIC CONTAINER REGISTRY)

Todas las imágenes se almacenan en un registro privado:

637423587001.dkr.ecr.us-east-1.amazonaws.com/
├── innovatech-backend:despachos-v1
├── innovatech-backend:ventas-v1
└── innovatech-frontend:v1


4. PIPELINE CI/CD (GITHUB ACTIONS)

FLUJO AUTOMÁTICO:

Developer hace push a rama 'deploy'
    ↓
GitHub Actions detecta trigger
    ↓
Checkout código
    ↓
Configure AWS credentials
    ↓
Login en Amazon ECR
    ↓
Build imagen Docker
    ↓
Push imagen a ECR
    ↓
Deploy en EKS (kubectl apply)
    ↓
Rollback automático si falla

ARCHIVOS WORKFLOW:
- .github/workflows/deploy-front.yaml (Frontend)
- .github/workflows/deploy-despachos.yaml (Backend Despachos)
- .github/workflows/deploy-ventas.yaml (Backend Ventas)

SECRETS NECESARIOS EN GITHUB:
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_SESSION_TOKEN
AWS_REGION (us-east-1)
ECR_REGISTRY
ECR_REPO_URL_FRONTEND
ECR_REPO_URL_BACKEND_DESPACHOS
ECR_REPO_URL_BACKEND_VENTAS
EC2_FRONTEND_INSTANCE_ID

================================================================================
CONFIGURACIÓN DE AUTOSCALING (HPA)
================================================================================

HORIZONTAL POD AUTOSCALER (HPA)

Aunque en este proyecto usamos replicas estáticas (2), la configuración HPA 
permitiría escalar automáticamente según demanda.

CONFIGURACIÓN RECOMENDADA:

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: despachos-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: despachos-api
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80

JUSTIFICACIÓN DE VALORES:
- Min Replicas: 2 (alta disponibilidad, mínimo dos instancias)
- Max Replicas: 5 (evita costos excesivos en AWS)
- CPU Threshold: 70% (escalado agresivo cuando CPU sube)
- Memory Threshold: 80% (protege contra Out of Memory)

APLICAR HPA:
kubectl apply -f k8s/hpa-config.yaml
kubectl get hpa -w  (monitorear cambios)

================================================================================
INSTALACIÓN Y DESPLIEGUE
================================================================================

PREREQUISITOS:

# AWS CLI configurado
aws configure

# kubectl instalado
kubectl version --client

# Docker instalado (para testing local)
docker --version

# Git
git clone https://github.com/Xmaster43/proyecto-devops.git
cd proyecto-devops


PASOS PARA DESPLEGAR:

PASO 1: CREAR/ACCEDER AL CLÚSTER EKS

# Actualizar kubeconfig
aws eks update-kubeconfig \
  --region us-east-1 \
  --name innovatech-cluster

# Verificar conexión
kubectl get nodes

Salida esperada:
NAME                         STATUS   ROLES    
ip-172-31-x-x.ec2.internal  Ready    <none>
ip-172-31-y-y.ec2.internal  Ready    <none>


PASO 2: CREAR SECRETS EN KUBERNETES

# Crear secret para credenciales MySQL
kubectl create secret generic mysql-secret \
  --from-literal=MYSQL_USER=admin \
  --from-literal=MYSQL_PASSWORD=admin123 \
  --from-literal=MYSQL_ROOT_PASSWORD=root123

# Verificar
kubectl get secrets


PASO 3: DESPLEGAR SERVICIOS

# Aplicar todos los manifiestos K8s
kubectl apply -f k8s/mysql-k8s.yaml
sleep 10
kubectl apply -f k8s/backends-k8s.yaml
sleep 10
kubectl apply -f k8s/frontend-k8s.yaml

# Verificar que los pods están corriendo
kubectl get pods -w
# Esperar a que todos muestren "Running"

# Verificar servicios
kubectl get svc


PASO 4: OBTENER URL DE ACCESO

# Obtener IP/URL del LoadBalancer
kubectl get svc frontend-despacho -o wide

# La URL será algo como:
# a985749b1c2114b01bec52fec2bebb3f-75497823.us-east-1.elb.amazonaws.com

# Acceder en navegador:
# http://a985749b1c2114b01bec52fec2bebb3f-75497823.us-east-1.elb.amazonaws.com

================================================================================
MONITOREO Y MÉTRICAS
================================================================================

CLOUDWATCH LOGS:

# Ver logs del Frontend
kubectl logs -f deployment/frontend-despacho

# Ver logs de Despachos
kubectl logs -f deployment/despachos-api

# Ver logs de Ventas
kubectl logs -f deployment/ventas-api

# Ver logs de MySQL
kubectl logs -f deployment/mysql-deployment


MÉTRICAS DE CPU Y MEMORIA:

# Ver uso de recursos en tiempo real
kubectl top nodes
kubectl top pods

Salida esperada:
NAME                       CPU(cores)   MEMORY(Mi)
despachos-api-xxxxx        45m          256Mi
ventas-api-xxxxx           38m          220Mi
frontend-despacho-xxxxx    12m          64Mi
mysql-deployment-xxxxx     120m         512Mi


DASHBOARD KUBERNETES:

# Iniciar proxy para acceder al dashboard
kubectl proxy

# Acceder en navegador:
# http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

================================================================================
FLUJO CI/CD COMPLETO
================================================================================

EJEMPLO: DESPLEGAR UN CAMBIO EN BACKEND DESPACHOS

PASO 1: Hacer cambio en código
vi back-Despachos_SpringBoot/Springboot-API-REST-DESPACHO/src/main/java/...

PASO 2: Commit con mensaje descriptivo
git add .
git commit -m "feat: Agregar endpoint de validación de despachos"

PASO 3: Push a rama 'deploy'
git push origin deploy

PASO 4: GitHub Actions se dispara automáticamente:
- Compila código
- Ejecuta tests
- Build imagen Docker
- Push a ECR
- Deploy en EKS

PASO 5: Monitorear progreso
kubectl get rollout status deployment/despachos-api -w

PASO 6: Verificar que el cambio está vivo
curl http://<ALB-URL>:8081/api/despachos

================================================================================
SEGURIDAD IMPLEMENTADA
================================================================================

SECRETS Y CREDENCIALES:
✓ MySQL password en Kubernetes Secrets (no en código)
✓ AWS credentials en GitHub Secrets (cifradas)
✓ IAM roles con mínimo privilegio
✓ VPC privada para bases de datos


NETWORKING:
✓ Security Groups: Frontend abierto (80), Backend cerrado
✓ NACLs: Bloqueo de tráfico no autorizado
✓ DNS interno: Nombres de servicios (mysql-service, despachos-service)


RBAC (ROLE-BASED ACCESS CONTROL):
Los pods solo pueden:
- Leer secrets de MySQL
- Conectarse entre servicios
- Acceder a volúmenes asignados

================================================================================
PROBLEMAS ENCONTRADOS Y SOLUCIONES
================================================================================

PROBLEMA 1: PODS EN CRASHLOOPBACKOFF

Causa: Credenciales MySQL incorrectas

Solución:
# Verificar secret
kubectl get secret mysql-secret -o yaml

# Revisar logs
kubectl logs deployment/despachos-api --previous

# Actualizar secret
kubectl delete secret mysql-secret
kubectl create secret generic mysql-secret \
  --from-literal=MYSQL_USER=admin \
  --from-literal=MYSQL_PASSWORD=admin123


PROBLEMA 2: FRONTEND NO PUEDE CONECTAR CON BACKENDS

Causa: URL incorrecta en environment

Solución:
En frontend-k8s.yaml, usar nombres de servicios internos:

env:
  - name: REACT_APP_API_DESPACHOS
    value: "http://despachos-service:8081"
  - name: REACT_APP_API_VENTAS
    value: "http://ventas-service:8082"


PROBLEMA 3: ESPACIO EN DISCO AGOTADO EN EC2

Causa: Imágenes Docker antiguas acumuladas

Solución:
En el workflow, agregar:

- name: Cleanup images
  run: |
    docker image prune -af
    docker container prune -f

================================================================================
CHECKLIST DE VERIFICACIÓN
================================================================================

Antes de la presentación, verificar:

[ ] Clúster EKS está activo y saludable
    kubectl get nodes
    kubectl get pods -A

[ ] Todos los pods están en estado "Running"
    kubectl get pods

[ ] Frontend es accesible por URL pública
    curl http://<ALB-URL>

[ ] Backends responden a requests
    curl http://<ALB-URL>:8081/api/despachos
    curl http://<ALB-URL>:8082/api/ventas

[ ] Logs están siendo registrados
    kubectl logs deployment/despachos-api

[ ] GitHub Actions pipeline completa exitosamente
    Ir a: https://github.com/Xmaster43/proyecto-devops/actions
    Ver últimas ejecuciones (deben estar en verde ✓)

[ ] Commits tienen mensajes descriptivos
    git log --oneline | head -20

================================================================================
EVIDENCIA DE EJECUCIÓN
================================================================================

CAPTURA DE PODS CORRIENDO:

$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
despachos-api-c5b45f6c6-7t6fq           1/1     Running   0          3m25s
despachos-api-c5b45f6c6-hxkwr           1/1     Running   0          3m13s
frontend-despacho-6bd9b77c-j9v4h        1/1     Running   0          3m24s
frontend-despacho-6bd9b77c-xqjrp        1/1     Running   0          3m21s
mysql-deployment-74d8b4b9df-9vwdw       1/1     Running   0          3m23s
ventas-api-66c54764cd-6djrr             1/1     Running   0          3m26s
ventas-api-66c54764cd-qmgpw             1/1     Running   0          3m26s


CAPTURA DE GITHUB ACTIONS:

✓ deploy-front.yaml (35s)
✓ deploy-despachos.yaml (55s)
✓ deploy-ventas.yaml (1m 16s)



================================================================================
CONCEPTOS DEVOPS APLICADOS
================================================================================

1. INFRASTRUCTURE AS CODE (IaC)

- Kubernetes manifests (YAML files)
- Docker multi-stage builds
- AWS CloudFormation (para crear EKS)

Ventajas:
- Reproducible en cualquier entorno
- Versionable en Git
- Fácil de compartir entre equipos


2. CONTINUOUS INTEGRATION (CI)

- GitHub Actions detecta cambios
- Automatic build de imágenes
- Push a registry privado (ECR)

Ventajas:
- Builds consistentes
- Early detection de errores
- Automatización de testing


3. CONTINUOUS DEPLOYMENT (CD)

- Automatic deploy a EKS
- Rollout updates sin downtime
- Rollback automático si falla

Ventajas:
- Releases rápidas
- Cero downtime
- Recuperación automática


4. MICROSERVICIOS

- Frontend independiente
- 2 backends con responsabilidades distintas
- Comunicación vía DNS interno

Ventajas:
- Escalabilidad selectiva
- Equipos independientes
- Despliegues aislados


5. ALTA DISPONIBILIDAD

- Múltiples replicas por servicio
- Load balancing automático
- Healthchecks y auto-recovery

Ventajas:
- Tolerancia a fallos
- Uptime 99.9%+
- Escalabilidad horizontal


6. SEGURIDAD

- Secrets cifrados
- RBAC en Kubernetes
- VPC privada
- Security Groups

Ventajas:
- Credenciales protegidas
- Acceso basado en roles
- Tráfico controlado

================================================================================
REFERENCIAS Y DOCUMENTACIÓN
================================================================================

- AWS EKS Documentation:
  https://docs.aws.amazon.com/eks/

- Kubernetes Official Docs:
  https://kubernetes.io/docs/

- GitHub Actions Docs:
  https://docs.github.com/en/actions

- Docker Best Practices:
  https://docs.docker.com/develop/dev-best-practices/

- Helm Package Manager:
  https://helm.sh/

================================================================================
CONTRIBUYENTES
================================================================================

- Josefa Nuñez         - Arquitectura EKS y Kubernetes
- Bastian Gomez       - Pipeline CI/CD y GitHub Actions
- Ivan Hernandez      - Dockerización y optimización de imágenes


================================================================================
INFORMACIÓN FINAL
================================================================================

Última actualización: Junio 12, 2026
Estado:Producción
Versión: 1.0.0


================================================================================
FIN DEL DOCUMENTO
================================================================================