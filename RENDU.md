# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :*DIENG Cheikh*
**Groupe :*ING3 RS2*

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   >Il affiche **3 nœuds** : **`k3d-lab-server-0`** qui a le rôle **`control-plane`** (le nœud maître qui orchestre et pilote le cluster), et **`k3d-lab-agent-0`** ainsi que **`k3d-lab-agent-1`** (rôle `<none>`) qui sont les deux **workers** chargés d'exécuter les pods applicatifs.
2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   >`docker ps` affiche **4 conteneurs Docker** : les 3 conteneurs exécutant `rancher/k3s` (`k3d-lab-server-0`, `k3d-lab-agent-0`, `k3d-lab-agent-1`) ainsi qu'un conteneur proxy `k3d-lab-serverlb` (load balancer). Concrètement, dans **k3d**, un nœud Kubernetes est simplement un **conteneur Docker** faisant tourner une instance légère de Kubernetes (k3s).

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   >La commande a créé **3 objets** : un **Deployment** (`deployment.apps/web`), un **ReplicaSet** (`replicaset.apps/web-68d995574f`), et un **Pod** (`pod/web-...`) hébergeant le conteneur Nginx. *(Le `service/web` visible dans `kubectl get all` a quant à lui été créé par `kubectl expose`)*.
4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   >L'ancien pod (`web-68d995574f-88lc6`) est passé en statut `Terminating`, et un **nouveau pod** (`web-68d995574f-lvvz2`) a été créé et démarré (`Running`) automatiquement. Cela s'explique par la **boucle de réconciliation** (auto-réparation) de Kubernetes : le **ReplicaSet** surveille en permanence l'état du cluster et recrée immédiatement un pod pour maintenir le nombre de réplicas souhaité (1).

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   >**Non**, le message a disparu et la page d'origine *« Welcome to nginx! »* est revenue. Le système de fichiers d'un conteneur est **éphémère** : lorsqu'un pod est supprimé, le ReplicaSet en recrée un tout neuf à partir de l'image Docker `nginx` d'origine, sans conserver les modifications effectuées manuellement à l'intérieur.
6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   >Même s'il est lancé en ciblant le Service (`svc/web`), `kubectl port-forward` établit en réalité un tunnel TCP direct vers **un pod spécifique** choisi au moment de son lancement. Lorsque ce pod est supprimé, son interface réseau se ferme (`network namespace ... is closed` / `lost connection to pod`) et le tunnel casse immédiatement (`Exit 1`) sans basculer automatiquement vers le nouveau pod.

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   >**`kubectl create`** est une commande **impérative** qui crée une ressource une seule fois : si elle existe déjà, la commande échoue (`AlreadyExists`). **`kubectl apply`** est une commande **déclarative** et **idempotente** : elle lit l'état désiré dans un manifeste YAML (versionnable dans Git), crée la ressource si elle n'existe pas, ou la met à jour sur place si elle a été modifiée.
8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   >Le `selector` permet au Service d'identifier dynamiquement vers quels pods  il doit router le trafic réseau. S'il ne correspond à aucun pod, le Service est créé mais ne possède aucun **Endpoint** (`kubectl get endpoints site` affiche `<none>`), et toute requête envoyée vers le Service échoue (connexion refusée / timeout).

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   Kubernetes a réparti les 3 réplicas de façon équilibrée à raison d'**un pod par nœud** (`k3d-lab-agent-0`, `k3d-lab-server-0` et `k3d-lab-agent-1`)
   site-75775484b-7nlzm   1/1     Running   0          116s   10.42.1.5   k3d-lab-agent-0    <none>           <none>
   site-75775484b-dp4xg   1/1     Running   0          116s   10.42.0.7   k3d-lab-server-0   <none>           <none>
   site-75775484b-rhfwr   1/1     Running   0          10m    10.42.2.7   k3d-lab-agent-1    <none>           <none>
   
10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
    >Le nouveau pod est passé au statut **`ErrImagePull`** puis **`ImagePullBackOff`** (`0/1 Ready`). C'est la commande **`kubectl describe pod -l app=site`** (via la section **`Events`** en bas de la sortie) qui a indiqué la cause exacte : l'image/le tag `nginx:9.99` est introuvable sur le registre Docker Hub (*`Failed to pull image "nginx:9.99": ... not found`*).
11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    >**Oui**, le site est resté parfaitement accessible sans aucune interruption de service. Lors d'un **Rolling Update**, Kubernetes applique une stratégie de déploiement progressif : tant que le nouveau pod ne passe pas à l'état `Ready` (ici bloqué sur `ImagePullBackOff`), il conserve les anciens pods sains (`nginx:1.28`) en cours d'exécution et le Service continue de router le trafic uniquement vers eux.

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    >https://redesigned-engine-xgwvrpqrq64c9vw-8585.app.github.dev/
13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    >Oui, `kubectl get deploy mario` affiche exactement le Deployment créé depuis Portainer. Portainer et `kubectl` communiquent tous deux avec le même composant central du cluster : l'**API Server Kubernetes (`kube-apiserver`)**.

## Bonus (facultatif)

14. Bonus réalisés et remarques :
    >- **Bonus A (Ingress)** : Exposition du Service `site` via Traefik (`k8s/bonus/ingress.yaml`) sur le port `8081`. Contrairement au `port-forward`, le site reste accessible sans coupure lors de la suppression d'un pod car Traefik route dynamiquement vers les Endpoints sains.
    >- **Bonus B (Probes & Ressources)** : Ajout d'une `readinessProbe` HTTP sur `/` (port `80`) et de `resources` (`requests` 50m/32Mi, `limits` 100m/64Mi) dans `k8s/site/deployment.yaml`. Si la sonde pointe vers un chemin inexistant (`/nope`), le pod reste `Running` mais `0/1 Ready` et est exclu des Endpoints du Service (aucun trafic ne lui est envoyé).
    >- **Bonus C (Maintenance de nœud)** : Exécution de `kubectl drain` sur `k3d-lab-agent-0` : le pod hébergé a été évincé et recréé sur un autre nœud disponible. Après `kubectl uncordon`, le nœud redevient planifiable (`Ready`), mais les pods existants ne reviennent pas automatiquement car Kubernetes ne déplace pas des pods sains sans événement de replanification.
    >- **Bonus D (Namespaces)** : Création du namespace `prod` et déploiement d'une copie isolée de `k8s/site/` (`kubectl apply -n prod -f k8s/site/`), observable séparément dans Portainer.

---

**Score obtenu avec `./scripts/check.sh` :** 18 / 18
