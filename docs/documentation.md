# TP Final DevOps : Stratégie et Mise en Œuvre

Ce document décrit la stratégie DevOps mise en place pour le projet **Christmas Gift List**, réalisé dans le cadre du TP final du module DevOps (M1 Cyber). 

L'objectif est de présenter toutes les étapes nécessaires pour passer d'un code local à une application entièrement conteneurisée, déployable sur différents environnements, avec une pipeline CI/CD automatisée.  

### Environnement

- **Système hôte :** Ubuntu 24.04.3 (machine virtuelle)  
- **Contributeurs :** **Yassine ESSAOURI** et **Ahmed ENNOURI**
- **Dépôt de base :** [Anthony-Jhoiro/2025-devops-tp-final](https://github.com/Anthony-Jhoiro/2025-devops-tp-final.git)

### Configuration initiale

Avant de commencer, préparez votre environnement avec les commandes suivantes :

```bash
# Mettre à jour les paquets
sudo apt update

# Installer Docker et Git
sudo apt install -y docker.io git

# Activer et démarrer Docker
sudo systemctl enable --now docker

# (Optionnel) Ajouter votre utilisateur au groupe docker pour éviter d'utiliser sudo à chaque commande
sudo usermod -aG docker $USER
```
> **📌 Note :** Après avoir ajouté votre utilisateur au groupe `docker`, déconnectez-vous et reconnectez-vous pour que les changements prennent effet.

Vérifiez que tout fonctionne correctement :
```bash
docker --version
git --version
```

💡 **Astuce :** Vous pouvez tester Docker sans sudo avec :
```bash
docker run hello-world
```


## Partie 1 – Gestion du code avec Git

### 1.1 Introduction

Le projet utilise Git pour la gestion du code, le versioning et le workflow CI/CD.  

L’objectif est de permettre un travail collaboratif efficace, avec des branches claires pour la production, la pré-production et les nouvelles fonctionnalités.

### 1.2 Fork du dépôt

Le projet ne doit pas être modifié au niveau du code, mais nous devons mettre en place une infrastructure DevOps complète.  
Pour cela, nous avons effectué un **fork** du dépôt GitHub original.

Commandes utilisées :

```bash
git clone https://github.com/EssYassine/2025-devops-tp-final.git
cd 2025-devops-tp-final/
```

Ensuite, nous avons ajouté le dépôt original comme upstream afin de pouvoir synchroniser si nécessaire :

```bash
git remote add upstream https://github.com/Anthony-Jhoiro/2025-devops-tp-final.git
git fetch upstream
```

### 1.3 Remotes configurés

Après cette configuration, les remotes du projet sont :

- origin → fork personnel (dépôt principal de travail)

- upstream → dépôt original (lecture seule)

Pour vérifier :
```bash
git remote -v
```

### 1.4 Branches et workflow

Pour ce projet, nous avons mis en place une branche principale `main` et une branche de pré-production `develop`.  

> **📌 Note :** Les branches `feature/*` seront créées à partir de `develop` lorsque nous aurons besoin d’ajouter des configurations DevOps ou des tests supplémentaires.

#### Branches utilisées

Le projet utilise une stratégie simple mais adaptée à un pipeline CI/CD :

| Branche             | Description                                                                                  | Règles / Remarques                                                                 |
|--------------------|----------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------|
| main             | Branche de production                                                                        | Le code déployé en production provient toujours de `main`. Protégée : merge uniquement via PR + pipeline vert. |
| develop          | Branche de pré-production / staging                                                          | Déploiement automatique vers l'environnement de test. Toute feature doit être fusionnée ici avant d’aller en production. |
| feature/<nom>    | Branches temporaires pour toute nouvelle fonctionnalité ou ajout de configuration DevOps     | Exemple : `feature/add-ci`, `feature/docker-compose`.                             |

#### Commandes utilisées

1. Création de la branche `develop` localement :
```bash
git checkout -b develop
```

2. Pousser develop sur le fork et configurer le suivi :
```bash
git push -u origin develop
```

3. Vérification des branches locales et distantes :
```bash
git branch -a
```

Exemple de sortie :
```bash
* develop
  main
  remotes/origin/HEAD -> origin/main
  remotes/origin/develop
  remotes/origin/main
  remotes/upstream/main
```

4. Exemple de création d’une future branche feature :
```bash
git checkout -b feature/<nom>
git push -u origin feature/<nom>
```

> **📌 Notes :** 
- Pour l’instant, aucune branche feature n’a encore été créée.
- Tout développement doit passer par develop avant de fusionner dans main.
- Les merges vers main déclencheront la création d’un tag Git et d’une release GitHub.

#### Workflow recommandé

1. Créer une branche feature :
```bash
git checkout -b feature/<nom>
```

2. Travailler et push :
```bash
git push -u origin feature/<nom>
```

3. Ouvrir une Pull Request vers develop.

4. Fusionner develop vers main après validation pour déclencher le déploiement en production.

### 1.5 Releases GitHub

À chaque merge dans main :

- Création d’un tag Git (v1.0.0, v1.1.0, …)

- Création d’une release GitHub

- Mise à jour automatique des images Docker Hub
