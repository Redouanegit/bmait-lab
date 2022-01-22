# Installation et configuration de Nginx Ingress Controller, Cert Manager et Let's Encrypt



Ajouter la repo helm de nginx ingress dans le cluster Kubernetes 

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```
Metre à jour la repo helm
```bash
helm repo update
```
Installer Nginx Ingress Controller Kubernetes utilisant Helm 3
```bash
kubectl create ns demo-bma
```
```bash
helm -n demo-bma install ingress-nginx ingress-nginx/ingress-nginx
```
Output
```bash
NAME: ingress-nginx
LAST DEPLOYED: Sat Jan 22 07:46:02 2022
NAMESPACE: demo-bma
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:	
The ingress-nginx controller has been installed.
It may take a few minutes for the LoadBalancer IP to be available.
You can watch the status by running 'kubectl --namespace demo-bma get services -o wide -w ingress-nginx-controller'

An example Ingress that makes use of the controller:
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example
    namespace: foo
  spec:
    ingressClassName: nginx
    rules:
      - host: www.example.com
        http:
          paths:
            - backend:
                service:
                  name: exampleService
                  port:
                    number: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>	
  type: kubernetes.io/tls
```
Vérifier Nginx ingress Controller
```bash
kubectl get services ingress-nginx-controller
```
Output:
```bash
	NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller   LoadBalancer   10.233.42.123   10.100.100.73   80:30128/TCP,443:30397/TCP   7m33s
```
Création du Deployment et du service pour l'application nginx
```bash
kubectl create ns demo-bma
```
```bash
sudo vim nginx-deploy.yaml
```
```bash
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nginx-app
  namespace: demo-bma
  labels:
    app: nginx-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: nginx
        image: nginx
```
```bash
sudo vim nginx-svc.yaml
```
```bash
apiVersion: v1
kind: Service
metadata:
  name: nginx-app
  namespace: demo-bma
spec:
  selector:
    app: nginx-app
  ports:
  - name: http
    targetPort: 80
    port: 80
  - name: https
    targetPort: 443
    port: 443
```
deployer les ressources Deployment et Service de l'application nginx dans kubernetes
```bash
kubectl create -f nginx-deploy.yml
kubectl create -f nginx-svc.yml
```
Creation de la resourece Ingress est exposition de l'application
```bash
sudo vim nginx-ingress.yaml
```
```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: demo-bma
  annotations:
    kubernetes.io/ingress.class: nginx   
spec:
  rules:
  - host: nginxapp.fosstechnix.info
    http:
      paths:
      - backend:
          service:
            name: nginx-app
            port:
              number: 80
        path: /
        pathType: Prefix
```
```bash
kubectl create -f nginx-ingress.yaml
```
On assume que l'enregistrement CNAME est déja ajouté pour pointer l'URL du Loadbalancer

Installation et configuration de Cret Manager
```bash
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.6.0/cert-manager.yaml
```
Configuration de LetsEncrypt issuer 
```bash
sudo vim letsencrupt-issuer.yaml
```
```bash
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: demo-bma
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: fosstechnixinfo@gmail.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: nginx
```
```bash
kubectl apply -f letsencrypt-issuer.yml
```
Creation de Let’s Encrypt TLS Certificate
```bash
sudo vim letsencrypt-cert.yml
```
```bash
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: labs-1.bmait.com
  namespace: demo-bma
spec:
  secretName: labs-1.bmait.com-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: labs-1.bmait.com
  dnsNames:
  - labs-1.bmait.com
```
```bash
kubectl apply -f letsencrypt-cert.yml
```
Pointer le certificat de Let’s Encrypt sur Nginx Ingress ressource
```bash
kubectl edit ingress nginx-ingress
```
```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    <mark>cert-manager.io/cluster-issuer: letsencrypt-prod</mark>
    kubernetes.io/ingress.class: nginx
  creationTimestamp: "2022-01-22T13:52:01Z"
  generation: 1
  name: nginx-ingress
  namespace: demo-bma
  resourceVersion: "25043451"
  uid: 8824a299-c906-4353-807a-2db8d3a6b87b
spec:
  <mark>tls:
  - hosts:
    - labs-1.bmait.com
    secretName: labs-1.bmait.com-tls</mark>
  rules:
  - host: labs-1.bmait.com
    http:
      paths:
      - backend:
          service:
            name: nginx-app
            port:
              number: 80
        path: /
        pathType: Prefix
```


