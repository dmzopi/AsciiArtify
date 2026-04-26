### 1. Встановлення k3d
```bash
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```
### 2. Створення кластера
```bash
k3d cluster create argo \
  --port "443:443@loadbalancer"
```
### 3. Install Argo CD
https://argo-cd.readthedocs.io/en/stable/getting_started/
```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd get all
```

### 4. Traefic ingress route
(see https://argo-cd.readthedocs.io/en/latest/operator-manual/ingress/)
```bash
# Створити маніфест
vi ingress.yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: argocd-server
  namespace: argocd
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: Host(`argocd.ubuntu-vm.home`)
      priority: 10
      services:
        - name: argocd-server
          port: 80
    - kind: Rule
      match: Host(`argocd.ubuntu-vm.home`) && Header(`Content-Type`, `application/grpc`)
      priority: 11
      services:
        - name: argocd-server
          port: 80
          scheme: h2c
  tls:
    certResolver: default

# Створити ingress правило
kubectl apply -f ingress.yaml

# Перевірити правило
kubectl -n argocd get ingressroutes.traefik.io

# argocd-server слухає на порту 443, треба перекинути на 80, щоб запобігти redirect loop: Client → HTTPS → Traefik → HTTP → ArgoCD → redirect to HTTPS → Traefik → ...

kubectl -n argocd patch configmap argocd-cmd-params-cm   --type merge   -p '{"data":{"server.insecure":"true"}}'
kubectl -n argocd rollout restart deployment argocd-server 
```

### 5. Отримати дефолтний пароль
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"| base64 -d; echo
```
### 6. Доступ до GUI
```bash
# Додати argocd.ubuntu-vm.home до /etc/hosts
ip_address argocd.ubuntu-vm.home
```
Відкрити ArgoCD в браузерs: https://argocd.ubuntu-vm.home
```
Username: admin
Password: дефолтний пароль
```
![Image](.data/argo.gif)


---
