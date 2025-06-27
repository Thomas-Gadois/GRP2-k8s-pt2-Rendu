# GRP2-k8s-pt2-Rendu

> **Membre du groupe 2 :**
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

![image (2)](https://github.com/user-attachments/assets/475571e0-074a-43df-b6bc-92f54b26381c)


Passer le service argocd-server en mode LoadBalancer :

```bash
kubectl patch svc argocd-serveur - argocd \ 
-p '{"spec": {"type": "LoadBalancer"}}'
```

Verrification du changement et récupération de l’ip externe du service :

```bash
kubectl get svc argocd-server -n argocd
```

![image (3)](https://github.com/user-attachments/assets/c0031054-7722-4b3c-a204-68ce690cb774)


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

![image (4)](https://github.com/user-attachments/assets/16cc6759-fd69-4aaa-87b7-76161cc321ca)


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

![image (5)](https://github.com/user-attachments/assets/63a0cfb6-61f8-4416-a0ab-9164e68ef258)


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

![image (6)](https://github.com/user-attachments/assets/17b792dc-42a4-4971-b5b2-10f1d6ea03a7)
![image (7)](https://github.com/user-attachments/assets/b20c5f10-7350-4b76-a595-887dedb41635)


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

![image (8)](https://github.com/user-attachments/assets/a9cff575-8946-4834-9c79-151e1d00e052)
![image (9)](https://github.com/user-attachments/assets/dda53c43-f27c-4c08-ac18-6100d0298824)

## Test des différents scénarios de résilience :

### En sync manuelle :

Suppression du pod hello-kubernetes :

```bash
kubectl delete service hello-kubernetes-hello-kubernetes -n hello-kubernetes
```

![image (10)](https://github.com/user-attachments/assets/e22f445d-c557-49b4-a42d-fef35ed234f7)
![image (11)](https://github.com/user-attachments/assets/e79f1873-cadc-404b-8484-af09b99bf7ca)

Depuis argocd l’application passe en missing :

![image (12)](https://github.com/user-attachments/assets/afb7472d-05e6-419c-bd87-2c86fde57adb)

La synchronisation est relancée :

![image (13)](https://github.com/user-attachments/assets/939902c6-5cf5-408f-8c21-8dbd4f4c4fd1)
![image (14)](https://github.com/user-attachments/assets/b7c42df7-ada1-45e7-b97c-3780c49c52e3)

Une fois le service remonté, on constate que le pod est de nouveau présent : 

![image (15)](https://github.com/user-attachments/assets/0d0bdb01-9763-4193-ae71-5b1f0e34852c)

### En sync automatique :

Configurer la Sync Policy en automatique : 

![image (16)](https://github.com/user-attachments/assets/8627750d-6686-47e7-9700-b0ab98b927b9)

Suppression du pod hello-kubernetes :

```bash
kubectl delete service hello-kubernetes-hello-kubernetes -n hello-kubernetes
```

Démonstration de l’auto-sync :


https://github.com/user-attachments/assets/801aa7b1-e88c-4ec2-90f8-6ee02ca486fa


---

## Ajout d’un certificat Let’s Encrypt à l’application déployée
