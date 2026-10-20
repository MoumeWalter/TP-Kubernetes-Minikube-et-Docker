# Réponses de synthèse

## 1. Différence entre un Pod autonome et un Pod géré par un Deployment
Un Pod autonome n'est surveillé par aucun contrôleur : s'il est supprimé ou s'il tombe, il disparaît définitivement. Nous l'avons constaté avec `whoami-pod` : après `kubectl delete pod whoami-pod`, `kubectl get pods` n'affichait plus rien. Un Pod géré par un Deployment est piloté par un ReplicaSet : il est recréé automatiquement, peut être multiplié (`kubectl scale`) et le Deployment gère aussi les mises à jour progressives.

## 2. Rôle du ReplicaSet dans l'auto-réparation
Le ReplicaSet compare en permanence le nombre de Pods portant les labels de son sélecteur (`app=whoami`) avec le nombre de réplicas souhaité (3). Quand un Pod est supprimé, il en manque un : le ReplicaSet en crée un nouveau. Lors de notre test, le Pod `whoami-c6f799d48-4nwwt` est passé en `Terminating` et `whoami-c6f799d48-zgckc` a été créé aussitôt (Pending, ContainerCreating, Running), ce qui a rétabli 3 Pods.

## 3. Rôle du Service et vérification du ciblage
Un Service fournit une adresse IP et un nom DNS stables, ainsi qu'un équilibrage de charge, devant des Pods dont les IP changent. Il sélectionne les Pods grâce à son `selector` (`app: whoami`). Pour vérifier qu'il cible les bons Pods, on exécute `kubectl get endpoints whoami-service` : les adresses listées (10.244.0.10, 10.244.0.11 et 10.244.0.12, port 80) doivent correspondre aux IP des Pods affichées par `kubectl get pods -o wide`. Si la liste est vide, il faut vérifier les labels, les réplicas et l'état des Pods.

## 4. Pourquoi charger l'image dans Minikube et utiliser imagePullPolicy: Never
L'image `monapp:1.0` a été construite dans le Docker de la machine hôte. Elle n'existe ni dans le nœud Minikube ni dans un registre public, donc Kubernetes ne peut pas la télécharger. `minikube image load monapp:1.0` la copie dans le nœud. `imagePullPolicy: Never` demande au kubelet de ne jamais tenter de téléchargement et d'utiliser l'image déjà présente localement. Sans cela, le Pod échouerait avec une erreur de téléchargement (ErrImagePull).
