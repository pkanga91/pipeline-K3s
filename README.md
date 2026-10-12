# Pipeline CI/CD avec GitLab CI et k3s sur AWS

Mon projet pour m'entraîner à la CI/CD et à Kubernetes.

Le principe : je modifie une petite page web (servie par **Nginx**), je fais un `git push`, et **GitLab CI** teste, construit l'image Docker puis la déploie toute seule sur un cluster **k3s** installé sur une instance **EC2 AWS**.

> Ce projet est fait pour m'entraîner. Si vous voyez des choses à améliorer, n'hésitez pas à me le dire !

## Pourquoi ce projet

- Comprendre comment fonctionne une pipeline CI/CD de bout en bout
- Découvrir Kubernetes avec **k3s** (une version légère), sans avoir besoin d'un gros ordinateur
- Utiliser un **runner GitLab** installé sur ma propre machine (une EC2)

J'ai utilisé une EC2 parce que mon ordinateur n'est pas assez puissant pour faire tourner Kubernetes en local.

## Comment ça marche

```
 git push ──► GitLab CI ──► test ──► build ──► deploy
                              │        │         │
                              │        │         └─ kubectl met à jour l'application sur k3s
                              │        └─ docker construit l'image et l'envoie dans le registre GitLab
                              └─ vérifie la page et les fichiers Kubernetes

 Tout s'exécute sur mon runner GitLab, installé sur l'EC2 qui héberge k3s.
```

## Les outils

| Outil | À quoi il sert |
|---|---|
| GitLab CI/CD | Lancer la pipeline à chaque `git push` |
| GitLab Runner | La machine qui exécute les jobs (installée sur mon EC2, exécuteur `shell`) |
| Docker | Construire l'image de l'application |
| GitLab Container Registry | Stocker les images |
| k3s | Le cluster Kubernetes léger |
| Nginx | Le serveur web qui affiche ma page |
| AWS EC2 | La machine qui héberge k3s et le runner |

## Les fichiers du projet

```
.
├── index.html            # la page web
├── Dockerfile            # construit l'image (nginx + ma page)
├── k3s/
│   ├── deployment.yaml   # 2 pods nginx
│   └── service.yaml      # rend l'appli accessible (NodePort 30080)
├── .gitlab-ci.yml        # la pipeline
└── docs/                 # captures d'écran
```

## La pipeline en détail

Le fichier `.gitlab-ci.yml` a trois étapes (`stages`) :

1. **test** : je vérifie que ma page contient bien le texte attendu, et que mes fichiers Kubernetes sont valides (`kubectl apply --dry-run=client`).
2. **build** : je me connecte au registre GitLab, je construis l'image avec `docker build`, puis je l'envoie avec `docker push`.
3. **deploy** : j'applique les fichiers Kubernetes, je remplace l'image par la nouvelle avec `kubectl set image`, et j'attends que les pods soient prêts.

Quelques points que j'ai retenus :

- Chaque image est nommée avec le numéro du commit (`$CI_COMMIT_SHORT_SHA`), pour savoir quelle version tourne et pouvoir revenir en arrière.
- Les jobs `build` et `deploy` ne tournent que sur la branche `main` (grâce à `rules`).
- Le tag `k3s` sur chaque job indique que c'est **mon runner** qui doit l'exécuter.

## Mise en place

### 1. L'EC2 et k3s

Instance Ubuntu **t3.small** (2 Go de RAM) créée sur AWS, avec le port 30080 ouvert uniquement pour mon IP dans le groupe de sécurité.

```bash
curl -sfL https://get.k3s.io | sh -
sudo k3s kubectl get nodes          # le nœud doit être "Ready"
```

### 2. Docker pour le runner

```bash
sudo apt install -y docker.io
sudo usermod -aG docker gitlab-runner
```

### 3. Le runner GitLab

Dans GitLab : *Settings > CI/CD > Runners > New project runner* (tag `k3s`). Puis sur l'EC2 :

```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt install -y gitlab-runner
sudo gitlab-runner register --url https://gitlab.com --token MON_TOKEN --executor shell
```

(Le token est secret : il n'est pas dans le dépôt.)

### 4. Donner l'accès à k3s au runner

```bash
sudo mkdir -p /home/gitlab-runner/.kube
sudo cp /etc/rancher/k3s/k3s.yaml /home/gitlab-runner/.kube/config
sudo chown -R gitlab-runner:gitlab-runner /home/gitlab-runner/.kube
sudo -u gitlab-runner kubectl get nodes
```

### 5. Lancer la pipeline

Le projet GitLab doit être **public**, sinon k3s ne peut pas télécharger l'image depuis le registre.

```bash
git add .
git commit -m "Mon message"
git push origin main
```

## Vérifier que ça marche

```bash
kubectl get pods
curl http://localhost:30080
```

Depuis mon navigateur : `http://IP_PUBLIQUE_EC2:30080`.

Pour tester la pipeline, je change le texte dans `index.html`, je fais un `git push`, et après quelques minutes la nouvelle version s'affiche sans que j'aie rien fait sur k3s. On peut aussi revenir à la version précédente :

```bash
kubectl rollout undo deployment/myweb-application
```

![Pipeline GitLab](docs/pipeline.png)
![Application dans le navigateur](docs/appli.png)

## Ce qui m'a posé problème (et ce que j'ai appris)

- **`gitlab-runner: command not found`** : le runner n'était pas installé. J'ai dû ajouter le dépôt officiel de GitLab, puis l'installer avec `apt`.
- **`TLS handshake timeout` avec k3s** : mon instance était trop petite. k3s a besoin d'au moins 2 Go de RAM, donc je suis passé à une `t3.small`.
- **`permission denied` sur `/etc/rancher/k3s/k3s.yaml`** : l'utilisateur `gitlab-runner` ne peut pas lire ce fichier. J'ai copié le fichier dans son dossier `.kube` pour qu'il ait sa propre configuration.
- **Ma première version du `.gitlab-ci.yml` ne marchait pas** : il manquait les deux-points après les noms de jobs, et je pensais que `image:` et `docker:dind` fonctionnaient avec mon runner. En fait, avec l'exécuteur `shell`, les jobs tournent directement sur l'EC2, donc j'ai installé Docker dessus.
- **Le déploiement ne changeait pas la version** : `rollout restart` redémarre les mêmes pods. Il faut construire l'image, l'envoyer dans le registre, puis faire `kubectl set image`.
- **k3s n'utilise pas Docker** : une image construite en local n'est pas visible par k3s, d'où l'importance du registre.

Ce que je retiens : une pipeline, c'est surtout de la configuration qu'on lit et qu'on corrige à chaque erreur. Les messages des logs de GitLab m'ont beaucoup aidé.

## Limites (je les connais)

- Le test est très simple (un `grep` sur la page) : il ne remplace pas de vrais tests d'application.
- Le runner utilise l'exécuteur `shell` et l'utilisateur `gitlab-runner` est dans le groupe `docker` : c'est pratique pour un labo, mais pas recommandé en production.
- Pas de HTTPS, pas de nom de domaine, une seule machine.
- Le projet GitLab est public pour simplifier l'accès au registre.

## Idées pour la suite

- Remplacer la page par une petite application (Flask) avec de vrais tests
- Ajouter un scan de sécurité de l'image (Trivy)
- Utiliser Helm ou Kustomize pour les fichiers Kubernetes
- Ajouter un environnement de test avant la production

## Où est le projet

La pipeline tourne sur GitLab : (https://gitlab.com/pierre_kanga/pipeline-k3s). Ce dépôt GitHub contient une copie du code pour le portfolio.

## Nettoyage

Pour ne pas payer pour rien : arrêter ou supprimer l'EC2 depuis la console AWS, et supprimer le runner dans GitLab (*Settings > CI/CD > Runners*).

## Auteur

Projet réalisé par [pkanga91](https://github.com/pkanga91), dans le cadre de ma recherche de poste en DevOps / administration systèmes.
