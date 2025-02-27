# Kube TP 1 - `Introduction à Kubernetes`

## 1. Installer `Minikube` et `Kubectl`
Suivre cette documentation : https://minikube.sigs.k8s.io/docs/start/ 
## 2. `Pod Nginx`

### Héberger un premier Pod `Nginx` :

✅ Option 1 : Créer un Pod Nginx via kubectl run (commande rapide)

Pour créer rapidement un Pod Nginx, on peut exécuter la commande suivante :

```bash
kubectl run my-nginx --image=nginx --restart=Never
```

- `kubectl run` permet de lancer un Pod unique.

- `--image=nginx` indique l’image Docker officielle de Nginx.

- `--restart=Never` indique à Kubernetes de ne pas créer de `ReplicaSet` ou de `Deployment` derrière ce Pod (c’est donc juste un Pod “simple”).

Note : On pourrait également créer un Deployment au lieu d’un Pod. L’avantage du Deployment est la possibilité de gérer les mises à jour (rolling updates, par exemple).

✅ Vérifier que le Pod est en cours de création et/ou en cours d’exécution
```bash
kubectl get pods
```

### Accéder à la page par défaut de votre Pod Nginx

Maintenant que le Pod Nginx tourne dans Minikube, il est isolé dans le cluster Kubernetes et n’est pas accessible directement depuis la machine (sauf si on met en place un Service de type NodePort ou LoadBalancer, etc.). Pour l’instant, on va utiliser kubectl port-forward afin de rediriger un port local vers le port 80 du Pod

✅ Lancer la redirection de port
```bash
kubectl port-forward pod/my-nginx 8080:80

```

Cette commande redirige le port 8080 de la machine hôte vers le port 80 du Pod (celui où Nginx écoute par défaut).
- Laissez cette commande tourner dans un terminal.
- Accéder à la page Nginx
- Ouvrez le navigateur sur l’URL suivante :
```bash
http://localhost:8080

```

✅ Option 2 : Utiliser un manifeste YAML (bonne pratique)

Créer un fichier YAML, par exemple my-nginx-pod.yaml, avec le contenu suivant :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      ports:
        - containerPort: 80

```
- `apiVersion`: Indique la version de l’API Kubernetes (ici v1 pour un objet Pod).
- `kind`: Spécifie le type de ressource à créer (ici Pod).
- `metadata.name`: Nom du Pod (ici my-nginx).
- `spec.containers`: Liste des conteneurs dans ce Pod.
    - `name`: Nom symbolique du conteneur (ici nginx-container).
    - `image`: Image Docker utilisée (ici nginx:latest).
    - `ports`: Les ports exposés par le conteneur (ici containerPort: 80).

Appliquer le manifeste

```bash
kubectl apply -f my-nginx-pod.yaml
```
ensuite :
```bash
kubectl get pods
kubectl port-forward pod/my-nginx 8080:80
http://localhost:8080
```

## Explication de ce que nous venons de faire
### Création du Pod

- Kubernetes lance un conteneur (basé sur l’image nginx) dans un Pod nommé my-nginx.
- Ce Pod est géré par l’API de Kubernetes, ce qui vous permet de suivre son état et de communiquer avec lui via kubectl.

### Port forwarding

- Par défaut, un Pod n’a pas d’exposition sur votre machine locale (Minikube exécute un cluster “virtuel”).
- La commande kubectl port-forward crée un tunnel entre un port local (ici 8080) et un port dans le cluster (ici 80 sur le conteneur Nginx).
Cela permet de tester des services rapidement, sans avoir à configurer un Ingress ou un Service de type LoadBalancer ou NodePort.

### Visualisation dans un navigateur

- Avec la redirection de port en place, quand vous tapez http://localhost:8080, ça redirige la requête HTTP vers le Pod Nginx, qui renvoie sa page par défaut.

✅ Supprimer le Pod :

```bash
kubectl delete pod my-nginx

```


## Héberger deux Pods : `phpMyAdmin` et MySQL (avec Minikube)

### 1. Deployment MySQL

- Créez un fichier `mysql-deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:5.7
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "root"
            - name: MYSQL_DATABASE
              value: "tp_db"
            - name: MYSQL_USER
              value: "user"
            - name: MYSQL_PASSWORD
              value: "password"
          ports:
            - containerPort: 3306

```

- `apiVersion / kind` : On utilise ici un objet `Deployment` (API `apps/v1`).
- `metadata.name `: Nom du déploiement.
- `replicas` : Nombre de Pods identiques à lancer (1 pour le test).
- `selector/matchLabels` : Correspond aux labels du Pod, pour que le Deployment sache quels Pods gérer.
- `env` : Variables d’environnement MySQL (mêmes que dans Docker Compose).
- `ports` : On expose le port 3306 à l’intérieur du Pod (pour MySQL).

Appliquez ce manifeste :

```bash
kubectl apply -f mysql-deployment.yaml

```

### 2. Service MySQL

Créez un fichier `mysql-service.yaml` :
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  selector:
    app: mysql
  ports:
    - port: 3306      # Port sur lequel le service écoute à l'intérieur du cluster
      targetPort: 3306 # Port du conteneur MySQL
      protocol: TCP
  type: ClusterIP     # (par défaut) Service interne au cluster

```

- `selector` : Doit correspondre au label app: mysql défini dans le Deployment.
- `ports` : Indique le port du Service (ici 3306) et le targetPort (port du conteneur).
- `type` : ClusterIP est le type par défaut, qui rend le service accessible à l’intérieur du cluster (non accessible directement depuis l’extérieur).

```bash
kubectl apply -f mysql-service.yaml

```

### 3. Deployment `phpMyAdmin`

Créez un fichier `phpmyadmin-deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: phpmyadmin-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: phpmyadmin
  template:
    metadata:
      labels:
        app: phpmyadmin
    spec:
      containers:
        - name: phpmyadmin
          image: phpmyadmin/phpmyadmin
          env:
            - name: PMA_HOST
              value: "mysql-service"  # Le nom du Service MySQL
            - name: PMA_PORT
              value: "3306"
            - name: MYSQL_ROOT_PASSWORD
              value: "root"
          ports:
            - containerPort: 80
```

`PMA_HOST` : On met le nom du Service MySQL (`mysql-service`) au lieu de localhost.
`PMA_PORT` : On indique le port du Service (3306).
`MYSQL_ROOT_PASSWORD` : Permet de se connecter en root.
`ports` : Le conteneur `phpMyAdmin` écoute par défaut sur le port 80.

Appliquez ce manifeste :
```bash
kubectl apply -f phpmyadmin-deployment.yaml

```

Vérifiez que tout est bien créé :

```bash
kubectl get deployments
kubectl get pods
kubectl get services

```

## Créer un Service associé au Pod MySQL

C’est déjà fait ci-dessus via `mysql-service.yaml`.

Le Service s’appelle `mysql-service`.
Il permet à d’autres Pods du cluster (`phpMyAdmin`, par exemple) de se connecter à MySQL en utilisant l’hostname `mysql-service` et le port `3306`.

## Connecter `phpMyAdmin` avec le Service `MySQL`
Grâce aux variables d’environnement dans `phpmyadmin-deployment.yaml`, `phpMyAdmin` se connecte automatiquement à MySQL via `mysql-service:3306`.
```bash
PMA_HOST=mysql-service
PMA_PORT=3306
```

## Vérifier avec `kubectl port-forward` que `phpMyAdmin` peut administrer `MySQL`

Pour accéder à l’interface web de `phpMyAdmin` depuis le navigateur local, nous allons faire un `port-forward` sur le Pod ou sur le Deployment phpMyAdmin.

```bash
kubectl port-forward pod/phpmyadmin-deployment-64999b89b5-bwrcr 8081:80
```

- Accéder à phpMyAdmin : http://localhost:8081
Utilisateur : root
Mot de passe : root

Vous devriez voir la base `tp_db` et pouvoir exécuter des requêtes, créer des tables, etc.
