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

### Test de l’auto healing :

On suprime un pod et il est recréé automatiquement :

![image](https://github.com/user-attachments/assets/e236d4a0-9cd1-4dba-92fe-a78608600e47)


---

## Ajout d’un certificat Let’s Encrypt à l’application déployée

![460742771-04a36e05-8762-4097-a938-e0b58d66bb6b](https://github.com/user-attachments/assets/c96a847b-9a4f-433c-b5ff-691a09caaa6a)

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

**On constate bien que le pod est scale automatiquement pour répondre a la montée en charge.**

---

## Déploiement d’une image depuis un registre privé

Depuis dockerhub, créer le répertoire privé : 

![image (1)](https://github.com/user-attachments/assets/713aa6ae-7bb7-4758-a236-ef5c24f42f6c)

Faire un clone de l’image hello-kubernetes

```bash
git clone https://github.com/paulbouwer/hello-kubernetes.git
cd hello-kubernetes\src\app
ls
```

![image (2)](https://github.com/user-attachments/assets/42945fc1-6938-49e9-85e7-a305613fae19)

Lancer le build de l’image : 

```bash
docker build -t hello-kubernetes: prive
```

![image (3)](https://github.com/user-attachments/assets/6556df32-cb53-44ad-a782-e6320c4ca75f)

Envoyer l’image docker dans le répo privé dans dockerhub

```bash
docker push hello-kubernetes: prive
```

![image (4)](https://github.com/user-attachments/assets/ce938f5d-83f7-47c9-84d7-1e67ee59c14e)

Depuis dockerhub, créer le tocken d’acces : 

![image (5)](https://github.com/user-attachments/assets/9ee211f6-400c-4901-87a2-e7e6d5a9a053)

Créer le secret avec le tocken d’acces : 

```bash
kubectl create secret docker-registry regcred --docker-username=username --docker-password=ACCES_TOKEN --docker-email
```

Creation du pod avec l’image privé :

```bash
apiVersion: v1 
kind: Pod
metadata:
	name: hello-kubernetes-private
spec:
	containers:
	- name: hello-kubernetes
		image: hello-kubernetes:prive 
	imagePullSecrets:
	- name: regcred
```

Déploiment du pod avec l’image privé :

```bash
kubectl apply -f pod-hello-private.yaml
```

Vérrification du déploiment :

```bash
kubectl get pods
kubectl describe pod hello-kuberntes-private
```

![image (6)](https://github.com/user-attachments/assets/df798669-eee5-4eee-8a36-c4accaf6530c)

![image (7)](https://github.com/user-attachments/assets/0e4a4466-65cd-45bf-9ed7-a48cde29d05a)

![image (8)](https://github.com/user-attachments/assets/c6b03599-0ddc-42e6-8b49-05834e4337d1)

Vérrification du déploiment sur dockerhub :

![image (9)](https://github.com/user-attachments/assets/98c0a9be-e37a-45bd-a671-618af6cf6e25)

---

## Service Mesh avec Istio

Téléchargement et installation d'Istio

```bash
.\istioctl install --set profile=demo -y
```

![image](https://github.com/user-attachments/assets/406beac2-5a18-4865-94aa-36483f1dcf73)

Installation des composants Istio

```bash
kubectl apply -f .\samples\bookinfo\platform\kube\bookinfo.yaml
```

![image](https://github.com/user-attachments/assets/3c85531f-239e-472f-8982-dc0c3192379d)

![image](https://github.com/user-attachments/assets/6757eb2d-cfb1-4efe-a5df-8160cc8788d3)

Activer l’injection automatique d’Istio sur le namespace default

```bash
kubectl label namespace default istio-injection-enabled --overwrite namespace/default labeled
```

 Supprimer les pods Bookinfo pour qu’ils soient recréés avec le sidecar :

![image](https://github.com/user-attachments/assets/9a608841-905f-4a90-8b29-f5e32c47c2fd)

Déployer le Gateway pour exposer l’app :

```bash
kubectl apply -f .\samples\bookinfo\networking\bookinfo-gateway.yaml
```

![image](https://github.com/user-attachments/assets/1b43d7bc-3594-462a-a64a-aebd03fecb8e)

Vérifier le service d'entrée (Ingress Gateway Istio) :

![image](https://github.com/user-attachments/assets/a3cb7811-7ec8-418d-8347-e83073eb71ba)

Récupérer l’URL pour accéder à l’app :

![image](https://github.com/user-attachments/assets/21b3ed5f-4852-4e65-a2ef-7d4a3d112463)

![image](https://github.com/user-attachments/assets/2f97ce0c-6434-40b6-929c-8835e275eb3e)

Tester la répartition de trafic :

![image](https://github.com/user-attachments/assets/bfaa8774-f8b0-47ff-ab6d-462c538e45e0)

![image](https://github.com/user-attachments/assets/860b4ff0-943c-40b3-9276-10ff6061de35)

![image](https://github.com/user-attachments/assets/ab8e785e-1fed-4cdb-82ca-4a1a682bdf5f)

![image](https://github.com/user-attachments/assets/1d04c245-4b95-4d7d-8c95-1121f6192256)

![image](https://github.com/user-attachments/assets/251d8e23-c18e-458e-a3f4-63579b9b32d3)

Accecible depuis notre loadbalancer :

```bash
 http://a75e1b4ee25e94c9c80278ad208479df-742479764.us-east-1.elb.amazonaws.com/productpage
```

![image](https://github.com/user-attachments/assets/31cd4bda-d471-4452-a5b9-085e698e9518)

Répartition du trafic :

![image](https://github.com/user-attachments/assets/12865ef8-23d5-4419-8c69-b461e01c4c57)

![image](https://github.com/user-attachments/assets/77ad4196-861b-4b1c-b884-ee963e4cbe57)

Configuration du routage par défaut :

![image](https://github.com/user-attachments/assets/bd223733-680e-4c34-b036-d0751dd33a30)

Check des virtuals services :

![image](https://github.com/user-attachments/assets/fe72d488-6bbd-414a-bf8f-a3cb202872ab)

![image](https://github.com/user-attachments/assets/f7c889bd-b74d-45a0-b27f-cebda668c2d4)

Test : 


https://github.com/user-attachments/assets/fd0184ff-e5c6-4a3c-9c5c-5e3d624d2fca

![image](https://github.com/user-attachments/assets/051a69ad-f6ac-4e03-a55b-310ed691dbb5)

---

## Exercice bonus : Déploiement d’un stack d’observabilité

La mise en place de la stack de monitoring Prometheus/Grafana avec du stockage persistant n'est  pas possible car le contrôleur EBS CSI ne dispose pas des permissions requises sur AWS.

Voici une démonstration de celle-ci sans le stockage persistant :
![image](https://github.com/user-attachments/assets/0793d291-e66b-44b0-b9ad-edfa7634e644)
![image](https://github.com/user-attachments/assets/a23b06fd-058f-4ca8-a302-5a5fc1a9fc78)
![imagesdeegfv](https://github.com/user-attachments/assets/c70852fa-2b10-4d3c-88d5-06d82a89aded)
![image](https://github.com/user-attachments/assets/34fad9fa-0b61-43ce-abf6-6910b63017e2)



