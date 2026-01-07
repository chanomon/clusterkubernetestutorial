# clusterkubernetestutorial
Tutorial para desplegar un cluster de kubernetes usando k3s. El host o master esta en un contenedor docker. Tambien se desplegara el servicio de linkding.

Tutorial Completo: Kubernetes (K3s) en Docker con Linkding
📚 Tabla de Contenidos

Introducción y Conceptos

Requisitos Previos

Parte 1: Instalación y Configuración del Entorno

Parte 2: Contenedor Docker Mejorado con K3s y Herramientas

Parte 3: Configuración de Kubernetes y Linkding

Parte 4: Operaciones Diarias y Mantenimiento

Solucionando Problemas Comunes

Conclusión y Próximos Pasos

Introducción y Conceptos
¿Qué es Docker?

Docker es una plataforma de contenedores que permite empaquetar aplicaciones y sus dependencias en unidades estandarizadas llamadas contenedores.

Piensa en un contenedor como una caja liviana y portátil que contiene todo lo necesario para ejecutar una aplicación: código, bibliotecas, variables de entorno y archivos de configuración.

¿Qué es Kubernetes y K3s?

Kubernetes (K8s) es un orquestador de contenedores que automatiza:

Despliegue

Escalado

Gestión de aplicaciones

K3s es una distribución ligera de Kubernetes desarrollada por Rancher, ideal para:

Entornos con recursos limitados

Desarrollo local

Edge y homelabs

¿Qué es Linkding?

Linkding es un gestor de marcadores auto-alojado, similar a:

Pocket

Raindrop.io

Permite guardar, organizar y buscar enlaces web.

Arquitectura del Sistema
Tu Máquina Local
   ↓
Docker Engine
   ↓
Contenedor K3s Mejorado
   ↓
Cluster K3s
   ↓
Pods (Linkding)

Requisitos Previos

Sistema operativo: Linux (Ubuntu / Debian recomendado)

4 GB RAM mínimo (8 GB recomendado)

20 GB de espacio libre en disco

Acceso a internet

Conocimientos básicos de terminal

Parte 1: Instalación y Configuración del Entorno
Crear estructura de directorios

Organizar los archivos es crucial para un manejo eficiente del cluster.

# Crear directorio principal para K3s
mkdir -p ~/k3s-cluster/{manifests,config,data,logs,backups}

# Estructura creada:
# ~/k3s-cluster/
# ├── manifests/    # Archivos YAML de Kubernetes
# ├── config/       # Archivos de configuración
# ├── data/         # Datos persistentes
# ├── logs/         # Logs del sistema
# └── backups/      # Copias de seguridad

Parte 2: Contenedor Docker Mejorado con K3s y Herramientas
Dockerfile Mejorado con K3s y herramientas

Este Dockerfile crea una imagen personalizada que incluye:

K3s

Bash

Neovim

Herramientas de desarrollo y administración

# ~/k3s-cluster/Dockerfile.k3s-complete
FROM alpine:3.18 as base

RUN apk add --no-cache docker-cli

FROM rancher/k3s:v1.29.1-k3s1 as k3s-base

FROM k3s-base

RUN apk add --no-cache \
    bash bash-completion vim neovim curl wget htop tree git sudo \
    openssh-client tmux jq yq ncdu rsync unzip tar gzip \
    bind-tools iputils net-tools busybox-extras

RUN sed -i 's/ash/bash/g' /etc/passwd

EXPOSE 6443 80 443 9090 30000-32767

CMD ["/bin/k3s", "server"]

Construir y ejecutar el contenedor
# Construir la imagen
docker build -t k3s-complete -f Dockerfile.k3s-complete .

# Ejecutar el contenedor

  docker run -d \
    --name k3s-server \
    --privileged \
    --restart=unless-stopped \
    -p 6443:6443 \
    -p 80:80 \
    -p 443:443 \
    -p 9090:9090 \
    -v ~/k3s-cluster/data:/var/lib/rancher/k3s \
    -v ~/k3s-cluster/manifests:/var/lib/rancher/k3s/server/manifests \
    -v ~/k3s-cluster/config:/etc/rancher/k3s \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v ~/k3s-cluster/logs:/var/log/k3s \
    k3s-complete \
    server \
    --cluster-init \
    --disable traefik \
    --disable servicelb

Parte 3: Configuración de Kubernetes y Linkding
ConfigMaps y Secrets

ConfigMap

Configuración no sensible

URLs, puertos, flags

Secret

Datos sensibles

Contraseñas, tokens, claves

Crear estructura para Linkding
mkdir -p ~/k3s-cluster/manifests/linkding
cd ~/k3s-cluster/manifests/linkding

1. Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: linkding

2. ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: linkding-config
  namespace: linkding
data:
  LD_DB_ENGINE: "sqlite3"
  LD_DB_NAME: "/data/linkding.db"
  LD_PORT: "9090"
  TZ: "America/Mexico_City"

3. Secret
apiVersion: v1
kind: Secret
metadata:
  name: linkding-secrets
  namespace: linkding
type: Opaque
stringData:
  LD_SECRET_KEY: "cambia-esto-en-produccion"

4. PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: linkding-data
  namespace: linkding
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: local-path

5. Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: linkding
  namespace: linkding
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    spec:
      containers:
      - name: linkding
        image: sissbruecker/linkding:latest
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: linkding-data

6. Service (NodePort)
apiVersion: v1
kind: Service
metadata:
  name: linkding-nodeport
  namespace: linkding
spec:
  type: NodePort
  ports:
    - port: 9090
      nodePort: 30090
  selector:
    app: linkding

Aplicar configuración
kubectl apply -f ~/k3s-cluster/manifests/linkding
kubectl get all -n linkding

Parte 4: Operaciones Diarias y Mantenimiento
Script de gestión
linkding start
linkding stop
linkding restart
linkding status
linkding logs

Solucionando Problemas Comunes
ImagePullBackOff
kubectl describe pod -n linkding <pod>
docker pull sissbruecker/linkding:latest

PVC en estado Pending
kubectl get storageclass
kubectl describe pvc -n linkding linkding-data

Conclusión y Próximos Pasos

Has creado un entorno completo con:

✅ Docker
✅ K3s en contenedor
✅ Herramientas de desarrollo
✅ Linkding en Kubernetes
✅ Scripts de mantenimiento

Próximos pasos

HTTPS con cert-manager

CI/CD con ArgoCD

Monitoreo con Prometheus + Grafana

Logging con Loki

Backups con Velero

GitOps con Flux
