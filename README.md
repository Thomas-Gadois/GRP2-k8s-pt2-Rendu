# GRP2-k8s-pt2-Rendu

> **Membre du groupe 2 :**
> - GADOIS Thomas
> - ANEBAJAGAN Vivien
> - PERROCHON Milan
> - NARAINSAMY Riven

---

## Consignes :
- Installation ArgoCD
- Déployer une application via ArgoCD
- Ajout d’un certificat Let’s Encrypt à l’application déployée précédemment
- Illustration HPA
- Service Mesh avec Istio
- Application onlineboutique
- Exercice bonus - Déploiement d’un stack d’observabilité

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













---

## Illustration HPA

**Déploiement de l'application en exemple :** https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/

```
kubectl apply -f https://k8s.io/examples/application/php-apache.yaml
```

```bash
#php-apache.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache

```

![image (17)](https://github.com/user-attachments/assets/2f75e2e2-941f-4545-8c6c-e178fcac07d3)

Vérifiez le déploiement :

```
kubectl get pods -l app=php-apache
```

![image (18)](https://github.com/user-attachments/assets/0d940e55-d0ab-4722-b59c-99929d19136a)
![image (19)](https://github.com/user-attachments/assets/c020d61e-c96f-43b5-8da3-1cf1af97e0a6)

Configurez l'autoscaler pour cibler 50% d'utilisation CPU :

```
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

Verrifier le déploiment de l’autosacler : 

```bash
kubectl get hpa
```

![image (20)](https://github.com/user-attachments/assets/b3981c3b-38de-41fe-8173-255faab6742e)
![image (21)](https://github.com/user-attachments/assets/f5c89776-9ee4-4fc9-977c-98310606d175)

Génération de charge pour tester le scaling

```bash
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- \
  /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

![image (22)](https://github.com/user-attachments/assets/d99b8d71-6b0b-4b07-b6e4-66b91fcd2b11)
![image (23)](https://github.com/user-attachments/assets/d1d10e59-5ecc-4b96-9e43-7f1aac54ef93)

### Monitoring en temps réel

```
kubectl get hpa php-apache --watch
```

![image (24)](https://github.com/user-attachments/assets/b499cb3f-1de2-4b6d-ab4a-1f8a628a0937)

```
kubectl get deployment php-apache
```

![image (25)](https://github.com/user-attachments/assets/03d8d140-8999-48ee-b55c-e516df5392b7)

```
kubectl get hpa php-apache --watch
```

![image (26)](https://github.com/user-attachments/assets/b48ab40d-6a58-44c3-90f1-06e8404a24c9)

---

## Déploiement d’une image depuis un registre privé

---

## Service Mesh avec Istio

---

## Application onlineboutique

---

## Exercice bonus : Déploiement d’un stack d’observabilité
