
# 🧩 TP Final – Mise en production d’un service MariaDB

## ⏱ Durée : 60 à 90 minutes

### 🎯 Objectifs
- Mettre en œuvre l’ensemble des compétences d’administration MariaDB vues en formation.
- Choisir les bons moteurs de stockage, gérer les utilisateurs et les droits, configurer et optimiser le serveur.
- Mettre en place une sauvegarde/restauration et produire un rapport technique.

---

## 🔍 Contexte
Vous êtes DBA pour une startup de e-commerce. Le service "commande & facturation" doit être mis en production sur un serveur MariaDB préinstallé.
La configuration actuelle n’est pas optimisée.

Votre mission : rendre la base prête pour la production, en respectant les bonnes pratiques de sécurité, de performance et de maintenance.

---

## 🛠 Travail demandé

### 1. Modélisation et moteurs de stockage
- Créez la base `ecom_db`.
- Créez les tables suivantes :
  - `orders` : enregistrement des commandes (id, date, montant).
  - `order_items` : détail des articles par commande.
  - `product_cache` : cache temporaire des produits.
  - `reporting_orders` : table pour reporting (lecture intensive).
- Choisissez le moteur de stockage adapté pour chaque table (`InnoDB`, `MyISAM`, `MEMORY`, etc.) et justifiez vos choix.
- Insérez environ 2 000 enregistrements dans `orders` et `order_items`.
- Vérifiez les moteurs via `SHOW TABLE STATUS`.

### 2. Utilisateurs et droits
- Créez les utilisateurs :
  - `app_user` : accès complet à `orders` et `order_items` sauf suppression.
  - `report_user` : lecture seule sur `reporting_orders`.
  - `audit_user` : lecture seule sur toutes les bases, uniquement depuis `localhost`.
- Vérifiez et documentez les privilèges avec `SHOW GRANTS`.

### 3. Configuration et optimisation
- Observez les variables suivantes :
  ```sql
  SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
  SHOW VARIABLES LIKE 'query_cache_size';
  SHOW VARIABLES LIKE 'max_connections';
  ```
- Proposez deux ajustements en fonction de la RAM estimée (ex. 4 GB).
- Appliquez temporairement vos modifications avec `SET GLOBAL`.
- Testez une requête lourde avant/après pour observer l’impact.

### 4. Sauvegarde et restauration
- Effectuez un dump complet de `ecom_db` :
  ```bash
  mysqldump -u root -p ecom_db > ecom_db_dump.sql
  ```
- Supprimez une table puis restaurez le dump pour valider la restauration.
- Décrivez dans votre rapport votre stratégie de sauvegarde et la fréquence recommandée.

### 5. Rapport de synthèse
Rédigez un rapport (max. 1 page) présentant :
- Les moteurs choisis et justifications.
- Les utilisateurs et leurs privilèges.
- Les paramètres modifiés et effets observés.
- La stratégie de sauvegarde.
- Les points de vigilance avant mise en production.

---

## ✅ Critères de réussite
- Actions exécutées correctement (création, configuration, sauvegarde).
- Choix techniques justifiés.
- Rapport clair et synthétique.
