# Setup Ingress Nginx for EKS cluster

![ingress-nginx-routes](/resources/images/Nginx/ingress-nginx-routes.png)

In this content, we will setup ingress NGINX inside the cluster, which is used to publicly access our applications.Let's move on to Rancher Home and expose a cluster on which we want to deploy INGRESS-NGINX, then start `kubectl shell` at the top right of the page.

First, We download & pull Helm chart of ingress-nginx

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

## Install Ingress Nginx without using CloudFlare

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --version 4.10.0 \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.config.enable-real-ip="true" \
  --set controller.config.use-forwarded-headers="true" \
  --set controller.service.external.enabled=true \
  --set controller.service.externalTrafficPolicy="Local" \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-proxy-protocol"="*" \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"="nlb" \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-nlb-target-type"="ip" \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-scheme"="internet-facing" \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-backend-protocol"="tcp"
```

> The ingress must deploy on public node (the node has public IP)

## Install Ingress Nginx using CloudFlare

```bash
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --version 4.10.0 \
  --create-namespace \
  --namespace ingress-nginx \
  --set controller.service.type=LoadBalancer \
  --set controller.config.enable-real-ip="true"
```

After a moment, the installation will be done, and we can check the deployment of `ingress-nginx` in `ingress-nginx` namespace.

At this time, `ingress-nginx` will create an ELB domain, which will be used to expose services. We can get it and update the *`CloudFlare`* or *`Route53`* records later.

```bash
kubectl get svc -o wide -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP                                                                    PORT(S)                      AGE   SELECTOR
ingress-nginx-controller             LoadBalancer   172.20.116.36    ab3293d252747434f95a55273ab21545-1488261532.ap-northeast-1.elb.amazonaws.com   80:31924/TCP,443:32428/TCP   49m   app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx
```

## Create ingress to expose service on the internet

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  labels:
    app: myapp
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 5000
```

Ater the ingress has been created, we are able to access our service publicly

## Restrict connection by whitelisting

Using [ingress-nginx](https://kubernetes.github.io/ingress-nginx/user-guide/basic-usage/) annotations will help to configure this optional

```yaml
nginx.ingress.kubernetes.io/whitelist-source-range: list-of-ips
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  namespace: staging
  annotations:
    # Whitelist specific IP addresses or CIDR ranges
    nginx.ingress.kubernetes.io/whitelist-source-range: "203.0.113.0/24, 198.51.100.4"
    # Optional: Add custom error page for denied access (403 Forbidden)
    nginx.ingress.kubernetes.io/custom-http-errors: "403"
    nginx.ingress.kubernetes.io/default-backend: "error-backend"
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
  # Optionally, you can define a default backend for handling restricted access or errors
  defaultBackend:
    service:
      name: error-backend
      port:
        number: 80
```

## Configure cert-manager for SSL

Some of our domain names are not issued certification by CloudFlare, so when we want to route our service through SSL, we are able to manage this using cert-manager. cert-manager is an open source project that builds on top of Kubernetes to provide X.509 certificates and issuers as first class resource types. For more detail, take a look to its [documentation](https://cert-manager.io/docs/).

### Install cert-manager to K8s cluster by Helm chart

- **Step 1. Add Helm repository**

  ```bash
  helm repo add jetstack https://charts.jetstack.io
  helm repo update
  ```

- **Step 2. Install `CustomResourceDefinitions`**

  ```bash
  kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.1/cert-manager.crds.yaml
  ```

- **Step 3. Install cert-manager**
  
  ```bash
  helm install cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --version v1.16.1
  ```

Now, wait the pods to be coming up, you can check it by command

```bash
$ kubectl get pods -n cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-6957cdbc5d-r8d58              1/1     Running   0          3h36m
cert-manager-cainjector-56c499cdb8-ps7jz   1/1     Running   0          3h36m
cert-manager-webhook-5585dcddb-trnjx       1/1     Running   0          3h36m
```

![cert-manager-overview](/resources/images/Nginx/cert-manager-overview.svg)

### Create ClusterIssuer

`Issuers`, and `ClusterIssuers`, are Kubernetes resources that represent certificate authorities (CAs) that are able to generate signed certificates by honoring certificate signing requests.

```bash
$ cat << EOF | kubectl apply -f
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    email: youremail@gmail.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt
    solvers:
      - http01:
          ingress:
            class: nginx
EOF
```

> *Note: should use validately email address*

Validate the ClusterIssuer

```bash
$ kubectl get clusterissuer
NAME               READY   AGE
letsencrypt   True    0h1m
```

### Create Certificate resource

`cert-manager` has the concept of Certificates that define a desired X.509 certificate which will be renewed and kept up to date. A Certificate is a namespaced resource that references an Issuer or ClusterIssuer that determine what will be honoring the certificate request.

```bash
$ cat << EOF | kubectl apply -f
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp
  namespace: staging
spec:
  secretName: myapp-tls
  dnsNames:
    - myapp.example.com
  issuerRef:
    name: letsencrypt
    kind: ClusterIssuer
EOF
```

Check the certificate

```bash
$ kubectl get certificate -n staging
NAME        READY   SECRET      AGE
myapp       True    myapp-tls   0h1m
myapp-tls   True    myapp-tls   0h1m
```

### Update ingress object for our service

Here is the ingress object for our application to serve the external traffic to the cluster over HTTPS securely:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    acme.cert-manager.io/http01-edit-in-place: 'true'
    cert-manager.io/cluster-issuer: letsencrypt
  name: myapp-ingress
  labels:
    app: myapp
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 5000
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
```