# TP Kubernetes — Premiers déploiements avec Minikube et Docker

## 1. Membres du binôme
- [Nom Prénom 1]
- [Nom Prénom 2]

## 2. Environnement
| Outil | Version |
|-------|---------|
| Système | Windows, terminal Git Bash |
| Docker | Client 28.5.1, Docker Desktop 4.48.0 (moteur 28.5.1, containerd 1.7.27) |
| kubectl | v1.34.1 (Kustomize v5.7.1) |
| Minikube | v1.39.0 (installé avec `winget install Kubernetes.minikube`) |
| Kubernetes (cluster) | v1.37.0, runtime containerd 2.3.4 |

## 3. Principales étapes d'exécution

### 3.1 Cluster
Commandes : `minikube start --driver=docker`, `minikube status`, `docker ps`, `kubectl cluster-info`, `kubectl get nodes -o wide`, `kubectl get pods -A`.
- Nœud : `minikube`, rôle control-plane, statut Ready, IP interne 192.168.49.2. Conteneur Docker associé : `gcr.io/k8s-minikube/kicbase:v0.0.51`.
- Composants système observés dans `kube-system` : `kube-apiserver-minikube`, `etcd-minikube`, `coredns-559f6c778d-gnj2w`. Également présents : `kube-scheduler-minikube`, `kube-controller-manager-minikube`, `kube-proxy`, `kindnet` et `storage-provisioner`.

### 3.2 Namespace et Pod
- Création du namespace `demo` et configuration comme namespace courant (`kubectl config set-context --current --namespace=demo`).
- Pod `whoami-pod` (image traefik/whoami:v1.10.3, port 80) : statut Running, IP 10.244.0.3, nœud minikube. L'image a été téléchargée en environ 5,6 s (voir `kubectl describe pod`).
- Le nom d'hôte du conteneur a été obtenu avec `kubectl port-forward pod/whoami-pod 8082:80` puis `curl http://localhost:8082` : `Hostname: whoami-pod`.
- Après `kubectl delete pod whoami-pod`, `kubectl get pods` répond « No resources found in demo namespace » : le Pod autonome n'est pas recréé.

### 3.3 Deployment (3 réplicas)
- `kubectl apply -f manifests/deployment.yaml` : Deployment `whoami` 3/3, ReplicaSet `whoami-c6f799d48`, 3 Pods Running. Labels des Pods : `app=whoami,pod-template-hash=c6f799d48`. Stratégie : RollingUpdate (25 % max unavailable, 25 % max surge).
- Auto-réparation : suppression du Pod `whoami-c6f799d48-4nwwt` observée avec `kubectl get pods -w`. Il passe en Terminating, et le Pod `whoami-c6f799d48-zgckc` est créé aussitôt (Pending, ContainerCreating, Running). Il y a de nouveau 3 Pods Running.
- Mise à l'échelle : 5 réplicas, puis 0, puis 3, avec vérification par `kubectl rollout status deployment/whoami`.

### 3.4 Service
- `whoami-service` : type ClusterIP, IP 10.106.171.217, port 80/TCP, sélecteur `app: whoami`, `targetPort: http`.
- `kubectl get endpoints whoami-service` : 10.244.0.10:80, 10.244.0.11:80 et 10.244.0.12:80, soit les 3 Pods. (Kubernetes affiche un avertissement : l'API Endpoints est dépréciée au profit de EndpointSlice.)
- Test : `kubectl port-forward service/whoami-service 8083:80` puis `curl http://localhost:8083`, réponse `Hostname: whoami-c6f799d48-bqdxp` (IP 10.244.0.12, un des endpoints).

### 3.5 Image personnalisée
- `monapp/index.html` (titre et message en français) et `monapp/Dockerfile` (`FROM nginx:1.27`, puis `COPY index.html /usr/share/nginx/html/index.html`).
- `docker build -t monapp:1.0 ./monapp` : build réussi. Puis `minikube image load monapp:1.0`, et l'image `docker.io/library/monapp:1.0` apparaît dans `minikube image ls`.
- Deployment `monapp` : 2 réplicas, `imagePullPolicy: Never`. Les Pods `monapp-664cfbcf46-5ntgd` et `monapp-664cfbcf46-l98jm` sont Running (1/1).
- Test : `kubectl port-forward deployment/monapp 8081:80`. Le navigateur affiche à `http://localhost:8081` le titre « Bonjour depuis Kubernetes ! » et le message en français.

### 3.6 Nettoyage
Suppression du Service, des Deployments et du Pod, puis du namespace `demo`. Retour au namespace `default` et `minikube stop`.

## 4. Preuves
| Fichier | Contenu |
|---------|---------|
| preuves/01-cluster.png | Nœud minikube Ready et Pods système en Running |
| preuves/02-deployment.png | Deployment whoami 3/3, ReplicaSet et 3 Pods Running |
| preuves/03-autoreparation.png | Pod 4nwwt supprimé, remplacé par zgckc ; 3 Pods Running |
| preuves/04-service-http.png | Endpoints du Service et réponse HTTP via port-forward (8083) |
| preuves/05-image-personnalisee.png | 2 Pods monapp Running et page française sur le port 8081 |

## 5. Difficultés rencontrées et solutions
1. **Minikube absent** : `minikube version` donnait « command not found ». Installation avec winget (v1.39.0). La commande restait introuvable car le PATH n'était pas rechargé. Solution : fermer et rouvrir toutes les fenêtres Git Bash.
2. **Port-forward lancé avant le Service** : l'erreur `services "whoami-service" not found` venait de l'ordre des étapes. Il faut appliquer `service.yaml` avant le `port-forward`.
3. **Port 8080 occupé** : il est utilisé par le conteneur Airflow d'un autre projet (`tp-airflow-airflow-apiserver-1`). Le `port-forward` demandé par le TP sur 8080 a donc été remplacé par un autre port local.
4. **Pas de shell dans l'image whoami** : `kubectl exec -it whoami-pod -- sh` échoue (`exec: "sh": executable file not found in $PATH`), car l'image traefik/whoami est minimale. Le nom d'hôte a été obtenu via `port-forward` et `curl`.
5. **Conversion de chemin sous Git Bash** : `kubectl exec whoami-pod -- /whoami` transformait `/whoami` en `C:/Program Files/Git/whoami`. Contournement : `MSYS_NO_PATHCONV=1`. La commande relançait ensuite un second serveur (« listen tcp :80: bind: address already in use ») puisque le port 80 était déjà utilisé par le serveur du Pod.
6. **Port 8082 encore occupé** : un ancien `port-forward` du Pod était resté ouvert (erreur « Only one usage of each socket address »). Le Service a été testé sur le port 8083.
7. **Fichier de manifeste vide** : le premier `kubectl apply -f monapp-deployment.yaml` a répondu « error: no objects passed to apply », car le fichier créé avec `touch` n'était pas rempli ni enregistré. Il a été complété puis appliqué.

## 6. Réponses aux questions
Voir `reponses.md`.
