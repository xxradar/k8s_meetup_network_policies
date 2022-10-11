## Kubernetes native network security policies
### Setting up a lab environment
```
kubectl create ns prod-nginx
kubectl create ns dev-nginx
kubectl create ns myhackns
```
```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: prod-nginx
  labels:
    app: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        env: prod
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
```
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-clusterip
  namespace: prod-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: nginx
EOF
```
### Check connectivity
```
kubectl get po -n prod-nginx -o wide  --show-labels
...
kubectl get svc -n prod-nginx -o wide --show-labels
...
```
```
kubectl run -it --rm -n prod-nginx --image xxradar/hackon debug
curl my-nginx-clusterip
curl <pod_ip>
```
## Network policies
### Default-deny
Apply a default-deny all policy
```
kubectl apply -n prod-nginx -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
   - Ingress
   - Egress
EOF
```
Check connectivity
```
kubectl run -it --rm -n prod-nginx --image xxradar/hackon debug
nslookup my-nginx-clusterip
...
curl my-nginx-clusterip
...
curl <a_prod-nginx_pod_ip>
...
```
### DNS egress 
Fix the DNS resolving <br>
If required (depending on cluster initialisation) label the `kube-system` namespace
```
kubectl label ns kube-system kubernetes.io/metadata.name=kube-system
```
```
kubectl apply -n prod-nginx -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
   - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
EOF
```
```
kubectl run -it --rm -n prod-nginx --image xxradar/hackon debug
nslookup my-nginx-clusterip
...
curl my-nginx-clusterip
...
curl <a_prod-nginx_pod_ip>
...
```
### HTTP ingress (server-side)
Enable access on port 80
```
kubectl apply -n prod-nginx -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-http
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels: {}
    ports:
    - protocol: TCP
      port: 80
EOF
```
Check connectivity
```
kubectl run -it --rm -n prod-nginx --image xxradar/hackon debug
nslookup my-nginx-clusterip
...
curl my-nginx-clusterip
...
curl <a_prod-nginx_pod_ip>
...
```
### HTTP egress (client-side)
```
kubectl apply -n prod-nginx -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-debug-egress
spec:
  podSelector:
    matchLabels:
      run: debug
  egress:
  - to:
    - podSelector:
        matchLabels: {}
EOF
```
```
kubectl run -it --rm -n prod-nginx --image xxradar/hackon debug
nslookup my-nginx-clusterip
...
curl my-nginx-clusterip
...
curl <a_prod-nginx_pod_ip>
...
```

### HTTP ingress different namespace (client-side)
Connectivity form a different namespace ...
```
kubectl apply -n prod-nginx -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-http
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels: {}
    ports:
    - protocol: TCP
      port: 80
  - from:
    - namespaceSelector:
        matchLabels:
          project: debug
      podSelector:
        matchLabels:
          mode: debug
    ports:
    - protocol: TCP
      port: 80
EOF
```
or
```
kubectl apply -n prod-nginx -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-http-other-namespace
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          project: debug
      podSelector:
        matchLabels:
          mode: debug
    ports:
    - protocol: TCP
      port: 80
EOF
```
```
kubectl label ns myhackns project=debug
```
```
kubectl run -it --rm  -n myhackns --image xxradar/hackon -l mode=debug debug
curl my-nginx-clusterip.prod-nginx
...
```
```
kubectl run -it --rm  -n myhackns --image xxradar/hackon -l mode=nodebug debug
curl my-nginx-clusterip.prod-nginx
...
```
```
kubectl run -it --rm  -n dev-nginx --image xxradar/hackon -l mode=debug debug
curl my-nginx-clusterip.prod-nginx
...
```
```
kubectl label ns dev-nginx project=debug

kubectl run -it --rm  -n dev-nginx --image xxradar/hackon -l mode=debug debug
curl my-nginx-clusterip.prod-nginx
...
```

