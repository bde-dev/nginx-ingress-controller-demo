# nginx-ingress-controller-demo

A repository containing a basic working set up of the NGINX Ingress Controller.

> Tested working on k3s, and assumes `kubectl` and `helm` installed.

## Basic Local Setup

### Install k3s

Run install k3s script

```bash
curl -sfL https://get.k3s.io | sh -
```

Create a k3s config file to disable traefik

```yaml
# /etc/rancher/k3s/config.yaml
disable:
  - traefik
```

Restart k3s

```bash
sudo systemctl restart k3s.service
```

### Add Helm Repos

```bash
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update nginx-stable
```

### Install Nginx Ingress Controller Chart

```bash
helm install nginx-ingress nginx-stable/nginx-ingress -n nginx-ingress --create-namespace
```

### Set Test Host

Grab the nginx ingress controller service IP provided by servicelb

```bash
kubectl get svc -n nginx-ingress
```

Add it to `/etc/hosts`

```yaml
# /etc/hosts
<nginx-ingress-ip> test.local
```

### Deploy Test Nginx Manifests

Apply these mainfests

```yaml
# nginx-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-nginx
  namespace: nginx-ingress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-nginx
  template:
    metadata:
      labels:
        app: test-nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

```yaml
# nginx-service.yml
apiVersion: v1
kind: Service
metadata:
  name: test-nginx
  namespace: nginx-ingress
spec:
  selector:
    app: test-nginx
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

```yaml
# nginx-ingress.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-nginx-ingress
  namespace: nginx-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: test.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: test-nginx
                port:
                  number: 80
```

```bash
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml
kubectl apply -f nginx-ingress.yaml
```

### Test

```bash
(ins)❯ curl -k http://test.local
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
