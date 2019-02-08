# HPA autoscaling using custom-metrics walthrough on AWS EKS cluster


## Make sure you have Helm installed if not, you can install it using following steps.

```
wget https://storage.googleapis.com/kubernetes-helm/helm-v2.12.3-linux-amd64.tar.gz
tar xzvzf helm-v2.12.3-linux-amd64.tar.gz --strip-components=1 -C /usr/local/bin/
```

```
kubectl -n kube-system create serviceaccount tiller

kubectl create clusterrolebinding tiller \
  --clusterrole cluster-admin \
  --serviceaccount=kube-system:tiller

helm init --service-account tiller
```

```
kubectl -n kube-system  rollout status deploy/tiller-deploy
Waiting for deployment "tiller-deploy" rollout to finish: 0 of 1 updated replicas are available...
deployment "tiller-deploy" successfully rolled out
```

```
helm version
Client: &version.Version{SemVer:"v2.12.3", GitCommit:"eecf22f77df5f65c823aacd2dbd30ae6c65f186e", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.12.3", GitCommit:"eecf22f77df5f65c823aacd2dbd30ae6c65f186e", GitTreeState:"clean"}
```

## Prometheus

```
helm install \
  --name mon \
  --namespace monitoring \
  stable/prometheus-operator \
  --set prometheus.prometheusSpec.serviceMonitorNamespaceSelector.matchNames[0]=default
```

## k8s prom adapter https://github.com/helm/charts/tree/master/stable/prometheus-adapter

```
helm install --name prometheus-adapter stable/prometheus-adapter --set prometheus.url="http://mon-prometheus-operator-prometheus.monitoring.svc",prometheus.port="9090" --set image.tag="v0.4.1" --set rbac.create="true" --namespace kube-system 
```

```
kubectl -n kube-system rollout status deploy/prometheus-adapter
Waiting for rollout to finish: 0 of 1 updated replicas are available...
deployment "prometheus-adapter" successfully rolled out
```


## Test custom metrics endpoint

```
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1                                                                                                                                                                                                                                            
{"kind":"APIResourceList","apiVersion":"v1","groupVersion":"custom.metrics.k8s.io/v1beta1","resources":[]}
```

## Launch app with endpoint

Deploy using `kubectl create -f`
```
---
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: example-deploy
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: example
        release: mon
    spec:
      containers:
      - name: example
        image: quay.io/brancz/prometheus-example-app:v0.1.0
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: example-service
  labels:
    release: mon
    app: example
spec:
  ports:
  - name: metrics-svc-port
    port: 80
    targetPort: 8080
  selector:
    app: example
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-sm
  namespace: monitoring
  labels:
    release: mon
    app: example
spec:
  jobLabel: examplemetrics
  selector:
    matchLabels:
      release: mon
      app: example
  namespaceSelector:
    matchNames:
    - default
  endpoints:
  - port: metrics-svc-port
    interval: 10s
    path: /metrics
---
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: example-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1beta1
    kind: Deployment
    name: example-deploy
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Object
    object:
      target:
        kind: Service
        name: example-service
      metricName: http_requests
      targetValue: 100
```

```
kubectl rollout status deploy/example-deploy
deployment "example-deploy" successfully rolled out
```
 
Generate load:
```
kubectl run -i --tty load-generator --image=busybox /bin/sh

Hit enter for command prompt

while true; do wget -q -O- http://example-service.default.svc.cluster.local; done
```

Open a 2nd session to watch HPA scale:
```
kubectl get hpa -w                                                                                                                                                                                                                                                       [34/967]
NAME              REFERENCE                   TARGETS         MINPODS   MAXPODS   REPLICAS   AGE                                                                                                                                                                                                              
example-app-hpa   Deployment/example-deploy   <unknown>/100   1         10        1          33s                                                                                                                                                                                                              
example-app-hpa   Deployment/example-deploy   <unknown>/100   1     10    1     60s
example-app-hpa   Deployment/example-deploy   2878m/100   1     10    1     4m                                                                                                                                                                                                                                
example-app-hpa   Deployment/example-deploy   23403m/100   1     10    1     4m30s
example-app-hpa   Deployment/example-deploy   40583m/100   1     10    1     5m                                                                                                                                                                                                                           
example-app-hpa   Deployment/example-deploy   51422m/100   1     10    1     5m30s                                                                                                                                                                                                                            
example-app-hpa   Deployment/example-deploy   107006m/100   1     10    1     6m
example-app-hpa   Deployment/example-deploy   138397m/100   1         10        2          7m
```
  
Test metric `http_requests`
```
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests"
```
 
## Troubleshooting

`ServiceMonitor` should be listed in Targets with valid endpoints, proper labels and no errors

```
 kubectl port-forward svc/mon-prometheus-operator-prometheus 9090:9090 --namespace monitoring
```

Open http://localhost:9090/targets

No `ServiceMonitor`?: Probably wrong `serviceMonitorNamespaceSelector` in `kubectl -n monitoring edit prometheus` (it only checks its own namespace by default)
No `Endpoints`?: Probably wrong labels, they have to match between components to be selected
