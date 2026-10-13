# Kubernetes avancé — Helm, Ingress, HPA, NetworkPolicy (pfe-tracker)

Chart Helm packageant l'application `pfe-tracker` (Node.js/Express + PostgreSQL, réutilisée des Projets 1 et 2) avec quatre fonctionnalités Kubernetes natives : gestion de paquet (**Helm**), exposition HTTP (**Ingress**), autoscaling (**HPA**), et isolation réseau "zero trust" (**NetworkPolicy**). Déployé sur le même cluster local `kind` que le Projet 2, dans un namespace dédié.

## Objectif du projet

Démontrer la maîtrise de fonctionnalités Kubernetes au-delà du simple `kubectl apply` de manifests bruts (Projet 2) : packaging réutilisable, exposition contrôlée, élasticité automatique, et posture de sécurité réseau explicite plutôt qu'implicite.

## Pourquoi découplé du GitOps du Projet 2

Le Projet 2 démontrait le **pattern GitOps** (CI → Git → ArgoCD → cluster). Ce projet démontre des **fonctionnalités Kubernetes natives** — un sujet différent. Déploiement volontairement en direct via `helm install`/`upgrade`, sans ArgoCD, pour garder chaque projet lisible sur son propre sujet.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│ cluster kind (local)                                  │
│                                                         │
│  namespace ingress-nginx        namespace pfe-tracker-k8s
│  ┌──────────────────┐           ┌─────────────────────┐
│  │ ingress-nginx-    │  routes   │  Ingress             │
│  │ controller         │ ───────► │  (host: pfe-tracker.local)
│  └──────────────────┘           │        │               │
│                                   │        ▼               │
│                                   │  Service app ──► pods app (1-5, HPA)
│                                   │        │               │
│                                   │        ▼               │
│                                   │  Service postgres ──► pod postgres
│                                   │                         │
│                                   │  NetworkPolicy :        │
│                                   │  default-deny-all +     │
│                                   │  3 règles explicites     │
│                                   └─────────────────────┘
└─────────────────────────────────────────────────────┘
```

## Helm — packaging de l'application

Chart (`pfe-tracker-chart`) remplaçant les manifests YAML bruts du Projet 2 par des templates paramétrables via `values.yaml` : images, réplicas, hôte Ingress, seuils HPA. Un seul `helm install`/`upgrade` au lieu de 5 `kubectl apply` séparés, avec rollback natif (`helm rollback`) en cas de problème.

## Ingress — exposition HTTP contrôlée

**ingress-nginx** reçoit le trafic HTTP externe et route selon le host déclaré (`pfe-tracker.local`) vers le Service `app` — équivalent Kubernetes de l'ALB du Projet 1. Remplace le `kubectl port-forward` direct utilisé au Projet 2 pour les tests.

> Sur `kind`, le nœud doit porter le label `ingress-ready=true` pour que le contrôleur puisse être planifié (voir incidents).

## HPA — autoscaling horizontal

Le `HorizontalPodAutoscaler` fait varier le nombre de replicas du pod `app` (1 à 5) selon le CPU réel, mesuré par **metrics-server**. Nécessite que le conteneur déclare des `resources.requests.cpu` — sans ça, le HPA n'a rien à comparer.

**Test réel** : une charge générée via 20 flux HTTP concurrents a fait passer le CPU de ~1% à 34%, dépassant le seuil de 50%/replica et déclenchant un scale-up automatique **1 → 5 replicas**. Arrêt de la charge → scale-down progressif automatique vers 1 replica.

## NetworkPolicy — isolation réseau "zero trust"

Par défaut, Kubernetes autorise tout trafic entre tous les pods d'un cluster. Une policy `default-deny-all` bloque tout par défaut dans le namespace, puis trois règles explicites autorisent uniquement : `app` → `postgres` (port 5432), trafic entrant depuis `ingress-nginx` → `app` (port 8080, et uniquement depuis ce namespace précis), et la résolution DNS (port 53, indispensable sinon même la résolution de noms de service casse).

**Test réel** : un pod de test (`netpol-test`), non autorisé, n'a pas pu établir de connexion TCP vers `postgres` (`Connection timed out`), mais a pu résoudre son nom DNS normalement — preuve que l'isolation est fine (bloque le trafic applicatif, pas le DNS), pas un simple "tout ou rien".

## Déploiement réel et incidents rencontrés

1. **Cluster `kind` arrêté après redémarrage WSL** : le conteneur Docker du nœud était `Exited`, pas supprimé — `docker start pfe-gitops-control-plane` a suffi à tout faire repartir (ArgoCD et l'app du Projet 2 inclus).
2. **Magasin de certificats racine WSL cassé** : `curl`/`apt` échouaient en SSL sur tous les hôtes HTTPS — corrigé par `apt-get install --reinstall ca-certificates && update-ca-certificates`.
3. **Pod ingress-nginx bloqué en `Pending`** : le manifeste `kind` exige un nœud labellisé `ingress-ready=true`, absent sur ce cluster (créé sans config spéciale). Corrigé par `kubectl label node ... ingress-ready=true`.
4. **`ImagePullBackOff` sur l'app** : `values.yaml` référençait le tag `latest`, jamais poussé par le pipeline CI du Projet 2 (qui ne pousse que des tags basés sur le hash de commit). Corrigé en récupérant le vrai tag déployé (`kubectl get deployment ... -o jsonpath=...`) et `helm upgrade --set app.image.tag=...`.
5. **Générateur de charge HPA bloqué sans erreur apparente** : un pod de test dans le même namespace que l'app n'a jamais pu la joindre — pas un bug, c'est la NetworkPolicy qui bloquait un trafic non autorisé. La charge a dû être générée depuis l'extérieur, via le chemin Ingress (seul chemin autorisé en entrée).
6. **Conflit de port local (8080)** : un service Tomcat local (autre projet) occupait déjà le port 8080, provoquant des réponses trompeuses lors des tests `curl`. Résolu en utilisant le port 9090 pour le `port-forward`.

## Reproduire ce projet

```bash
kind create cluster --name pfe-gitops   # ou réutiliser un cluster existant
kubectl label node <nom-du-nœud> ingress-ready=true
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.2/deploy/static/provider/kind/deploy.yaml
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

kubectl create namespace pfe-tracker-k8s
helm install pfe-tracker . -n pfe-tracker-k8s --set app.image.tag=<tag-existant>
```

## Stack technique

Kubernetes (`kind`) · Helm · ingress-nginx · metrics-server · HorizontalPodAutoscaler · NetworkPolicy · Node.js/Express · PostgreSQL
