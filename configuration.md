# Atelier : Installation et configuration de MariaDB

**Objectif :** Configurer MariaDB tester l'accès depuis différents clients.



### Exercice : Changement du port d'écoute

Par défaut, MariaDB écoute sur le port 3306. Nous allons le modifier pour utiliser le port 3307.

**Étape 1 : Modifier la configuration**
```bash
# Éditer le fichier de configuration
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf

# Modifier ou ajouter la ligne :
port = 3307
```

**Étape 2 : Redémarrer le service**
```bash
sudo systemctl restart mariadb
```

**Étape 3 : Vérifier le changement**
```bash
# Vérifier que le port 3307 est en écoute
sudo ss -tlnp | grep 3307
```

**Étape 4 : Tester la connexion**
```bash
# Connexion en spécifiant le port
mysql -u root -p --port=3307

# Ou
mysql -u root -p -P 3307
```

### Exercice : Création d'un utilisateur pour l'accès distant

```sql
-- Se connecter à MariaDB
mysql -u root -p -P 3307

-- Créer un utilisateur avec accès depuis n'importe quelle adresse
CREATE USER 'admin_demo'@'%' IDENTIFIED BY 'MotDePasse123!';

-- Accorder tous les privilèges
GRANT ALL PRIVILEGES ON *.* TO 'admin_demo'@'%' WITH GRANT OPTION;

-- Appliquer les changements
FLUSH PRIVILEGES;

-- Vérifier la création
SELECT User, Host FROM mysql.user WHERE User = 'admin_demo';
```

**Modifier bind-address pour autoriser les connexions distantes :**
```bash
# Éditer la configuration
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf

# Modifier ou commenter la ligne bind-address
# bind-address = 127.0.0.1
bind-address = 0.0.0.0

# Redémarrer
sudo systemctl restart mariadb
```

### Exercice : Test depuis le client en ligne de commande

```bash
# Test local avec le nouveau port
mysql -u admin_demo -p -P 3307

# Une fois connecté, exécuter :
SHOW DATABASES;
SELECT USER(), CURRENT_USER();
SHOW VARIABLES LIKE 'port';
```

### Exercice  : Configuration de PHPMyAdmin pour le nouveau port




**Modifier la configuration de PHPMyAdmin :**
```bash
# Éditer le fichier de configuration
sudo nano /etc/phpmyadmin/config.inc.php

# Modifier la section serveur
$cfg['Servers'][$i]['host'] = 'localhost';
$cfg['Servers'][$i]['port'] = '3307';
$cfg['Servers'][$i]['connect_type'] = 'tcp';
```

**Redémarrer le serveur web :**
```bash
sudo systemctl restart apache2
```

**Se connecter via PHPMyAdmin :**
1. Accéder à l'interface web
2. Utiliser les identifiants : `admin_demo` / `MotDePasse123!`
3. Vérifier que la connexion fonctionne
4. Explorer les bases de données système

### Exercice 2.6 : Test avec un outil graphique (DBeaver ou MySQL Workbench)

**Installer DBeaver :**

https://dbeaver.io/download/

**Configuration de la connexion dans DBeaver :**
1. Nouvelle connexion → MariaDB
2. Paramètres :
   - Host : localhost (ou l'IP du serveur)
   - Port : 3307
   - Database : (laisser vide)
   - Username : admin_demo
   - Password : MotDePasse123!
3. Tester la connexion
4. Explorer la structure des bases

**Requêtes de validation :**
```sql
-- Afficher les informations de connexion
SELECT 
    CONNECTION_ID(),
    USER(),
    DATABASE(),
    VERSION();

-- Lister les bases
SHOW DATABASES;

-- Vérifier la configuration du port
SHOW VARIABLES LIKE 'port';
SHOW VARIABLES LIKE 'bind_address';

-- Afficher les utilisateurs configurés
SELECT User, Host, plugin FROM mysql.user;
```

### Questions de validation

1. Sur quel port votre serveur MariaDB écoute-t-il maintenant ?
2. Pouvez-vous vous connecter depuis PHPMyAdmin ?
3. Quelle commande permet de vérifier les ports en écoute sur votre système ?
4. Pourquoi est-il important de modifier `bind-address` pour autoriser les connexions distantes ?
5. Quels sont les risques d'utiliser `bind-address = 0.0.0.0` en production ?
