# Despliegue de Grafana en Kubernetes

Práctica individual de **Procesos de Software (UDLA)** para aplicar los conceptos de Deployment, ReplicaSet y Service en Kubernetes.

## Tecnologías

- Kubernetes mediante Minikube con el controlador Docker
- Docker Desktop en Windows 11
- Grafana 11.6.0 (`grafana/grafana:11.6.0`)
- kubectl

## Estructura

```text
k8s/
├── deployment.yaml
└── service.yaml
```

`deployment.yaml` define dos réplicas iniciales de Grafana en el puerto 3000. Kubernetes crea un ReplicaSet para mantener la cantidad de Pods solicitada. `service.yaml` expone los Pods con un Service de tipo NodePort en el puerto 30543.

Cada contenedor solicita 100m de CPU y 128Mi de memoria; sus límites son 500m de CPU y 512Mi de memoria.

## Despliegue

Desde la raíz del repositorio, con Docker Desktop activo:

```powershell
minikube start --driver=docker
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl get deployments
kubectl get rs
kubectl get pods -l app=grafana -o wide
kubectl get svc grafana-service
```

Para abrir Grafana:

```powershell
minikube service grafana-service
```

En Windows con el driver Docker, Minikube puede crear un túnel y mostrar una dirección `127.0.0.1` con un puerto temporal. La terminal del túnel debe permanecer abierta durante el acceso.

## Escalado horizontal

El Deployment comienza con dos réplicas. Para aumentar a tres:

```powershell
kubectl scale deployment grafana-deployment --replicas=3
kubectl rollout status deployment/grafana-deployment
kubectl get deployments
kubectl get rs
kubectl get pods -l app=grafana -o wide
```

**Resultado de la práctica:** Deployment `3/3`, ReplicaSet con tres réplicas listas y tres Pods de Grafana en estado `Running`.

## Autorrecuperación

Listar los Pods y seleccionar uno de Grafana:

```powershell
kubectl get pods -l app=grafana
```

Eliminar el Pod seleccionado, sustituyendo el nombre de ejemplo por uno que exista en el clúster:

```powershell
kubectl delete pod grafana-deployment-5bdb898f5-xbxf9
kubectl rollout status deployment/grafana-deployment
kubectl get deployments
kubectl get rs
kubectl get pods -l app=grafana -o wide
```

**Resultado observado:** se eliminó el Pod `grafana-deployment-5bdb898f5-xbxf9` y Kubernetes creó `grafana-deployment-5bdb898f5-2j8zk`. El Deployment recuperó tres réplicas listas, sin intervención manual para crear el Pod de reemplazo.

Los nombres de los Pods son generados automáticamente y serán diferentes en una nueva ejecución.

## ¿Para qué se usa Grafana?

Grafana permite visualizar información de distintas fuentes de datos mediante dashboards, consultar métricas y logs, configurar alertas y apoyar el monitoreo de aplicaciones e infraestructura. En esta actividad se demuestra **su despliegue y exposición**, no la configuración de dashboards ni de fuentes de datos.

## Consideraciones

Este entorno es educativo y utiliza **un único nodo de Minikube**. Las tres réplicas no equivalen a alta disponibilidad ante la caída del nodo. Tampoco se configura almacenamiento persistente o una base de datos compartida para Grafana: las credenciales y otros cambios locales podrían no sincronizarse entre Pods. Para producción se necesitaría una arquitectura de persistencia, seguridad y disponibilidad adecuada.

Las capturas reales de la ejecución y los resultados se documentan en el informe de la asignatura.
