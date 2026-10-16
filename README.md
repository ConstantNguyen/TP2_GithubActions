# TP2 - GitHub Actions (CI/CD et SonarCloud)

## Objectif

Mettre en place un pipeline GitHub Actions qui, à chaque push :
1. compile et teste l'application (`simple-api`) ;
2. analyse la qualité du code avec SonarCloud ;
3. construit les images Docker (`simple-api`, `database`, `http-server`) et les publie sur Docker Hub depuis la branche `main`.

## Structure du dépôt

- `.github/workflows/main.yml` : définition du pipeline
- `simple-api/` : application Java Spring Boot
- `database/` : image PostgreSQL
- `http-server/` : image du serveur httpd (reverse proxy)
- `docker-compose.yml` : lancement de l'ensemble en local

## Pipeline

| Job | Rôle | Déclenchement |
|---|---|---|
| `test-backend` | Build Maven, tests unitaires et d'intégration, analyse SonarCloud | push sur `main`, pull request |
| `build-and-push-docker-image` | Build des 3 images Docker, push sur Docker Hub | après `test-backend`, push réel uniquement sur `main` |

Secrets GitHub utilisés : `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `SONAR_TOKEN`.


### 2-1 : Qu'est-ce que Testcontainers ?

Testcontainers est une bibliothèque Java qui démarre des conteneurs Docker éphémères pendant l'exécution des tests. Ici, elle lance un conteneur `postgres:12.0` auquel l'application se connecte. Les tests d'intégration tournent donc sur une vraie base de données, identique à celle de production, sans rien installer à la main. Le conteneur est supprimé automatiquement à la fin des tests (via le conteneur Ryuk). Docker doit être lancé pour que les tests fonctionnent.

### 2-2 : Pour quelle raison utilise-t-on des variables sécurisées (secrets) ?

Pour ne jamais écrire d'identifiants (mot de passe, token) en clair dans le code. Le dépôt est versionné et potentiellement lisible par d'autres, et un secret commité reste dans l'historique Git même après suppression. Les secrets GitHub sont chiffrés, accessibles uniquement pendant l'exécution du workflow et masqués dans les logs. En cas de fuite, il suffit de révoquer le token sans modifier le code.

### 2-3 : Pourquoi `needs: test-backend` sur le job de build Docker ?

Par défaut, les jobs d'un workflow s'exécutent en parallèle. `needs` force le job de build et push à attendre la réussite du job de test. Si les tests échouent, aucune image n'est construite ni publiée, ce qui évite de livrer du code cassé. Sans `needs`, les deux jobs démarreraient en même temps et une image pourrait être publiée alors que les tests sont rouges. (Le nom du job dans notre fichier est `test-backend`, alors que l'énoncé parle de `build-and-test-backend`.)

### 2-4 : Pour quelle raison pousse-t-on les images Docker ?

Le runner GitHub est une machine temporaire, détruite à la fin du job avec les images qu'elle contient. Le push les envoie dans un registre (Docker Hub) pour les conserver. Elles peuvent alors être récupérées avec `docker pull` par un serveur de déploiement ou un collègue, et lancées à l'identique sans reconstruction. C'est la base de la livraison continue : produire à chaque commit validé sur `main` un artefact prêt à être déployé.

## Quality Gate SonarCloud

SonarCloud analyse le code à chaque exécution du pipeline (bugs, vulnérabilités, code smells, couverture de tests, duplications) et applique un Quality Gate pour empêcher du code de mauvaise qualité d'arriver sur `main`.

Résultats observés :

| Métrique | Résultat |
|---|---|
| Reliability | A (0 bug) |
| Maintainability | A (0 code smell) |
| Security | D (5 vulnérabilités à examiner) |
| Hotspots reviewed | 100 % |
| Coverage | 92,1 % |
| Duplications | 0,0 % |

## Lancer le projet en local

TP3 dans un autre github : [Lien vers le TP3](https://github.com/ConstantNguyen/TP3_Ansible)