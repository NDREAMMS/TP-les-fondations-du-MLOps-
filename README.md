# TP - Les fondations du MLOps

Ce projet met en place une petite boucle de machine learning opérationnelle avec ClearML :

- un modèle de classification d’images est servi via Gradio,
- les utilisateurs corrigent ou valident les prédictions,
- les faux positifs / labels utiles sont stockés comme feedback,
- dès qu’un lot de feedback est suffisant, un dataset est publié,
- un trigger ClearML lance automatiquement une pipeline de retraining,
- le nouveau modèle est comparé à la version de production et éventuellement promu.

## Vue d’ensemble de la pipeline

Le flux est scindé en 3 notebooks principaux :

1. NB-A : inference + feedback utilisateur
2. NB-B : déclenchement automatique
3. NB-C : pipeline de retraining

### 1) NB-A — Inference + feedback (Gradio)

Le notebook A lance une interface Gradio pour tester un modèle ResNet18 sur des images CIFAR-10.

À chaque prédiction, l’utilisateur peut choisir de :
- valider la prediction,
- corriger le label,
- ou ignorer la donnée.

Les images annotées sont sauvegardées localement dans un dossier de type :

- ~/clearml_feedback/feedback/<true_label>/uuid.png

Chaque fois qu’un utilisateur confirme/corrige une image, le système met à jour un compteur local. Quand ce compteur atteint 5 échantillons, une nouvelle version du dataset `image-feedback` est créée dans ClearML et publiée avec le tag `auto-feedback`.

Cela sert de signal d’activation pour le reste de la chaîne MLOps.

### 2) NB-B — Trigger ClearML

Le notebook B configure un `TriggerScheduler` / watcher côté ClearML.

Son rôle est de surveiller le projet `Image-Classifier/Gradio-Retrain-Demo-IABD` et de détecter la publication d’une nouvelle version du dataset `image-feedback` portant le tag `auto-feedback`.

Quand un nouveau dataset est détecté :
- le watcher récupère son ID,
- il clone la pipeline `retrain_pipeline`,
- il l’enqueue dans la queue `default`.

Autrement dit, le dataset devient le déclencheur de l’entraînement automatique.

### 3) NB-C — Pipeline de retraining

Le notebook C contient une pipeline ClearML construite avec `PipelineDecorator`.

La pipeline est découpée en 3 étapes :

#### a. prepare_data
- cherche la dernière version du dataset `image-feedback`,
- récupère son contenu localement,
- renvoie le chemin du dossier de feedback.

Si aucun feedback n’existe, la pipeline continue sur le jeu de base CIFAR-10.

#### b. train
- charge le dataset CIFAR-10 de base depuis ClearML,
- charge les images de feedback (si disponibles),
- concatène les datasets pour former l’ensemble d’entraînement,
- entraîne un ResNet18 sur CPU ou GPU selon l’environnement,
- calcule la précision sur validation,
- sauvegarde le modèle et le publie comme modèle candidat dans ClearML.

Le modèle nouvellement entraîné reçoit des tags comme :
- `candidate`
- `resnet18`

#### c. register_model
- récupère le modèle de production actuel (tag `production`),
- compare son accuracy avec celui du nouveau candidat,
- si le candidat est meilleur, il est promu en production,
- sinon il est marqué en staging.

La logique de promotion est simple :

- meilleur score → `production`
- score inférieur → `staging`

## Flux complet

Le flux réel est donc :

1. l’utilisateur utilise l’UI Gradio,
2. une image est prédite,
3. le label réel est validé ou corrigé,
4. le feedback est stocké,
5. après 5 éléments, un dataset est publié dans ClearML,
6. le trigger détecte le dataset,
7. la pipeline de retraining se lance,
8. le modèle est entraîné sur CIFAR-10 + feedback,
9. le nouveau modèle est comparé au modèle en prod,
10. le meilleur modèle est promu.

## Artifacts produits

Au cours du projet, on génère typiquement :

- un dataset `image-feedback` versionné dans ClearML,
- des modèles candidates dans ClearML,
- un modèle promu en production,
- des métriques d’entraînement (accuracy, logs, outputs),
- une trace d’exécution complète dans l’UI ClearML.

## Intérêt de cette architecture

Cette approche permet de construire un vrai cycle de feedback d’apprentissage continu :

- les usages réels alimentent les données,
- les données déclenchent l’automatisation,
- la pipeline réentraîne le modèle,
- la qualité du modèle est évaluée de façon reproductible,
- la chaîne de livraison reste traçable via ClearML.

## Commandes / outils utilisés

Le projet repose sur :

- Python
- PyTorch / torchvision
- Gradio
- ClearML
- ClearML PipelineDecorator
- ClearML Agent pour l’exécution des jobs sur queue

## Résumé

La pipeline n’est pas seulement un entraînement ponctuel : c’est une boucle de feedback continue entre les prédictions utilisateur, les données de correction, les datasets de feedback, le retraining automatique et la publication d’un modèle amélioré.

# TP-les-fondations-du-MLOps-
