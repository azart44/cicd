# Documentation du pipeline GitLab CI/CD pour le déploiement Terraform

## Contexte

Ce pipeline GitLab CI/CD est conçu pour déployer de l'infrastructure à l'aide de Terraform dans différents environnements (développement, staging et production) en fonction de la branche Git et du nom du job.

## Utilisation

Le pipeline est déclenché automatiquement lorsqu'un commit est poussé sur les branches `develop`, `feature/*`, `fix/*`, `main` ou `master`. Les jobs sont exécutés en fonction de la branche et des règles définies dans le fichier `.gitlab-ci.yml`.

## Environnements

Le pipeline prend en charge trois environnements :

1. Développement (dev) : Cet environnement est utilisé pour les branches de développement (`develop`, `feature/*`, `fix/*`). Il utilise l'espace de travail Terraform `hprod` et le fichier de variables `hprod.tfvars`.

2. Staging (stg) : Cet environnement est utilisé pour les branches de production (`main` ou `master`) lorsque le nom du job contient ":stg". Il utilise l'espace de travail Terraform `stg` et le fichier de variables `stg.tfvars`.

3. Production (prod) : Cet environnement est utilisé pour les branches de production (`main` ou `master`) lorsque le nom du job ne contient pas ":stg". Il utilise l'espace de travail Terraform `prod` et le fichier de variables `prod.tfvars`.

## Étapes du pipeline

Le pipeline est composé de quatre étapes principales :

1. `Vérification du code` : Cette étape exécute une analyse SonarQube du code source.

2. `Planification du déploiement` : Cette étape initialise Terraform, sélectionne l'espace de travail approprié et exécute un plan Terraform. Les artifacts du plan sont conservés pendant 30 minutes.

3. `Déploiement de l'infrastructure` : Cette étape initialise Terraform, sélectionne l'espace de travail approprié et applique le plan Terraform. Cette étape est déclenchée manuellement.

4. `Destruction de l'infrastructure` : Cette étape initialise Terraform, sélectionne l'espace de travail approprié et détruit l'infrastructure. Cette étape est déclenchée manuellement.

## Dépendances entre les jobs

Le pipeline utilise des règles de dépendances pour s'assurer que les jobs de production ne s'exécutent que si les jobs de staging correspondants ont réussi :

- `plan:stg` dépend de `sonarqube-check`.
- `plan:prd` dépend de `plan:stg`.
- `deploy:stg` dépend de `plan:stg`.
- `deploy:prd` dépend de `plan:prd`.
- `destroy:prd` dépend de `destroy:stg`.

## Variables et configurations

Le pipeline utilise plusieurs variables et configurations :

- `TF_ROOT` : Le répertoire racine des configurations Terraform.
- `AWS_ROLE` : Le rôle IAM AWS utilisé pour l'authentification.
- `TF_MODULE_USR` et `TF_MODULE_TOKEN` : Les informations d'identification pour accéder aux modules Terraform.
- `AWS_ACCOUNT_ID` : L'ID du compte AWS, défini en fonction de l'environnement.
- `TF_VAR_ci_projet_name` : Le nom du projet GitLab CI/CD.

Les fichiers de variables Terraform (`hprod.tfvars`, `stg.tfvars`, `prod.tfvars`) doivent être placés dans le répertoire `env/` à la racine des configurations Terraform.

## Tags et exécuteurs

Le pipeline utilise le tag `gitlab-runner-aws-niv-oniiva` pour spécifier l'exécuteur GitLab Runner à utiliser pour les jobs Terraform. L'étape `sonarqube-check` utilise également ce tag pour s'exécuter sur le même exécuteur.

## Image Docker

Le pipeline utilise une image Docker personnalisée (`306241591911.dkr.ecr.eu-west-1.amazonaws.com/agt_deploy_terraform:1.5.6`) qui contient les outils nécessaires pour exécuter Terraform et interagir avec AWS. Cette image est définie au début du fichier `.gitlab-ci.yml`.

## before_script

La section `before_script` du pipeline effectue plusieurs tâches de configuration avant l'exécution des jobs :

1. Détermination de l'environnement en fonction de la branche et du nom du job.
2. Configuration de l'ID du compte AWS en fonction de l'environnement.
3. Configuration du nom du projet GitLab CI/CD comme variable Terraform.
4. Configuration de l'URL Git pour accéder aux modules Terraform.
5. Changement de répertoire vers `TF_ROOT`.