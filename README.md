# GRP2-k8s-pt2-Rendu

> **Membre du groupe 2 :**
> 
> - GADOIS Thomas
> - ANEBAJAGAN Vivien
> - PERROCHON Milan
> - NARAINSAMY Riven

---

## Installation ArgoCD :

Commande d’installation :

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Verrification du fonctionnement des pods :

```bash
kubectl get pods -n argocd
```

![image.png](attachment:46a62695-5940-4588-bc7f-39db00f9e91b:image.png)

Passer le service argocd-server en mode LoadBalancer :

```bash
kubectl patch svc argocd-serveur - argocd \ 
-p '{"spec": {"type": "LoadBalancer"}}'
```

Verrification du changement et récupération de l’ip externe du service :

```bash
kubectl get svc argocd-server -n argocd
```

![image.png](attachment:ec651d7e-feb4-45e3-8107-f7465ba662f5:image.png)

```bash
https://aa88b4a36778a48f49818c945fbc8ccb-33358181.us-east-1.elb.amazonaws.com/applications
```

Récupération des credentials pour se connecter au service :

```bash
kubectl -n argocd \ 
get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d && echo
```

Connexion au service avec l’ip externe et le mot de passe :

![image.png](attachment:6e5e4a53-5728-47c2-b845-1cdafa95865a:image.png)

---

## Déployer une application via ArgoCD :

Déployer l’application depuis le repository : https://github.com/paulbouwer/hello-kubernetes

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx helm repo update
```

Verrification du déploiment du service :

```bash
kubectl get svc ingress-nginx-controller -n ingress-ngninx
```

![image.png](attachment:cd8a537f-3f01-4489-bbcd-9f964fc58d8e:image.png)

Création de l’application hello-k8s : 

```bash
argocd app create hello-k8s \ 
--repo https://github.com/paulbouwer/hello-kubernetes.git\ 
--path . \
--dest-server https://kubernetes.default.svc \
--dest-namespace default
```

Verrification du déploiment de l'application :

```bash
kubectl get pods,svc -n ingress-ngninx
```

![image.png](attachment:b7bb1e0b-3023-4816-8440-da892d7e02ce:image.png)

![image.png](attachment:b9c9f84f-80ee-4bd3-a454-9228c560e849:image.png)

Fichier du déploiment de l’application :

```bash
#hello-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-kubernetes
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-kubernetes
  template:
    metadata:
      labels:
        app: hello-kubernetes
    spec:
      containers:
      - name: hello-kubernetes
        image: paulbouwer/hello-kubernetes:1.7
        ports:
        - containerPort: 8080
```

Fichier du service :

```bash
#hello-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-kubernetes
spec:
  selector:
    app: hello-kubernetes
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

Commande de déploiment : 

```bash
kubectl apply -f hello-service.yaml
```

Installation de l’Ingress Controller :

Fichier Ingress : 

```bash
#hello-ingress.yaml :
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-ingress
spec:
  rules:
  - host: <load-balancer-host>
    http:
      paths:
      - path: /
        backend:
          service:
            name: hello-kubernetes
            port:
              number: 80
```

Commande de déploiment : 

```bash
kubectl apply -f hello-ingress.yaml
```

Verrification du déploiment :

```bash
kubectl get ingress
```

Vérification de la page avec l’url de l’ingress : 

```bash
http://a372728969e5144ca86a0499e4cfe11c-706910546.us-east-1.elb.amazonaws.com/
```

![image.png](attachment:86193405-9fa1-4903-8145-49760b2e5396:image.png)

![image.png](attachment:53731e09-f3f3-4b43-851e-eac9eb9cf94a:image.png)
