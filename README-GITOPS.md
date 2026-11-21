# Taller individual: GitOps con Argo CD y Kubernetes

**Asignatura:** Procesos de Software (UDLA)  
**Aplicación:** Grafana 11.6.0  
**Entorno de ejecución:** Docker Desktop, Minikube y Kubernetes en Windows 11

Esta práctica reutiliza el despliegue de Grafana del taller anterior, pero incorpora **gestión declarativa desde GitHub mediante Argo CD** y un **Horizontal Pod Autoscaler (HPA)**. No se modifica el código del taller anterior en `main`.

## Estructura

| Ruta | Propósito |
| --- | --- |
| `gitops-argocd/deployment.yml` | Deployment con 2 réplicas, probes, labels, requests y limits |
| `gitops-argocd/service.yml` | Service NodePort, puerto 3000 y NodePort 30544 |
| `gitops-argocd/hpa.yml` | HPA autoscaling/v2, 2–5 réplicas, objetivo de CPU 50 % |
| `argocd/application.yaml` | Recurso Application que apunta a esta rama y activa la sincronización automática |

**Importante:** las tres definiciones que debe sincronizar Argo CD están exclusivamente en `gitops-argocd/`. El recurso `argocd/application.yaml` se crea **una sola vez** para registrar la aplicación en Argo CD.

## 1. Comprobación del entorno

En PowerShell, con Docker Desktop iniciado:

```powershell
minikube status
kubectl config use-context minikube
kubectl get nodes
Get-PSDrive C
```

Si Minikube está detenido, iniciarlo con:

```powershell
minikube start --driver=docker
```

Argo CD necesita recursos de cómputo adicionales. Si falla por memoria o disco, verificar primero la capacidad disponible en Docker Desktop/Minikube antes de hacer cambios.

## 2. Metrics Server

```powershell
minikube addons enable metrics-server
kubectl rollout status deployment/metrics-server -n kube-system --timeout=180s
kubectl top nodes
```

Metrics Server puede tardar en proporcionar datos. Sin métricas de CPU, el HPA no podrá calcular el porcentaje de utilización.

## 3. Instalación de Argo CD (según la presentación de clase)

```powershell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd
kubectl rollout status deployment/argocd-server -n argocd --timeout=300s
```

Si el namespace ya existe, no volver a crearlo; comprobarlo con `kubectl get namespace argocd`.

Abrir un **PowerShell independiente** y mantenerlo abierto:

```powershell
kubectl port-forward svc/argocd-server -n argocd 8089:443
```

Acceder a https://localhost:8089 y utilizar el usuario `admin`. Para leer la contraseña inicial en PowerShell:

```powershell
$encoded = kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}'
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($encoded))
```

No publicar esa contraseña ni el contenido de secretos en capturas o en el repositorio. Si ya se modificó la contraseña de Argo CD, el secreto inicial puede haber sido eliminado.

## 4. Registrar la aplicación GitOps

Clonar este repositorio o descargar el archivo `argocd/application.yaml` desde la rama `taller/argo-cd-gitops`. Ejecutar en la raíz del repositorio:

```powershell
kubectl apply -n argocd -f argocd/application.yaml
kubectl get applications.argoproj.io -n argocd
kubectl get deployment,svc,hpa -n grafana-gitops
kubectl get pods -n grafana-gitops -l app=grafana
```

Esta aplicación usa:
- **Repo:** `https://github.com/Santi7870/grafana-kubernetes.git`
- **Revision:** `taller/argo-cd-gitops`
- **Path:** `gitops-argocd`
- **Destination:** `grafana-gitops`
- **Automated sync:** `prune: true` y `selfHeal: true`

Si los Pods del entorno tienen recursos limitados, verificar primero los eventos y el estado del clúster antes de reintentar.

## 5. Verificar Deployment y Service

```powershell
kubectl rollout status deployment/grafana-deployment -n grafana-gitops --timeout=240s
kubectl get deployment,rs,pods,svc -n grafana-gitops -o wide
kubectl describe svc grafana-service -n grafana-gitops
```

En otra consola abrir la aplicación web con el NodePort:

```powershell
minikube service grafana-service -n grafana-gitops --url
```

Con el driver Docker para Windows, Minikube puede proporcionar una dirección local con puerto temporal. **Mantener abierta la consola que crea el túnel**.

## 6. Verificar HPA y escalado automático

```powershell
kubectl get hpa -n grafana-gitops
kubectl describe hpa grafana-hpa -n grafana-gitops
kubectl top pods -n grafana-gitops
kubectl get pods -n grafana-gitops -w
```

El HPA está configurado con 2 réplicas como mínimo, 5 como máximo y objetivo CPU de 50 %. Para generar tráfico de forma controlada, se puede crear temporalmente un Pod de carga dentro del namespace:

```powershell
kubectl run grafana-load -n grafana-gitops --image=busybox:1.36 --restart=Never -- /bin/sh -c 'while true; do wget -q -O /dev/null http://grafana-service:3000/api/health; done'
```

En otra consola revisar el HPA y las réplicas. **No se garantiza que esta carga de prueba eleve la CPU por encima del 50 %**. Si no escala, verificar las métricas y usar una prueba de carga más adecuada sin falsear el resultado.

Detener la carga:

```powershell
kubectl delete pod grafana-load -n grafana-gitops
```

El HPA tarda un tiempo en reducir réplicas tras caer la carga.

## 7. Demostrar la sincronización automática desde Git

En GitHub, desde la **rama `taller/argo-cd-gitops`**, modificar solamente la anotación del Deployment:

```yaml
annotations:
  udla.edu.ec/gitops-revision: "cambio-01"
```

Guardar con commit y comprobar en la interfaz de Argo CD la nueva revisión y el estado **Synced / Healthy**. También verificar la anotación con:

```powershell
kubectl get deployment grafana-deployment -n grafana-gitops -o jsonpath='{.spec.template.metadata.annotations.udla\.edu\.ec/gitops-revision}'
```

Argo CD puede tardar unos minutos en detectar el commit mediante su revisión periódica. No confundir sincronización automática desde Git con `kubectl apply` manual de los manifiestos.

## 8. Evidencias para el informe

Capturar: repositorio público y rama; manifiestos; Minikube y Metrics Server; recursos del namespace; dos Pods iniciales; Grafana en el navegador; panel Argo CD con repo/rama/path y estado sincronizado; commit de cambio en Git y nueva sincronización; HPA con métricas y escalado observado (si se consigue).

**Limitaciones técnicas:** Grafana usando SQLite local en múltiples réplicas no es una arquitectura de alta disponibilidad de producción; esta práctica verifica despliegue GitOps y HPA en un clúster local de laboratorio. Además, el HPA necesita métricas y carga suficiente, y un Pod de carga de red no asegura escalamiento CPU.

## 9. Referencias

- Presentación de clase: *Intro ARGO CD*, Mónica Sánchez, UDLA
- Documentación Argo CD: https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/
- HPA: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
- Metrics Server: https://minikube.sigs.k8s.io/docs/handbook/addons/metrics-server/
