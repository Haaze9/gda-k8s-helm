# TP HELM

Helm est un gestionaire de packet kubernetes , il permet d'installer des chart

une Chart est un bloc coherent contenant tout les manifests neccessaire à l'installation de l'application 

Une release est une version déployer d'une chart



** Installation de helm **
Il existe plusieurs methode d'installation de helm , ici l'installation se fera par script disponible sur le [site officiel de helm](https://helm.sh/docs/intro/install/)

```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```
Verifier l'installation avec la commande ``helm version``

il est possible d'installer la chart sans la modifier . 
Pour modifier la chart helm intall set --<variables><variable> <NoM de la charte>

---

``` 
milla@debian-k8s:~$ helm version
version.BuildInfo{Version:"v3.17.2", GitCommit:"cc0bbbd6d6276b83880042c1ecb34087e84d41eb", GitTreeState:"clean", GoVersion:"go1.23.7"}
``` 

## Installation helm Chart Nexcloud

Sur artifacthub.io , recuperer les information pour l'installation de la chart nextcloud 

#### Installation du repo helm nextcloud
- Installer le repo permettra de recuperer la charte / le package nextcloud 

```sh 
helm repo add nextcloud https://nextcloud.github.io/helm/
```
- Installer le package helm avec la commande helm install 
```
milla@debian-k8s:~/gda-k8s-h3lm$ helm install nxt-release nextcloud/nextcloud
NAME: nxt-release
LAST DEPLOYED: Sat Mar 22 22:04:59 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
```

La commande ```Helm get manifest```
 affiche les fichier de ressources (ingress, deployment ..)

## Configuration de la chart , Set  des variable 

Pour récupérer la valeur des variables utilisateur utiliser la commande 

```helm get value <nomdela release> ```

Afin de recuperer toutes les variables de la chart , **ici aucune variable n'ont été definie la chart helm à été installer puis déployée**

helm get value --all 
```
export APP_HOST=127.0.0.1
export APP_PASSWORD=$(kubectl get secret --namespace default nxt-release-nextcloud -o jsonpath="
 ```



- Vérification du déploiement 
```
milla@debian-k8s:~/gda-k8s-h3lm$ kubectl get po
NAME                                     READY   STATUS    RESTARTS   AGE
nxt-release-nextcloud-5bfb7f4d4c-zs7gf   0/1     Running   0          2m4s
```

# Création d'une charte personnalisée 

Ici l'ojectif est de crée une charte helm qui permettrait de déployer 3 Magasin 

- Création de la Chart(structure)

la commande helm create permet de generer une arborescence de fichier par defaut afin de structurer notre chart ici le nom de la chart crée est mystore 

``` helm create mystore```

Apres suppresion des fichier inutiles voici l'arborescence finale

```

├── mystore
│   ├── charts
│   ├── Chart.yaml
│   ├── templates
│   │   ├── deployment.yaml
│   │   ├── _helpers.tpl
│   │   ├── ingress.yaml
│   │   ├── NOTES.txt
│   │   └── service.yaml
│   └── values.yaml

```
## Présentation des manifests

INGRESS.yaml
```apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mysite-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
{{- range .Values.stores }}
  - host: {{ .name }}.fr
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: {{ .name }}-service
            port:
              number: {{ .containerPort | default 80 }}

     {{- if .hasBioPath }}
      - path: /bio
        pathType: Prefix
        backend:
          service:
            name: {{ .bioServiceName | default (printf "%s-service" .name) }}
            port:
              number: {{ .bioContainerPort | default 80 }}
        {{- end }}
{{- end }}

```
- VALUES.yaml

**Le fichier values.yaml contient les information qui seront recupéré via les variables.**

Pour le magasin bio une condition a été crée pour redirigé les requetes vers le bon magasin 

```
stores:
  - name: mesbonslegumes-bio
    image: hazegoodlife/website-bio:v1
    hasBioPath: true
    bioServiceName: mesbonslegumes-bio-service
    bioContainerPort: 80
    replicas: 1
    containerPort: 80

  - name: monbonlait
    image: hazegoodlife/haaze:milksite
    replicas: 1
    containerPort: 80

  - name: mesbonslegumes
    image: hazegoodlife/haaze:veggiesite
    replicas: 1
    containerPort: 80

```

- DEPLOYMENT.yaml
```{{- range .Values.stores }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .name }}
  labels:
    app: {{ .name }}
spec:
  replicas: {{ .replicas | default 1 }}  # Défaut : 1 réplica
  selector:
    matchLabels:
      app: {{ .name }}
  template:
    metadata:
      labels:
        app: {{ .name }}
    spec:
      containers:
      - name: {{ .name }}
        image: {{ .image }}
        ports:
        - containerPort: {{ .containerPort | default 80 }}  # Défaut : port 80
---
{{- end }}

```

## Installation de la nouvelle chart 

- Installer la nouvelle chart avec la commande helm install
```
milla@debian-k8s:~/gda-k8s-h3lm$ helm install mystore ./mystore
NAME: mystore
LAST DEPLOYED: Thu Apr  3 23:11:49 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Déploiement des magasins OK plus qu'a testé CURLLLLLLL
```

- Verification de la création des ressources 

```
milla@debian-k8s:~/gda-k8s-h3lm$ kubectl get po

NAME                                  READY   STATUS    RESTARTS   AGE
mesbonslegumes-6f794bf58c-8dd8c       1/1     Running   0          66s
mesbonslegumes-bio-5c4cd8dc49-db7j5   1/1     Running   0          66s
monbonlait-86b9946969-l9m4z           1/1     Running   0          66s
```
- Verification Création Ingress 

milla@debian-k8s:~/gda-k8s-h3lm$ kubectl get ingress
NAME             CLASS     HOSTS                                                   ADDRESS          PORTS   AGE
mysite-ingress   traefik   mesbonslegumes-bio.fr,monbonlait.fr,mesbonslegumes.fr   192.168.10.135   80      5m32s

- Test de validation avec Curl 

```

milla@debian-k8s:~/gda-k8s-h3lm$ curl http://monbonlait.fr
<!DOCTYPE html>
<html>
<head>
  <title>Bienvenue!</title>
</head>
<body>
  <h1>Bienvenue sur mon site de Lait </h1>
  <p>l'amour est dans le pré</p>
</body>
</html>

### Push de la nouvelle chart sur Artifacthub.io

Tout d'abord sur Github dans les paremetres du repo > puis "pages" et selectionner la branche qui servira les pages 

