# 📘 Microservices en AKS con NGINX Ingress + ArgoCD

------------------------------------------------------------------------

## 🌐 ¿Qué es un Ingress?

En Kubernetes, un **Ingress** es un recurso que permite exponer
servicios HTTP y HTTPS al exterior del clúster mediante reglas de
enrutamiento.

En lugar de exponer cada servicio con un `LoadBalancer` diferente (lo
que generaría múltiples IPs públicas y mayor coste), el Ingress actúa
como un **punto de entrada único** que:

-   Recibe todo el tráfico externo
-   Analiza la URL solicitada
-   Redirige la petición al servicio interno correspondiente
-   Permite centralizar TLS, reglas y control de tráfico

En este proyecto utilizamos **NGINX Ingress Controller**, que interpreta
las reglas del recurso Ingress y enruta el tráfico dentro del clúster.

Flujo real:

Internet\
↓\
Azure Load Balancer (IP pública)\
↓\
NGINX Ingress Controller\
↓\
Reglas del Ingress\
↓\
Service (ClusterIP)\
↓\
Pods

------------------------------------------------------------------------

## 🧱 Arquitectura

Aplicación compuesta por:

-   🖥 Frontend (Vite)
-   🐍 Backend Python (puerto 8000)
-   ☕ Backend Java (puerto 8080)
-   🌐 NGINX Ingress Controller
-   ☁️ Azure Kubernetes Service (AKS)
-   🔁 ArgoCD (GitOps)

Todo el tráfico entra por la IP pública del Ingress Controller.

------------------------------------------------------------------------

# 🔥 Paso 0 -- Instalar y Exponer NGINX Ingress Controller

## Instalar NGINX Ingress Controller

``` bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

## Verificar Service del Ingress

``` bash
kubectl get svc -n ingress-nginx
```

Debe aparecer:

-   TYPE: LoadBalancer
-   EXTERNAL-IP: `<IP pública asignada>`{=html}

Esa IP será la puerta de entrada de la aplicación.

------------------------------------------------------------------------

# 🧭 Paso 1 -- Crear Namespace

``` yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ejemplito
```

------------------------------------------------------------------------

# 🐍 Paso 2 -- Deployment Backend

Deployment con contenedor Python (8000) y Java (8080).

------------------------------------------------------------------------

# 🔌 Paso 3 -- Service Backend

Service tipo ClusterIP exponiendo ambos puertos:

-   8000 → Python
-   8080 → Java

------------------------------------------------------------------------

# 🖥 Paso 4 -- Deployment Frontend

Deployment del frontend servido en puerto 80.

------------------------------------------------------------------------

# 🔌 Paso 5 -- Service Frontend

Service tipo ClusterIP apuntando al frontend.

------------------------------------------------------------------------

# 🌍 Paso 6 -- Ingress

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: ejemplito
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:

      - path: /
        pathType: Prefix
        backend:
          service:
            name: front-service
            port:
              number: 80

      - path: /api/python
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-backend-service
            port:
              number: 8000

      - path: /api/java
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-backend-service
            port:
              number: 8080
```

------------------------------------------------------------------------

# 🎯 Variables del Frontend

``` env
VITE_CORE_API=/api/python
VITE_AUDIT_API=/api/java
VITE_ENDPOINT=/v1/simpson
```

No se debe hardcodear IP ni puertos internos.

------------------------------------------------------------------------

# 🔍 Comandos útiles

``` bash
kubectl get pods -n ejemplito
kubectl get svc -n ejemplito
kubectl get ingress -n ejemplito
kubectl get svc -n ingress-nginx
```

------------------------------------------------------------------------

# ❗ Problemas comunes

503 Service Temporarily Unavailable\
→ El Service del Ingress no coincide con el nombre real del Service.

404 Whitelabel Error\
→ La ruta no coincide con el RequestMapping del backend.

ERR_CONNECTION_REFUSED\
→ Se está llamando directamente a :8000 o :8080 desde el navegador.

------------------------------------------------------------------------

# 🏗 Arquitectura Final

Browser\
↓\
Azure Load Balancer\
↓\
NGINX Ingress Controller\
↓\
ClusterIP Services\
↓\
Pods
