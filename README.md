Esta práctica tiene como objetivo implementar una serie de mejoras al código para despIegarlo en producción. Para ello se debe tener en cuenta los siguientes prerequisitos:
# Tareas a realizar
- Añadir un nuevo endpoint `/bye`  a la aplicación <b>Fast API</b>  y que devuelva como mensaje `Bye Bye`
- Creación de test unitarios para el nuevo endpoint
```bash 
pytest 
```
```bash 
pytest --cov
```
```bash 
pytest --cov --cov-report=html 
```
- Creación del Helm chart para desplegar la aplicación en Kubernetes. Para ello inicializamos un minikube que lleva por nombre `fastapi`
 ```bash 
 minikube start -p monitoring-test
 ```
 Configuramos las alarmas de Alert-manager en slack, añadiendo la app webhook y conectandolo al canal `#prometheus-alarmas` se monitoriza la CPU y se envian las alertas criticas al canal mencionado.
 ```bash
 slack_configs:
      - api_url: 'https://hooks.slack.com/services/url-del-hook'
        send_resolved: true
        channel: '#prometheus-alarmas'
 ```
  Añadimos el repositorio de Helm prometheus community
 
 ```bash
 helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
 helm repo update
```
Desplegamos el chart de kube-prometheus-stack del helm con los valores definidos en el fichero custom_values_prometheus.yaml
```bash
helm -n monitoring upgrade --install prometheus prometheus-community/kube-prometheus-stack -f custom_values_prometheus.yaml --create-namespace --wait --version 34.1.1
```

Verificamos que los pods han sido creados correctamente para el stack de monitoring
```bash
kubectl get po -n monitoring -o wide
```
Añadimos el repositorio de alta disponibilidad

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```
Habilitamos el metric server al cluster creado anteriormente `fastapi`, que es quien va a permitir autoescalar
```bash
minikube addons enable metrics-server -p monitoring-demo
    ▪ Using image k8s.gcr.io/metrics-server/metrics-server:v0.4.2
🌟  The 'metrics-server' addon is enabled
```
Creación de un helm chart de la aplicación
```bash
helm dep up fast-api-webapp
```
Desplegamos la aplicación y observamos los pods estan siendo creados

```bash
helm -n fast-api upgrade my-app --install --create-namespace fast-api-webapp
kubectl -n fast-api get po -w
```
Simulamos una prueba de estres .
Para ello nos conectamos al siguiente pod.
```bash
export POD_NAME=$(kubectl get pods --namespace fast-api -l "app.kubernetes.io/name=fast-api-webapp,app.kubernetes.io/instance=my-app" -o jsonpath="{.items[0].metadata.name}")
kubectl -n fast-api exec --stdin --tty $POD_NAME -- /bin/sh
apk update && apk add git go
git clone https://github.com/jaeg/NodeWrecker.git
cd NodeWrecker
go build -o extress main.go
./extress -abuse-memory -escalate -max-duration 10000000
```
importamos el deashboard de grafana :
```bash
kubectl -n monitoring port-forward svc/prometheus-grafana 3000:3000

kubectl -n fast-api port-forward svc/my-app-fast-api-webapp 8081:8081

```