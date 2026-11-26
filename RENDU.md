# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :** Glouzouic Mateo
**Groupe :** ⚠️ À COMPLÉTER

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   > 3 nœuds, tous `Ready` :
   > - `k3d-lab-server-0` (rôles `control-plane,master`) : le plan de contrôle. Il héberge l'API Server, le scheduler, le controller-manager et la base d'état (etcd/SQLite dans k3s). C'est lui qui décide où placer les pods et qui maintient l'état désiré.
   > - `k3d-lab-agent-0` et `k3d-lab-agent-1` (rôle `<none>`) : les workers. Ils exécutent les pods des applications grâce au kubelet et au runtime de conteneurs (containerd).
   >
   > Sur un cluster « vide », `kubectl get pods -A` montre déjà des pods système dans le namespace **`kube-system`** : `coredns` (DNS interne), `traefik` et `svclb-traefik` (Ingress Controller et load balancer), `metrics-server`, `local-path-provisioner` (stockage), ainsi que les jobs `helm-install-traefik*`.
2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   > `docker ps` montre des conteneurs Docker : `k3d-lab-server-0`, `k3d-lab-agent-0`, `k3d-lab-agent-1` (image `rancher/k3s`), plus `k3d-lab-serverlb`, le load balancer qui publie le port `8081->80`, et éventuellement `k3d-lab-tools`.
   > Concrètement, dans k3d, **un nœud Kubernetes est un conteneur Docker** dans lequel tourne k3s. On simule donc une « machine » du cluster par un conteneur. Les pods tournent à leur tour dans le containerd de ce conteneur (conteneurs dans un conteneur).

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   > Un **Deployment** (`deployment.apps/web`), qui crée un **ReplicaSet** (`replicaset.apps/web-<hash>`), qui crée à son tour un **Pod** (`pod/web-<hash>-<id>`). Le **Service** `service/web` vient lui de `kubectl expose` (le service `kubernetes` existe par défaut).
4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   > Un nouveau pod a été recréé immédiatement, **avec un autre nom** (nouveau suffixe aléatoire). L'ancien passe en `Terminating` pendant que le nouveau passe par `Pending`, puis `ContainerCreating`, puis `Running`. C'est le **ReplicaSet** qui fait ça : il doit maintenir 1 réplica et constate un écart entre l'état désiré (1 pod) et l'état réel (0 pod). C'est la **boucle de réconciliation** de Kubernetes (auto-réparation).

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   > **Non.** On retrouve la page *Welcome to nginx!*. Le système de fichiers d'un conteneur est **éphémère** : la modification a été faite dans la couche inscriptible du conteneur de l'ancien pod. Le nouveau pod repart de l'image `nginx` d'origine, et la modification est perdue avec l'ancien pod.
6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   > Un `kubectl port-forward svc/web` ne passe pas vraiment par le Service : au lancement, il choisit **un seul pod** derrière le Service et ouvre le tunnel directement vers lui. Quand ce pod est supprimé, le tunnel n'a plus de cible et s'arrête (erreur visible dans `/tmp/web.log`). Il ne bascule pas automatiquement sur le nouveau pod, il faut le relancer.

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   > - `kubectl create` est **impératif** : « crée cet objet ». Il échoue (`AlreadyExists`) si l'objet existe déjà et ne sait pas mettre à jour.
   > - `kubectl apply -f` est **déclaratif** : « voici l'état désiré ». Il crée l'objet s'il n'existe pas, sinon il calcule la différence et ne modifie que ce qui a changé (annotation `last-applied-configuration`). Il est **idempotent** : on peut le relancer autant de fois qu'on veut, et les fichiers YAML versionnés dans Git deviennent la source de vérité.
8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   > Le `selector` indique **quels pods reçoivent le trafic** du Service : tous les pods dont les labels correspondent (ici `app: site`) sont ajoutés à ses **endpoints**, et le trafic est réparti entre eux. S'il ne correspond à aucun pod, le Service existe mais n'a **aucun endpoint** (`kubectl get endpoints site` est vide) : les connexions échouent et le `port-forward` ne trouve pas de pod. C'est ce qui arrivait avec `app: TODO`. De la même façon, un Deployment dont le `selector` ne correspond pas aux labels de son template est refusé (`selector does not match template labels`).

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   ```
   NAME                    READY   STATUS    RESTARTS   AGE    IP          NODE               NOMINATED NODE   READINESS GATES
   site-54dbc4f754-22fxc   1/1     Running   0          86s    10.42.1.9   k3d-lab-agent-1    <none>           <none>
   site-54dbc4f754-gxw8s   1/1     Running   0          119s   10.42.0.5   k3d-lab-server-0   <none>           <none>
   site-54dbc4f754-rpkr4   1/1     Running   0          96s    10.42.2.7   k3d-lab-agent-0    <none>           <none>
   ```
   > Un réplica par nœud : `k3d-lab-agent-0`, `k3d-lab-agent-1` et `k3d-lab-server-0`. Le scheduler répartit les pods sur les nœuds pour équilibrer la charge et limiter l'impact de la perte d'un nœud. Le control plane reçoit aussi un pod, car dans k3s le nœud server n'a pas de « taint » qui l'interdirait.
10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
    > Le nouveau pod restait en **`ErrImagePull`**, puis en **`ImagePullBackOff`** (READY `0/1`). La cause exacte apparaît avec **`kubectl describe pod -l app=site`**, dans la section *Events* en bas : `Failed to pull image "nginx:9.99" ... manifest for nginx:9.99 not found` (le tag n'existe pas sur Docker Hub).
11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    > **Oui.** Le Deployment fait un **rolling update** : il crée d'abord un pod avec la nouvelle version et ne supprime les anciens que lorsque le nouveau est prêt (`maxUnavailable` = 25 %, arrondi à 0 avec 3 réplicas, et `maxSurge` = 25 %). Le nouveau pod ne devenant jamais prêt, la mise à jour reste bloquée et les anciens pods en `nginx:1.28` continuent de servir le site. Le Service n'envoie le trafic qu'aux pods prêts. `kubectl rollout undo deploy/site` est ensuite revenu à la révision précédente, puis `kubectl diff -f k8s/site/` a permis de vérifier que le cluster correspond de nouveau aux fichiers.

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    > ⚠️ À COMPLÉTER : `https://<nom-du-codespace>-8585.app.github.dev/`
13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    > Oui : `kubectl get deploy mario -o yaml` montre le Deployment créé depuis Portainer. C'est un Deployment Kubernetes standard, identique sur le fond à ceux créés en ligne de commande. Portainer ajoute seulement quelques labels et annotations à son nom (`io.portainer.kubernetes.application...`). Portainer et `kubectl` parlent tous les deux au **même composant : l'API Server** (kube-apiserver) du control plane. Portainer, qui tourne dans le cluster, l'appelle avec un ServiceAccount qui a les droits nécessaires. `kubectl` l'appelle avec le kubeconfig.

## Bonus (facultatif)

14. Bonus réalisés et remarques :
    > - **A, Ingress** : `k8s/bonus/ingress.yaml` envoie `/` vers le Service `site`, port 80, via Traefik, accessible sur le port 8081. Contrairement au `port-forward`, l'Ingress passe par le Service, qui répartit le trafic entre tous les pods prêts : si on supprime des pods, le site **reste accessible**.
    > - **B, Probes et ressources** : ajout d'une `readinessProbe` HTTP sur `/` (port 80) et de `resources` (requests 50m CPU / 32Mi, limits 100m CPU / 64Mi). Si la probe pointe vers `/nope`, nginx répond 404 : les pods restent `Running` mais **jamais `Ready` (0/1)**. Ils ne sont pas ajoutés aux endpoints du Service, et un rolling update vers cette config se bloque (les anciens pods sont conservés).
    > - **C, Drain** : `kubectl drain` marque le nœud `SchedulingDisabled` et expulse ses pods, que les ReplicaSets recréent sur les autres nœuds. Après `uncordon`, ils **ne reviennent pas tout seuls** : le scheduler ne place que les nouveaux pods et ne rééquilibre jamais les pods déjà en place. Il faut par exemple un `kubectl rollout restart deploy/site` pour les redistribuer.

---

**Score obtenu avec `./scripts/check.sh` :** ⚠️ À COMPLÉTER ... / 18
