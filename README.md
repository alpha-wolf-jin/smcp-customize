# smcp-customize

**First time update GitHub**
```
echo "# smcp-customize" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/alpha-wolf-jin/smcp-customize.git
git config --global credential.helper 'cache --timeout 7200'
git push -u origin main


git add . ; git commit -a -m "update README" ; git push -u origin main
```

## Deploying extra ingress/egress gateways using ServiceMeshControlPlane

Template for SMCP 
- https://docs.openshift.com/container-platform/4.10/service_mesh/v2x/ossm-reference-smcp.html

Deploying extra ingress/egress gateways using ServiceMeshControlPlane resource Reference: 
- https://access.redhat.com/articles/6619501

Configure custom certificate for the Service Mesh ingress gateway in RHOCP 4 Reference:
- https://access.redhat.com/solutions/6650301

### SMCP Template

```
[root@localhost aro08]# cat service-mesh-02.yaml
apiVersion: maistra.io/v2
kind: ServiceMeshControlPlane
metadata:
  name: basic
spec:
  proxy:
    runtime:
      container:
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 128Mi
  tracing:
    type: Jaeger
  gateways:
    ingress: 
      service:
        type: ClusterIP
        ports:
        - name: status-port
          port: 15020
        - name: http2
          port: 80
          targetPort: 8080
        - name: https
          port: 443
          targetPort: 8443
      meshExpansionPorts: []
    egress: 
      service:
        type: ClusterIP
        ports:
        - name: status-port
          port: 15020
        - name: http2
          port: 80
          targetPort: 8080
        - name: https
          port: 443
          targetPort: 8443
    additionalIngress:
      ui-ingressgateway:
        enabled: true
        runtime:
          deployment:
            autoScaling:
              enabled: false
            replicas: 1
          container:
            resources:
              requests:
                cpu: 10m
                memory: 50Mi
              limits:
                cpu: 100m
                memory: 200Mi
          pod:
            metadata:
             labels:
               app: ui-ingressgateway
        service: 
          type: LoadBalancer
      http-ingressgateway:
        enabled: true
        runtime:
          deployment:
            autoScaling:
              enabled: false
            replicas: 1
          container:
            resources:
              requests:
                cpu: 10m
                memory: 50Mi
              limits:
                cpu: 100m
                memory: 200Mi
          pod:
            metadata:
             labels:
               app: http-ingressgateway
        service: 
          type: LoadBalancer

    additionalEgress:
      ui-egressgateway:
        enabled: true
        runtime:
          deployment:
            autoScaling:
              enabled: false
            replicas: 1
          container:
            resources:
              requests:
                cpu: 10m
                memory: 50Mi
              limits:
                cpu: 100m
                memory: 200Mi
          pod:
            metadata:
             labels:
               app: ui-egressgateway
        service: {}
      http-egressgateway:
        enabled: true
        runtime:
          deployment:
            autoScaling:
              enabled: false
            replicas: 1
          container:
            resources:
              requests:
                cpu: 10m
                memory: 50Mi
              limits:
                cpu: 100m
                memory: 200Mi
          pod:
            metadata:
             labels:
               app: http-egressgateway
        service: {}

  policy: {}

  telemetry:
    type:  Istiod # Istiod or Mixer
  addons:
    grafana:
      enabled: true
      install:
        config:
          env: {}
          envSecrets: {}
        persistence:
          enabled: true
          storageClassName: managed-premium
          accessMode: ReadWriteOnce
          capacity:
            requests:
              storage: 5Gi
        service:
          ingress:
            contextPath: /grafana
            tls:
              termination: reencrypt
    kiali:
      name: kiali
      enabled: true
      install: # install kiali CR if not present
        dashboard:
          viewOnly: false
          enableGrafana: true
          enableTracing: true
          enablePrometheus: true
      service:
        ingress:
          contextPath: /kiali
    jaeger:
      name: jaeger
      install:
        storage:
          type: Elasticsearch # or Memory
          memory:
            maxTraces: 100
          elasticsearch:
            nodeCount: 3
            storage: {}
            redundancyPolicy: SingleRedundancy
            indexCleaner: {}
        ingress: {} # jaeger ingress configuration
  runtime:
    components:
      pilot:
        deployment:
          replicas: 2
        pod:
          affinity: {}
        container:
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 128Mi
      grafana:
        deployment: {}
        pod: {}
      kiali:
        deployment: {}
        pod: {}
```
> Remove Jaege Elasticsearch setting due to insuffecient resource

### Deploy SMCP

```
[root@localhost aro08]# oc apply -f service-mesh-02.yaml

[root@localhost aro08]# oc get smcp
NAME    READY   STATUS            PROFILES      VERSION   AGE
basic   10/10   ComponentsReady   ["default"]   2.1.2     76s

```

### Ingress & Egress Pods

```
# oc get pod
NAME                                                     READY   STATUS    RESTARTS      AGE
elasticsearch-cdm-istiosystemjaeger-1-76478b4c-7c9sz     2/2     Running   0             22m
elasticsearch-cdm-istiosystemjaeger-2-65768cf499-m4gv7   2/2     Running   0             22m
elasticsearch-cdm-istiosystemjaeger-3-6d479dc-2q9fz      2/2     Running   0             22m
grafana-99d6457f-2zpzq                                   2/2     Running   0             19m
http-egressgateway-544f75d54d-wjr4q                      1/1     Running   0             22m
http-ingressgateway-7b69c76759-9cmns                     1/1     Running   0             22m
istio-egressgateway-7fd685d798-9k7lf                     1/1     Running   0             22m
istio-ingressgateway-79bf956b68-9cztf                    1/1     Running   0             22m
istiod-basic-6867796997-bxwc8                            1/1     Running   0             22m
istiod-basic-6867796997-wbg6d                            1/1     Running   0             22m
jaeger-collector-5498fff9c9-x98jr                        1/1     Running   1 (20m ago)   21m
jaeger-query-5775b96574-6929f                            3/3     Running   1 (20m ago)   20m
kiali-74c78868df-z75bd                                   1/1     Running   0             6m28s
prometheus-69cd5746f-x6kxz                               2/2     Running   0             22m
ui-egressgateway-84dc658d4d-hh2kg                        1/1     Running   0             22m
ui-ingressgateway-5bb969c64c-s76vz                       1/1     Running   0             22m
wasm-cacher-basic-567568d75-wrgp8                        1/1     Running   0             7m


[root@localhost aro08]# oc get pod | grep -i gress
http-egressgateway-544f75d54d-dp5qt     1/1     Running   0          2m33s
http-ingressgateway-7b69c76759-56xzs    1/1     Running   0          2m33s
istio-egressgateway-7fd685d798-2dnk8    1/1     Running   0          2m33s
istio-ingressgateway-79bf956b68-dd8nm   1/1     Running   0          2m34s
ui-egressgateway-84dc658d4d-fdc5s       1/1     Running   0          2m33s
ui-ingressgateway-5bb969c64c-br66p      1/1     Running   0          2m33s

```
> Note: the cutomize ingress & egress have the dedicated pods now.
- ui-ingressgateway-xxx
- http-ingressgateway-xxx
- ui-egressgateway-xxx
- http-egressgateway-xxx



### SVC

```
[root@localhost aro08]# oc get svc
NAME                        TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                                                      AGE
elasticsearch               ClusterIP      172.30.106.79    <none>         9200/TCP                                                     26m
elasticsearch-cluster       ClusterIP      172.30.93.107    <none>         9300/TCP                                                     26m
elasticsearch-metrics       ClusterIP      172.30.253.251   <none>         60001/TCP                                                    26m
grafana                     ClusterIP      172.30.111.72    <none>         3000/TCP                                                     26m
http-egressgateway          ClusterIP      172.30.230.101   <none>         80/TCP,443/TCP,15443/TCP                                     26m
http-ingressgateway         LoadBalancer   172.30.47.117    20.237.1.125   15021:30792/TCP,80:30619/TCP,443:30027/TCP,15443:32729/TCP   26m
istio-egressgateway         ClusterIP      172.30.6.13      <none>         15020/TCP,80/TCP,443/TCP                                     26m
istio-ingressgateway        ClusterIP      172.30.194.106   <none>         15020/TCP,80/TCP,443/TCP                                     26m
istiod-basic                ClusterIP      172.30.11.19     <none>         15010/TCP,15012/TCP,443/TCP,15014/TCP,8188/TCP               26m
jaeger-collector            ClusterIP      172.30.233.114   <none>         9411/TCP,14250/TCP,14267/TCP,14268/TCP                       25m
jaeger-collector-headless   ClusterIP      None             <none>         9411/TCP,14250/TCP,14267/TCP,14268/TCP                       25m
jaeger-query                ClusterIP      172.30.213.227   <none>         443/TCP,16685/TCP                                            25m
kiali                       ClusterIP      172.30.246.6     <none>         20001/TCP,9090/TCP                                           10m
prometheus                  ClusterIP      172.30.64.14     <none>         9090/TCP                                                     26m
ui-egressgateway            ClusterIP      172.30.153.241   <none>         80/TCP,443/TCP,15443/TCP                                     26m
ui-ingressgateway           LoadBalancer   172.30.217.183   20.237.1.223   15021:30518/TCP,80:32580/TCP,443:32691/TCP,15443:32000/TCP   26m
wasm-cacher-basic           ClusterIP      172.30.75.8      <none>         80/TCP                                                       11m
zipkin                      ClusterIP      172.30.180.144   <none>         9411/TCP       

```

### all resources

```
[root@localhost aro08]# oc get all,ep
NAME                                                         READY   STATUS    RESTARTS      AGE
pod/elasticsearch-cdm-istiosystemjaeger-1-76478b4c-7c9sz     2/2     Running   0             27m
pod/elasticsearch-cdm-istiosystemjaeger-2-65768cf499-m4gv7   2/2     Running   0             27m
pod/elasticsearch-cdm-istiosystemjaeger-3-6d479dc-2q9fz      2/2     Running   0             27m
pod/grafana-99d6457f-2zpzq                                   2/2     Running   0             24m
pod/http-egressgateway-544f75d54d-wjr4q                      1/1     Running   0             27m
pod/http-ingressgateway-7b69c76759-9cmns                     1/1     Running   0             27m
pod/istio-egressgateway-7fd685d798-9k7lf                     1/1     Running   0             27m
pod/istio-ingressgateway-79bf956b68-9cztf                    1/1     Running   0             27m
pod/istiod-basic-6867796997-bxwc8                            1/1     Running   0             27m
pod/istiod-basic-6867796997-wbg6d                            1/1     Running   0             27m
pod/jaeger-collector-5498fff9c9-x98jr                        1/1     Running   1 (25m ago)   26m
pod/jaeger-query-5775b96574-6929f                            3/3     Running   1 (25m ago)   25m
pod/kiali-74c78868df-z75bd                                   1/1     Running   0             11m
pod/prometheus-69cd5746f-x6kxz                               2/2     Running   0             27m
pod/ui-egressgateway-84dc658d4d-hh2kg                        1/1     Running   0             27m
pod/ui-ingressgateway-5bb969c64c-s76vz                       1/1     Running   0             27m
pod/wasm-cacher-basic-567568d75-wrgp8                        1/1     Running   0             11m

NAME                                TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                                                      AGE
service/elasticsearch               ClusterIP      172.30.106.79    <none>         9200/TCP                                                     27m
service/elasticsearch-cluster       ClusterIP      172.30.93.107    <none>         9300/TCP                                                     27m
service/elasticsearch-metrics       ClusterIP      172.30.253.251   <none>         60001/TCP                                                    27m
service/grafana                     ClusterIP      172.30.111.72    <none>         3000/TCP                                                     27m
service/http-egressgateway          ClusterIP      172.30.230.101   <none>         80/TCP,443/TCP,15443/TCP                                     27m
service/http-ingressgateway         LoadBalancer   172.30.47.117    20.237.1.125   15021:30792/TCP,80:30619/TCP,443:30027/TCP,15443:32729/TCP   27m
service/istio-egressgateway         ClusterIP      172.30.6.13      <none>         15020/TCP,80/TCP,443/TCP                                     27m
service/istio-ingressgateway        ClusterIP      172.30.194.106   <none>         15020/TCP,80/TCP,443/TCP                                     27m
service/istiod-basic                ClusterIP      172.30.11.19     <none>         15010/TCP,15012/TCP,443/TCP,15014/TCP,8188/TCP               27m
service/jaeger-collector            ClusterIP      172.30.233.114   <none>         9411/TCP,14250/TCP,14267/TCP,14268/TCP                       26m
service/jaeger-collector-headless   ClusterIP      None             <none>         9411/TCP,14250/TCP,14267/TCP,14268/TCP                       26m
service/jaeger-query                ClusterIP      172.30.213.227   <none>         443/TCP,16685/TCP                                            26m
service/kiali                       ClusterIP      172.30.246.6     <none>         20001/TCP,9090/TCP                                           11m
service/prometheus                  ClusterIP      172.30.64.14     <none>         9090/TCP                                                     27m
service/ui-egressgateway            ClusterIP      172.30.153.241   <none>         80/TCP,443/TCP,15443/TCP                                     27m
service/ui-ingressgateway           LoadBalancer   172.30.217.183   20.237.1.223   15021:30518/TCP,80:32580/TCP,443:32691/TCP,15443:32000/TCP   27m
service/wasm-cacher-basic           ClusterIP      172.30.75.8      <none>         80/TCP                                                       11m
service/zipkin                      ClusterIP      172.30.180.144   <none>         9411/TCP                                                     27m

NAME                                                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/elasticsearch-cdm-istiosystemjaeger-1   1/1     1            1           27m
deployment.apps/elasticsearch-cdm-istiosystemjaeger-2   1/1     1            1           27m
deployment.apps/elasticsearch-cdm-istiosystemjaeger-3   1/1     1            1           27m
deployment.apps/grafana                                 1/1     1            1           27m
deployment.apps/http-egressgateway                      1/1     1            1           27m
deployment.apps/http-ingressgateway                     1/1     1            1           27m
deployment.apps/istio-egressgateway                     1/1     1            1           27m
deployment.apps/istio-ingressgateway                    1/1     1            1           27m
deployment.apps/istiod-basic                            2/2     2            2           27m
deployment.apps/jaeger-collector                        1/1     1            1           26m
deployment.apps/jaeger-query                            1/1     1            1           26m
deployment.apps/kiali                                   1/1     1            1           11m
deployment.apps/prometheus                              1/1     1            1           27m
deployment.apps/ui-egressgateway                        1/1     1            1           27m
deployment.apps/ui-ingressgateway                       1/1     1            1           27m
deployment.apps/wasm-cacher-basic                       1/1     1            1           11m

NAME                                                               DESIRED   CURRENT   READY   AGE
replicaset.apps/elasticsearch-cdm-istiosystemjaeger-1-76478b4c     1         1         1       27m
replicaset.apps/elasticsearch-cdm-istiosystemjaeger-2-65768cf499   1         1         1       27m
replicaset.apps/elasticsearch-cdm-istiosystemjaeger-3-6d479dc      1         1         1       27m
replicaset.apps/grafana-99d6457f                                   1         1         1       27m
replicaset.apps/http-egressgateway-544f75d54d                      1         1         1       27m
replicaset.apps/http-ingressgateway-7b69c76759                     1         1         1       27m
replicaset.apps/istio-egressgateway-7fd685d798                     1         1         1       27m
replicaset.apps/istio-ingressgateway-79bf956b68                    1         1         1       27m
replicaset.apps/istiod-basic-6867796997                            2         2         2       27m
replicaset.apps/jaeger-collector-5498fff9c9                        1         1         1       26m
replicaset.apps/jaeger-query-5775b96574                            1         1         1       25m
replicaset.apps/jaeger-query-645ff86884                            0         0         0       26m
replicaset.apps/kiali-74c78868df                                   1         1         1       11m
replicaset.apps/prometheus-69cd5746f                               1         1         1       27m
replicaset.apps/ui-egressgateway-84dc658d4d                        1         1         1       27m
replicaset.apps/ui-ingressgateway-5bb969c64c                       1         1         1       27m
replicaset.apps/wasm-cacher-basic-567568d75                        1         1         1       11m

NAME                                                   REFERENCE                     TARGETS            MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/jaeger-collector   Deployment/jaeger-collector   12%/90%, 10%/90%   1         100       1          26m

NAME                                    SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/jaeger-es-index-cleaner   55 23 * * *   False     0        <none>          26m

NAME                                            HOST/PORT                                                        PATH   SERVICES               PORT          TERMINATION          WILDCARD
route.route.openshift.io/grafana                grafana-istio-system.apps.aro.example.opentlc.com                       grafana                <all>         reencrypt/Redirect   None
route.route.openshift.io/http-ingressgateway    http-ingressgateway-istio-system.apps.aro.example.opentlc.com           http-ingressgateway    8080                               None
route.route.openshift.io/istio-ingressgateway   istio-ingressgateway-istio-system.apps.aro.example.opentlc.com          istio-ingressgateway   8080                               None
route.route.openshift.io/jaeger                 jaeger-istio-system.apps.aro.example.opentlc.com                        jaeger-query           https-query   reencrypt            None
route.route.openshift.io/kiali                  kiali-istio-system.apps.aro.example.opentlc.com                         kiali                  20001         reencrypt/Redirect   None
route.route.openshift.io/prometheus             prometheus-istio-system.apps.aro.example.opentlc.com                    prometheus             <all>         reencrypt/Redirect   None
route.route.openshift.io/ui-ingressgateway      ui-ingressgateway-istio-system.apps.aro.example.opentlc.com             ui-ingressgateway      8080                               None

NAME                                  ENDPOINTS                                                          AGE
endpoints/elasticsearch               10.128.2.23:60000,10.129.2.18:60000,10.131.0.21:60000              27m
endpoints/elasticsearch-cluster       10.128.2.23:9300,10.129.2.18:9300,10.131.0.21:9300                 27m
endpoints/elasticsearch-metrics       10.128.2.23:60001,10.129.2.18:60001,10.131.0.21:60001              27m
endpoints/grafana                     10.128.2.24:3001                                                   27m
endpoints/http-egressgateway          10.131.0.19:15443,10.131.0.19:8080,10.131.0.19:8443                27m
endpoints/http-ingressgateway         10.131.0.16:15443,10.131.0.16:15021,10.131.0.16:8080 + 1 more...   27m
endpoints/istio-egressgateway         10.131.0.18:15020,10.131.0.18:8080,10.131.0.18:8443                27m
endpoints/istio-ingressgateway        10.131.0.15:15020,10.131.0.15:8080,10.131.0.15:8443                27m
endpoints/istiod-basic                10.128.2.22:8188,10.129.2.17:8188,10.128.2.22:15012 + 7 more...    27m
endpoints/jaeger-collector            10.131.0.23:14268,10.131.0.23:14250,10.131.0.23:9411 + 1 more...   26m
endpoints/jaeger-collector-headless   10.131.0.23:14268,10.131.0.23:14250,10.131.0.23:9411 + 1 more...   26m
endpoints/jaeger-query                10.131.0.25:16685,10.131.0.25:8443                                 26m
endpoints/kiali                       10.129.2.22:9090,10.129.2.22:20001                                 11m
endpoints/prometheus                  10.131.0.14:3001                                                   27m
endpoints/ui-egressgateway            10.131.0.20:15443,10.131.0.20:8080,10.131.0.20:8443                27m
endpoints/ui-ingressgateway           10.131.0.17:15443,10.131.0.17:15021,10.131.0.17:8080 + 1 more...   27m
endpoints/wasm-cacher-basic           10.129.2.21:8080                                                   11m
endpoints/zipkin                      10.131.0.23:9411                                                   27m
[root@localhost aro08]# 

```

### Mapping SVC and POD

```
[root@localhost aro08]# oc get svc ui-ingressgateway -o yaml
apiVersion: v1
kind: Service
metadata:
  ...
  labels:
    app: ui-ingressgateway
    app.kubernetes.io/component: istio-ingress
    ...
    istio: ingressgateway
    ...
  ...
  name: ui-ingressgateway
  namespace: istio-system
  ...
spec:
  ...
  clusterIPs:
  - 172.30.134.116
  ...
  selector:
    app: ui-ingressgateway
    istio: ingressgateway
  ...
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 20.237.32.49

[root@localhost aro08]# oc get pod  -l app=ui-ingressgateway,istio=ingressgateway
NAME                                 READY   STATUS    RESTARTS   AGE
ui-ingressgateway-5bb969c64c-br66p   1/1     Running   0          29m

[root@localhost aro08]# oc get pod  -l app=http-ingressgateway,istio=ingressgateway
NAME                                   READY   STATUS    RESTARTS   AGE
http-ingressgateway-7b69c76759-56xzs   1/1     Running   0          29m

[root@localhost aro08]# oc get pod  -l app=ui-egressgateway,istio=egressgateway
NAME                                READY   STATUS    RESTARTS   AGE
ui-egressgateway-84dc658d4d-fdc5s   1/1     Running   0          29m

[root@localhost aro08]# oc get pod  -l app=http-egressgateway,istio=egressgateway
NAME                                  READY   STATUS    RESTARTS   AGE
http-egressgateway-544f75d54d-dp5qt   1/1     Running   0          29m

```

### Mapping Ingress & Egress resources

```

[root@localhost aro08]# oc get all  -l app=ui-ingressgateway,istio=ingressgateway
NAME                                     READY   STATUS    RESTARTS   AGE
pod/ui-ingressgateway-5bb969c64c-br66p   1/1     Running   0          32m

NAME                        TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                                                      AGE
service/ui-ingressgateway   LoadBalancer   172.30.134.116   20.237.32.49   15021:32675/TCP,80:30623/TCP,443:30481/TCP,15443:31772/TCP   32m

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ui-ingressgateway   1/1     1            1           32m

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/ui-ingressgateway-5bb969c64c   1         1         1       32m

NAME                                         HOST/PORT                                                     PATH   SERVICES            PORT   TERMINATION   WILDCARD
route.route.openshift.io/ui-ingressgateway   ui-ingressgateway-istio-system.apps.aro.example.opentlc.com          ui-ingressgateway   8080                 None



[root@localhost aro08]# oc get all -l app=ui-egressgateway,istio=egressgateway
NAME                                    READY   STATUS    RESTARTS   AGE
pod/ui-egressgateway-84dc658d4d-fdc5s   1/1     Running   0          34m

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ui-egressgateway   1/1     1            1           34m

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/ui-egressgateway-84dc658d4d   1         1         1       34m

```

## Jaeger Persistent Storage

>elasticsearch redundancyPolicy 
Refer to : https://docs.openshift.com/container-platform/4.7/logging/config/cluster-logging-log-store.html
- FullRedundancy
- MultipleRedundancy
- SingleRedundancy
- ZeroRedundancy
> Select ZeroRedundancy due to tight resource

## Deploy more work node with 


```
[root@localhost aro08]# oc get -o jsonpath='{.status.infrastructureName}{"\n"}' infrastructure cluster
aro-nnkdl

[root@localhost aro08]# export INFRA_ID=aro-nnkdl

[root@localhost aro08]# export RESOURCE_GROUP=openenv-dk7fm

```

# Deploy The Apps

**Below test is done on other cluster**

```
# oc new-project ingress-lb

# echo hello-world-01 >index-01.html

# oc create configmap index-html --from-file=index.html=./index-01.html

# vim deploy-http.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web
    app.kubernetes.io/component: web
    app.kubernetes.io/instance: web
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      deployment: web
  template:
    metadata:
      labels:
        deployment: web
    spec:
      containers:
      - image: registry.redhat.io/rhel8/httpd-24:1-161.1638356842
        name: web
        ports:
        - containerPort: 8080
          protocol: TCP
        - containerPort: 8443
          protocol: TCP
        resources: {}
        volumeMounts:
        - name: index-html
          mountPath: /var/www/html/index.html
          readOnly: true
          subPath: index.html
      volumes:
      - configMap:
          defaultMode: 420
          items:
          - key: index.html
            path: index.html
          name: index-html
        name: index-html


# oc apply -f ./deploy-http.yaml 

# oc get pod
NAME                  READY   STATUS    RESTARTS   AGE
web-75df46b89-t7fsc   1/1     Running   0          27s

# oc get deploy
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web    1/1     1            1           18m

# oc expose deploy web

# oc get svc
NAME   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
web    ClusterIP   172.30.29.58   <none>        8080/TCP,8443/TCP   20s

# oc expose svc web

# oc get route
NAME   HOST/PORT                                       PATH   SERVICES   PORT     TERMINATION   WILDCARD
web    web-ingress-lb.apps.ipm9b8a3.eastus.aroapp.io          web        port-1                 None

# curl web-ingress-lb.apps.ipm9b8a3.eastus.aroapp.io
hello-world-01

# oc delete route web


```

# Setup Service Mesh

```
# oc project istio-system

# oc get svc -l istio=ingressgateway
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                                                      AGE
http-ingressgateway    LoadBalancer   172.30.188.252   20.121.183.22    15021:31356/TCP,80:30576/TCP,443:30806/TCP,15443:30850/TCP   79m
istio-ingressgateway   ClusterIP      172.30.110.3     <none>           15020/TCP,80/TCP,443/TCP                                     79m
ui-ingressgateway      LoadBalancer   172.30.22.192    20.121.183.126   15021:32540/TCP,80:30093/TCP,443:31261/TCP,15443:31850/TCP   79m


# oc get svc -l app=http-ingressgateway,istio=ingressgateway -n istio-system
NAME                  TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                                                      AGE
http-ingressgateway   LoadBalancer   172.30.47.117   20.237.1.125   15021:30792/TCP,80:30619/TCP,443:30027/TCP,15443:32729/TCP   5h30m

# oc get svc -l app=ui-ingressgateway,istio=ingressgateway -n istio-system
NAME                TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                                                      AGE
ui-ingressgateway   LoadBalancer   172.30.217.183   20.237.1.223   15021:30518/TCP,80:32580/TCP,443:32691/TCP,15443:32000/TCP   5h30m


# vim service-mesh-roll.yaml
apiVersion: maistra.io/v1
kind: ServiceMeshMemberRoll
metadata:
  name: default
spec:
  members:
  - ingress-lb

# oc apply -f service-mesh-roll.yaml

# oc project ingress-lb

# vim deploy-http.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web
    app.kubernetes.io/component: web
    app.kubernetes.io/instance: web
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      deployment: web
  template:
    metadata:
      annotations:
        sidecar.istio.io/inject: "true"
      labels:
        deployment: web
...

# oc apply -f  deploy-http.yaml

# cat service-mesh-roll.yaml
apiVersion: maistra.io/v1
kind: ServiceMeshMemberRoll
metadata:
  name: default
  namespace: istio-system
spec:
  members:
  - ingress-lb

# oc apply -f service-mesh-roll.yaml

# oc get pod
NAME                   READY   STATUS    RESTARTS   AGE
web-866dd6f769-5878r   1/1     Running   0          3m1s

# oc delete pod web-866dd6f769-5878r

# oc get pod
NAME                   READY   STATUS    RESTARTS   AGE
web-866dd6f769-vmpmw   2/2     Running   0          58s

```
> 1. project is added into ServiceMeshMemberRoll
> 2. annotations (sidecar.istio.io/inject: "true") is added into deployment conf
> 3. When first 2 are configured, the sidcar container will be injected into new POD automatically during provison. If there is no new provision, the injection will not happen.


```

[root@localhost aro08]# cat service-mesh-gw.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: http-ingress-gateway
spec:
  selector:
    app: http-ingressgateway
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - '*'
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: http-ingress-gateway
spec:
  hosts:
  - '*'
  gateways:
  - http-ingress-gateway
  http:
  - match:
    - uri:
        exact: /
    route:
    - destination:
        host: web
        port:
          number: 8080

# oc apply -f service-mesh-gw.yaml -n ingress-lb

# oc get gateway
NAME                   AGE
http-ingress-gateway   27s

# oc get VirtualService
NAME                   GATEWAYS                   HOSTS   AGE
http-ingress-gateway   ["http-ingress-gateway"]   ["*"]   40s

# oc get svc
NAME   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
web    ClusterIP   172.30.26.118   <none>        8080/TCP,8443/TCP   19m

# oc get svc -l app=http-ingressgateway,istio=ingressgateway -n istio-system
NAME                  TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                                                      AGE
http-ingressgateway   LoadBalancer   172.30.47.117   20.237.1.125   15021:30792/TCP,80:30619/TCP,443:30027/TCP,15443:32729/TCP   5h30m

# curl http://20.237.1.125
hello-world-01

# oc get svc -l app=ui-ingressgateway,istio=ingressgateway -n istio-system
NAME                TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                                                      AGE
ui-ingressgateway   LoadBalancer   172.30.217.183   20.237.1.223   15021:30518/TCP,80:32580/TCP,443:32691/TCP,15443:32000/TCP   5h30m

# Note: the failure is expected
# curl http://20.237.1.223
curl: (7) Failed to connect to 20.237.1.223 port 80: Connection timed out

# vi /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

20.237.1.125   websingtel.example.com

# curl http://websingtel.example.com
hello-world-01

```

# Use the FQDN

```
# oc edit gw http-ingress-gateway
spec:
  selector:
    app: http-ingressgateway
    istio: ingressgateway
  servers:
  - hosts:
    - 'websingtel.example.com'
    port:
      name: http
      number: 80
      protocol: HTTP

# oc edit VirtualService http-ingress-gateway
spec:
  gateways:
  - http-ingress-gateway
  hosts:
  - 'websingtel.example.com'
  http:
  - match:
    - uri:
        exact: /
    route:
    - destination:
        host: web
        port:
          number: 8080

# curl http://websingtel.example.com
hello-world-01

# curl http://20.237.1.125
# 
# Now only specific FQDN works

```

# Configure custom certificate for the Service Mesh ingress gateway

https://access.redhat.com/solutions/6650301

## Self-signed certificates and key generation

### Create a root CA certificate and private key to sign the certificates for your services

```
# openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt

```

### Create a certificate and a private key for *.example.com

```
# openssl req -out httpbin.example.com.csr -newkey rsa:2048 -nodes -keyout httpbin.example.com.key -subj "/CN=*.example.com/O=httpbin organization"

# openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in httpbin.example.com.csr -out httpbin.example.com.crt

# ls -artl
...
-rw-------.  1 root root    1704 May 22 01:02 example.com.key
-rw-r--r--.  1 root root    1180 May 22 01:02 example.com.crt
-rw-------.  1 root root    1704 May 22 01:03 httpbin.example.com.key
-rw-r--r--.  1 root root     940 May 22 01:03 httpbin.example.com.csr
-rw-r--r--.  1 root root    1046 May 22 01:04 httpbin.example.com.crt

```

### Create a secret for serving the certificate

```
# oc create secret tls webcert --key=httpbin.example.com.key --cert=httpbin.example.com.crt -n istio-system

```

### Make the required configurations to make the traffic reach at istio-ingress gateway

- If it is a cloud environment, use ingressgateway  service as loadbalacer to get externalIP and do the infra level changes by adding A records.
- For on-premise, bare-metal environments, use the NodePort service.

### Review the services in the control plane namespace

```
# oc get svc -l app=http-ingressgateway,istio=ingressgateway -n istio-system
NAME                  TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                                                      AGE
http-ingressgateway   LoadBalancer   172.30.47.117   20.237.1.125   15021:30792/TCP,80:30619/TCP,443:30027/TCP,15443:32729/TCP   6h9m

```

**After confirming that traffic can reach the ingress gateway, add a gateway rule to allow the traffic to the cluster**

```
# oc project
Using project "ingress-lb" on server "https://api.aro.example.opentlc.com:6443".

# oc get gw
NAME                   AGE
http-ingress-gateway   88m

# oc edit gw http-ingress-gateway
. . .
spec:
  selector:
    app: http-ingressgateway
    istio: ingressgateway
  servers:
  - hosts:
    - websingtel.example.com
    port:
      name: https
      number: 443
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: webcert

# curl https://websingtel.example.com  --cacert example.com.crt
hello-world-01

# curl https://websingtel.example.com  --cacert example.com.crt
hello-world-01
[root@localhost aro08]# curl -kvvI https://websingtel.example.com  --cacert example.com.crt
* Rebuilt URL to: https://websingtel.example.com/
*   Trying 20.237.1.125...
* TCP_NODELAY set
* Connected to websingtel.example.com (20.237.1.125) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: example.com.crt
  CApath: none
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, [no content] (0):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, [no content] (0):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, [no content] (0):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, [no content] (0):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, [no content] (0):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=*.example.com; O=httpbin organization
*  start date: May 21 17:04:22 2022 GMT
*  expire date: May 21 17:04:22 2023 GMT
*  issuer: O=example Inc.; CN=example.com
*  SSL certificate verify ok.
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* TLSv1.3 (OUT), TLS app data, [no content] (0):
* TLSv1.3 (OUT), TLS app data, [no content] (0):
* TLSv1.3 (OUT), TLS app data, [no content] (0):
* Using Stream ID: 1 (easy handle 0x55876c2186b0)
* TLSv1.3 (OUT), TLS app data, [no content] (0):
> HEAD / HTTP/2
> Host: websingtel.example.com
> User-Agent: curl/7.61.1
> Accept: */*
> 
* TLSv1.3 (IN), TLS handshake, [no content] (0):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS handshake, [no content] (0):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS app data, [no content] (0):
* Connection state changed (MAX_CONCURRENT_STREAMS == 2147483647)!
* TLSv1.3 (OUT), TLS app data, [no content] (0):
* TLSv1.3 (IN), TLS app data, [no content] (0):
< HTTP/2 200 
HTTP/2 200 
< date: Sat, 21 May 2022 17:20:49 GMT
date: Sat, 21 May 2022 17:20:49 GMT
< server: istio-envoy
server: istio-envoy
< last-modified: Sat, 21 May 2022 15:38:07 GMT
last-modified: Sat, 21 May 2022 15:38:07 GMT
< etag: "f-5df8764b9c464"
etag: "f-5df8764b9c464"
< accept-ranges: bytes
accept-ranges: bytes
< content-length: 15
content-length: 15
< content-type: text/html; charset=UTF-8
content-type: text/html; charset=UTF-8
< x-envoy-upstream-service-time: 4
x-envoy-upstream-service-time: 4

< 
* Connection #0 to host websingtel.example.com left intact

```
