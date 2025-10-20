## Atelier : Découverte des outils d'administration

**Objectif :** Se familiariser avec les outils du DBA MariaDB et visualiser des bases existantes.

### Exercice : Installation de MariaDB

**Sur Linux (Debian/Ubuntu) :**
```bash
# Installation
sudo apt update
sudo apt install mariadb-server mariadb-client

# Sécurisation
sudo mysql_secure_installation

# Vérification
mysql --version
sudo systemctl status mariadb
```

**Validez l'installation en vous connectant :**
```bash
mysql -u root -p
```

### Exercice 1.1 : Exploration avec le client mysql
```bash
# Connexion au serveur
mysql -u root -p

# Une fois connecté, exécuter :
SHOW DATABASES;
USE mysql;
SHOW TABLES;
DESCRIBE user;
SELECT Host, User FROM user;
```

### Exercice 1.2 : Utilisation de mysqladmin
```bash
# Vérifier le statut du serveur
mysqladmin -u root -p status

# Afficher les variables système
mysqladmin -u root -p variables

# Lister les processus actifs
mysqladmin -u root -p processlist
```

### Exercice 1.3 : Découverte de PHPMyAdmin

**Installation de PHPMyAdmin et apache2:**
```bash
sudo apt install apache2
sudo apt install libapache2-mod-php
sudo apt install phpmyadmin
```
* Sur l'écran de configuration de phpmyadmin, sélectionner apache2 (avec la barre d'espace) puis allez sur OK
* A la question "Créer la base de données phpmyadmin :" oui
* Entrer un mot de passe pour phpmyadmin (à conserver)
* Puis indiquer le mot de passe root de mariadb

1. Accéder à l'interface web de PHPMyAdmin
2. Explorer la structure de la base système `mysql`
3. Examiner la table `mysql.user`
4. Visualiser les privilèges d'un utilisateur
5. Exécuter une requête simple dans l'onglet SQL

### Exercice 1.4 : Extraction d'informations système
```sql
-- Afficher la version de MariaDB
SELECT VERSION();

-- Lister les moteurs de stockage disponibles
SHOW ENGINES;

-- Consulter les variables de configuration
SHOW VARIABLES LIKE 'datadir';
SHOW VARIABLES LIKE 'port';

-- Visualiser les bases de données existantes
SELECT 
    SCHEMA_NAME,
    DEFAULT_CHARACTER_SET_NAME,
    DEFAULT_COLLATION_NAME
FROM information_schema.SCHEMATA;
```

**Questions à explorer :**
- Où sont stockées les données sur le disque ?
- Quel est le port d'écoute du serveur ?
- Quels sont les moteurs de stockage disponibles ?
- Combien d'utilisateurs sont configurés ?
