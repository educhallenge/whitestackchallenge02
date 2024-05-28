# PROCEDIMIENTO DEL CHALLENGE 01 EN MICROSOFT AZURE


## PASO 1: DESPLEGAR APLICACIÓN WEB

En un virtual machine con Kubernetes en Microsoft Azure se clonó el repositorio nginx desde github.

```
azureuser@paragon:~$ git clone https://github.com/gespinozat/ws-challenge-2.git
Cloning into 'ws-challenge-2'...
remote: Enumerating objects: 15, done.
remote: Counting objects: 100% (15/15), done.
remote: Compressing objects: 100% (13/13), done.
remote: Total 15 (delta 3), reused 14 (delta 2), pack-reused 0
Receiving objects: 100% (15/15), 7.78 KiB | 2.59 MiB/s, done.
Resolving deltas: 100% (3/3), done.

azureuser@paragon:~$ cd ws-challenge-2/
azureuser@paragon:~/ws-challenge-2$ kubectl apply -f nginx.yaml
networkpolicy.networking.k8s.io/nginx created
serviceaccount/nginx created
secret/nginx-tls created
service/nginx created
deployment.apps/nginx created


## PASO 2: CREAR SERVICE MONITOR PARA OBTENCIÓN DE MÉTRICAS Y PASO 3: Desplegar Grafana e integrarlo con Prometheus

Primero se instaló un stack de Prometheus con Grafana usando Helm.
```
azureuser@paragon:~$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
"prometheus-community" has been added to your repositories

azureuser@paragon:~$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "prometheus-community" chart repository
Update Complete. ⎈Happy Helming!⎈

azureuser@paragon:~$ helm install my-kube-prometheus-stack prometheus-community/kube-prometheus-stack
NAME: my-kube-prometheus-stack
LAST DEPLOYED: Mon May 27 06:34:30 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace default get pods -l "release=my-kube-prometheus-stack"

Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.

azureuser@paragon:~$ kubectl get all
NAME                                                               READY   STATUS    RESTARTS   AGE
pod/alertmanager-my-kube-prometheus-stack-alertmanager-0           2/2     Running   0          76s
pod/my-kube-prometheus-stack-grafana-5457d9fc5c-t99tt              3/3     Running   0          88s
pod/my-kube-prometheus-stack-kube-state-metrics-7d5bd9f96b-h48s9   1/1     Running   0          88s
pod/my-kube-prometheus-stack-operator-67bcb86ccb-rn5mw             1/1     Running   0          88s
pod/my-kube-prometheus-stack-prometheus-node-exporter-jrktr        1/1     Running   0          88s
pod/nginx-6bff48bf56-rzz88                                         2/2     Running   0          5m22s
pod/prometheus-my-kube-prometheus-stack-prometheus-0               2/2     Running   0          75s

NAME                                                        TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                       AGE
service/alertmanager-operated                               ClusterIP      None             <none>        9093/TCP,9094/TCP,9094/UDP    76s
service/kubernetes                                          ClusterIP      10.96.0.1        <none>        443/TCP                       30h
service/my-kube-prometheus-stack-alertmanager               ClusterIP      10.104.0.37      <none>        9093/TCP,8080/TCP             88s
service/my-kube-prometheus-stack-grafana                    ClusterIP      10.106.2.48      <none>        80/TCP                        88s
service/my-kube-prometheus-stack-kube-state-metrics         ClusterIP      10.107.210.173   <none>        8080/TCP                      88s
service/my-kube-prometheus-stack-operator                   ClusterIP      10.99.96.58      <none>        443/TCP                       88s
service/my-kube-prometheus-stack-prometheus                 ClusterIP      10.101.97.152    <none>        9090/TCP,8080/TCP             88s
service/my-kube-prometheus-stack-prometheus-node-exporter   ClusterIP      10.107.148.221   <none>        9100/TCP                      88s
service/nginx                                               LoadBalancer   10.111.250.140   <pending>     80:32149/TCP,9113:31623/TCP   5m22s
service/prometheus-operated                                 ClusterIP      None             <none>        9090/TCP                      75s

NAME                                                               DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/my-kube-prometheus-stack-prometheus-node-exporter   1         1         1       1            1           kubernetes.io/os=linux   88s

NAME                                                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-kube-prometheus-stack-grafana              1/1     1            1           88s
deployment.apps/my-kube-prometheus-stack-kube-state-metrics   1/1     1            1           88s
deployment.apps/my-kube-prometheus-stack-operator             1/1     1            1           88s
deployment.apps/nginx                                         1/1     1            1           5m22s

NAME                                                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/my-kube-prometheus-stack-grafana-5457d9fc5c              1         1         1       88s
replicaset.apps/my-kube-prometheus-stack-kube-state-metrics-7d5bd9f96b   1         1         1       88s
replicaset.apps/my-kube-prometheus-stack-operator-67bcb86ccb             1         1         1       88s
replicaset.apps/nginx-6bff48bf56                                         1         1         1       5m22s

NAME                                                                  READY   AGE
statefulset.apps/alertmanager-my-kube-prometheus-stack-alertmanager   1/1     76s
statefulset.apps/prometheus-my-kube-prometheus-stack-prometheus       1/1     75s

```

Se creó el archivo myservicemonitor.yaml que se aplicó para crear el service monitor

```
azureuser@paragon:~$ more myservicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nginx-ingress-sm
  labels:
    app: nginx-ingress-sm
    release: my-kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app.kubernetes.io/instance: nginx
      app.kubernetes.io/name: nginx
  endpoints:
  -  port: metrics
     interval: 5s


azureuser@paragon:~$ kubectl apply -f myservicemonitor.yaml
```

Se publica el puerto 9090 de Prometheus. Luego se usa un navegador y se valida que la web de Prometheus muestra que el target de nginx fue creado exitosamente y permite monitorear las metrics de nginx.

```
azureuser@paragon:~$ kubectl port-forward service/my-kube-prometheus-stack-prometheus 9090 --address="0.0.0.0"
Forwarding from 0.0.0.0:9090 -> 9090
Handling connection for 9090
```
