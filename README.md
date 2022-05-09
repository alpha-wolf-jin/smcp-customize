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
          #    jaeger:
          #      name: jaeger
          #      install:
          #        storage:
          #          type: Elasticsearch # or Memory
          #          memory:
          #            maxTraces: 100000
          #          elasticsearch:
          #            nodeCount: 3
          #            storage: {}
          #            redundancyPolicy: SingleRedundancy
          #            indexCleaner: {}
          #        ingress: {} # jaeger ingress configuration
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
[root@localhost aro08]# oc get pod
NAME                                    READY   STATUS    RESTARTS   AGE
grafana-99d6457f-gbbrm                  2/2     Running   0          71s
http-egressgateway-544f75d54d-dp5qt     1/1     Running   0          72s
http-ingressgateway-7b69c76759-56xzs    1/1     Running   0          72s
istio-egressgateway-7fd685d798-2dnk8    1/1     Running   0          72s
istio-ingressgateway-79bf956b68-dd8nm   1/1     Running   0          73s
istiod-basic-6867796997-nkprr           1/1     Running   0          83s
istiod-basic-6867796997-vrrts           1/1     Running   0          82s
jaeger-8c46c4f58-lc4k7                  2/2     Running   0          73s
prometheus-69cd5746f-xkzfg              2/2     Running   0          77s
ui-egressgateway-84dc658d4d-fdc5s       1/1     Running   0          72s
ui-ingressgateway-5bb969c64c-br66p      1/1     Running   0          72s
wasm-cacher-basic-567568d75-6xs79       1/1     Running   0          51s


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
grafana                     ClusterIP      172.30.224.24    <none>         3000/TCP                                                     3m37s
http-egressgateway          ClusterIP      172.30.154.14    <none>         80/TCP,443/TCP,15443/TCP                                     3m38s
http-ingressgateway         LoadBalancer   172.30.181.240   20.237.32.58   15021:31712/TCP,80:31045/TCP,443:30368/TCP,15443:32633/TCP   3m38s
istio-egressgateway         ClusterIP      172.30.116.82    <none>         15020/TCP,80/TCP,443/TCP                                     3m38s
istio-ingressgateway        ClusterIP      172.30.183.91    <none>         15020/TCP,80/TCP,443/TCP                                     3m39s
istiod-basic                ClusterIP      172.30.71.0      <none>         15010/TCP,15012/TCP,443/TCP,15014/TCP,8188/TCP               3m49s
jaeger-agent                ClusterIP      None             <none>         5775/UDP,5778/TCP,6831/UDP,6832/UDP                          3m39s
jaeger-collector            ClusterIP      172.30.63.55     <none>         9411/TCP,14250/TCP,14267/TCP,14268/TCP                       3m39s
jaeger-collector-headless   ClusterIP      None             <none>         9411/TCP,14250/TCP,14267/TCP,14268/TCP                       3m39s
jaeger-query                ClusterIP      172.30.155.56    <none>         443/TCP,16685/TCP                                            3m39s
prometheus                  ClusterIP      172.30.184.39    <none>         9090/TCP                                                     3m43s
ui-ingressgateway           LoadBalancer   172.30.134.116   20.237.32.49   15021:32675/TCP,80:30623/TCP,443:30481/TCP,15443:31772/TCP   3m38s
wasm-cacher-basic           ClusterIP      172.30.84.78     <none>         80/TCP                                                       3m17s
zipkin                      ClusterIP      172.30.225.80    <none>         9411/TCP                                                     3m39s

```

### all resources

```
[root@localhost aro08]# oc get all,ep
NAME                                        READY   STATUS    RESTARTS   AGE
pod/grafana-99d6457f-gbbrm                  2/2     Running   0          9m8s
pod/http-egressgateway-544f75d54d-dp5qt     1/1     Running   0          9m9s
pod/http-ingressgateway-7b69c76759-56xzs    1/1     Running   0          9m9s
pod/istio-egressgateway-7fd685d798-2dnk8    1/1     Running   0          9m9s
pod/istio-ingressgateway-79bf956b68-dd8nm   1/1     Running   0          9m10s
pod/istiod-basic-6867796997-nkprr           1/1     Running   0          9m20s
pod/istiod-basic-6867796997-vrrts           1/1     Running   0          9m19s
pod/jaeger-8c46c4f58-lc4k7                  2/2     Running   0          9m10s
pod/prometheus-69cd5746f-xkzfg              2/2     Running   0          9m14s
pod/ui-egressgateway-84dc658d4d-fdc5s       1/1     Running   0          9m9s
pod/ui-ingressgateway-5bb969c64c-br66p      1/1     Running   0          9m9s
pod/wasm-cacher-basic-567568d75-6xs79       1/1     Running   0          8m48s

NAME                                TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                                                      AGE
service/grafana                     ClusterIP      172.30.224.24    <none>         3000/TCP                                                     9m8s
service/http-egressgateway          ClusterIP      172.30.154.14    <none>         80/TCP,443/TCP,15443/TCP                                     9m9s
service/http-ingressgateway         LoadBalancer   172.30.181.240   20.237.32.58   15021:31712/TCP,80:31045/TCP,443:30368/TCP,15443:32633/TCP   9m9s
service/istio-egressgateway         ClusterIP      172.30.116.82    <none>         15020/TCP,80/TCP,443/TCP                                     9m9s
service/istio-ingressgateway        ClusterIP      172.30.183.91    <none>         15020/TCP,80/TCP,443/TCP                                     9m10s
service/istiod-basic                ClusterIP      172.30.71.0      <none>         15010/TCP,15012/TCP,443/TCP,15014/TCP,8188/TCP               9m20s
service/jaeger-agent                ClusterIP      None             <none>         5775/UDP,5778/TCP,6831/UDP,6832/UDP                          9m10s
service/jaeger-collector            ClusterIP      172.30.63.55     <none>         9411/TCP,14250/TCP,14267/TCP,14268/TCP                       9m10s
service/jaeger-collector-headless   ClusterIP      None             <none>         9411/TCP,14250/TCP,14267/TCP,14268/TCP                       9m10s
service/jaeger-query                ClusterIP      172.30.155.56    <none>         443/TCP,16685/TCP                                            9m10s
service/prometheus                  ClusterIP      172.30.184.39    <none>         9090/TCP                                                     9m14s
service/ui-ingressgateway           LoadBalancer   172.30.134.116   20.237.32.49   15021:32675/TCP,80:30623/TCP,443:30481/TCP,15443:31772/TCP   9m9s
service/wasm-cacher-basic           ClusterIP      172.30.84.78     <none>         80/TCP                                                       8m48s
service/zipkin                      ClusterIP      172.30.225.80    <none>         9411/TCP                                                     9m10s

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana                1/1     1            1           9m9s
deployment.apps/http-egressgateway     1/1     1            1           9m10s
deployment.apps/http-ingressgateway    1/1     1            1           9m10s
deployment.apps/istio-egressgateway    1/1     1            1           9m10s
deployment.apps/istio-ingressgateway   1/1     1            1           9m11s
deployment.apps/istiod-basic           2/2     2            2           9m21s
deployment.apps/jaeger                 1/1     1            1           9m11s
deployment.apps/prometheus             1/1     1            1           9m15s
deployment.apps/ui-egressgateway       1/1     1            1           9m10s
deployment.apps/ui-ingressgateway      1/1     1            1           9m10s
deployment.apps/wasm-cacher-basic      1/1     1            1           8m49s

NAME                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/grafana-99d6457f                  1         1         1       9m11s
replicaset.apps/http-egressgateway-544f75d54d     1         1         1       9m12s
replicaset.apps/http-ingressgateway-7b69c76759    1         1         1       9m12s
replicaset.apps/istio-egressgateway-7fd685d798    1         1         1       9m12s
replicaset.apps/istio-ingressgateway-79bf956b68   1         1         1       9m13s
replicaset.apps/istiod-basic-6867796997           2         2         2       9m23s
replicaset.apps/jaeger-8c46c4f58                  1         1         1       9m13s
replicaset.apps/prometheus-69cd5746f              1         1         1       9m17s
replicaset.apps/ui-egressgateway-84dc658d4d       1         1         1       9m12s
replicaset.apps/ui-ingressgateway-5bb969c64c      1         1         1       9m12s
replicaset.apps/wasm-cacher-basic-567568d75       1         1         1       8m51s

NAME                                            HOST/PORT                                                        PATH   SERVICES               PORT          TERMINATION          WILDCARD
route.route.openshift.io/grafana                grafana-istio-system.apps.aro.example.opentlc.com                       grafana                <all>         reencrypt/Redirect   None
route.route.openshift.io/http-ingressgateway    http-ingressgateway-istio-system.apps.aro.example.opentlc.com           http-ingressgateway    8080                               None
route.route.openshift.io/istio-ingressgateway   istio-ingressgateway-istio-system.apps.aro.example.opentlc.com          istio-ingressgateway   8080                               None
route.route.openshift.io/jaeger                 jaeger-istio-system.apps.aro.example.opentlc.com                        jaeger-query           https-query   reencrypt            None
route.route.openshift.io/prometheus             prometheus-istio-system.apps.aro.example.opentlc.com                    prometheus             <all>         reencrypt/Redirect   None
route.route.openshift.io/ui-ingressgateway      ui-ingressgateway-istio-system.apps.aro.example.opentlc.com             ui-ingressgateway      8080                               None

NAME                                  ENDPOINTS                                                          AGE
endpoints/grafana                     10.131.0.56:3001                                                   9m13s
endpoints/http-egressgateway          10.129.2.68:15443,10.129.2.68:8080,10.129.2.68:8443                9m14s
endpoints/http-ingressgateway         10.131.0.53:15443,10.131.0.53:15021,10.131.0.53:8080 + 1 more...   9m14s
endpoints/istio-egressgateway         10.131.0.54:15020,10.131.0.54:8080,10.131.0.54:8443                9m14s
endpoints/istio-ingressgateway        10.129.2.67:15020,10.129.2.67:8080,10.129.2.67:8443                9m15s
endpoints/istiod-basic                10.129.2.65:8188,10.131.0.51:8188,10.129.2.65:15012 + 7 more...    9m25s
endpoints/jaeger-agent                10.128.2.27:5778,10.128.2.27:5775,10.128.2.27:6832 + 1 more...     9m15s
endpoints/jaeger-collector            10.128.2.27:14268,10.128.2.27:14250,10.128.2.27:9411 + 1 more...   9m15s
endpoints/jaeger-collector-headless   10.128.2.27:14268,10.128.2.27:14250,10.128.2.27:9411 + 1 more...   9m15s
endpoints/jaeger-query                10.128.2.27:16685,10.128.2.27:8443                                 9m15s
endpoints/prometheus                  10.129.2.66:3001                                                   9m19s
endpoints/ui-ingressgateway           10.131.0.52:15443,10.131.0.52:15021,10.131.0.52:8080 + 1 more...   9m14s
endpoints/wasm-cacher-basic           10.129.2.69:8080                                                   8m53s
endpoints/zipkin                      10.128.2.27:9411                                                   9m15s
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
