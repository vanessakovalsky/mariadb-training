# Atelier  : Gestion des utilisateurs et sécurité

**Objectif :** Créer des utilisateurs, configurer les accès distants, tester les privilèges et diagnostiquer les problèmes de connexion.

### Exercice 1.1 : Autorisation des connexions distantes

**Étape 1 : Vérifier la configuration réseau**
```bash
# Vérifier bind-address
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf

# S'assurer que bind-address permet les connexions distantes
# bind-address = 0.0.0.0  (toutes les interfaces)
# ou commenter la ligne

# Redémarrer MariaDB
sudo systemctl restart mariadb

# Vérifier que le port est accessible
sudo netstat -tlnp | grep 3306
```

**Étape 2 : Configurer le firewall**
```bash
# UFW (Ubuntu/Debian)
sudo ufw allow 3306/tcp
sudo ufw reload

# Firewalld (CentOS/RHEL)
sudo firewall-cmd --permanent --add-port=3306/tcp
sudo firewall-cmd --reload
```

### Exercice 1.2 : Création d'utilisateurs avec différents niveaux d'accès

```sql
-- Connexion en tant que root
mysql -u root -p

-- Créer une base de données de test
CREATE DATABASE test_security;
USE test_security;

-- Créer des tables de test
CREATE TABLE clients (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nom VARCHAR(100),
    email VARCHAR(100),
    telephone VARCHAR(20),
    solde DECIMAL(10,2)
);

CREATE TABLE commandes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT,
    date_commande DATE,
    montant DECIMAL(10,2),
    statut VARCHAR(20),
    FOREIGN KEY (client_id) REFERENCES clients(id)
);

-- Insérer des données de test
INSERT INTO clients (nom, email, telephone, solde) VALUES
('Alice Martin', 'alice@example.com', '0612345678', 1000.00),
('Bob Dupont', 'bob@example.com', '0687654321', 500.50),
('Charlie Bernard', 'charlie@example.com', '0698765432', 250.75);

INSERT INTO commandes (client_id, date_commande, montant, statut) VALUES
(1, '2025-01-15', 150.00, 'Livrée'),
(2, '2025-01-16', 75.50, 'En cours'),
(3, '2025-01-17', 200.00, 'En attente');
```

**Créer différents types d'utilisateurs :**

```sql
-- 1. Administrateur local (tous les droits)
CREATE USER 'admin_local'@`localhost` IDENTIFIED BY 'AdminPass123!';
GRANT ALL PRIVILEGES ON *.* TO 'admin_local'@'localhost' WITH GRANT
-- 2. Utilisateur applicatif (lecture/écriture sur test_security)
CREATE USER 'app_user'@`%` IDENTIFIED BY 'AppPass456!';
GRANT SELECT, INSERT, UPDATE, DELETE ON test_security.* TO 'app_user'@'%';

-- 3. Utilisateur en lecture seule
CREATE USER 'readonly_user'@`%` IDENTIFIED BY 'ReadPass789!';
GRANT SELECT ON test_security.* TO 'readonly_user'@'%';

-- 4. Utilisateur avec accès limité à certaines colonnes
CREATE USER 'marketing_user'@`%` IDENTIFIED BY 'MarketPass123!';
GRANT SELECT (id, nom, email) ON test_security.clients TO 'marketing_user'@'%';

-- 5. Utilisateur depuis un réseau spécifique
CREATE USER 'network_user'@`192.168.1.%` IDENTIFIED BY 'NetworkPass456!';
GRANT SELECT, INSERT, UPDATE ON test_security.* TO 'network_user'@'192.168.1.%';

-- 6. Utilisateur avec SSL obligatoire
CREATE USER 'secure_user'@`%` 
IDENTIFIED BY 'SecurePass789!' 
REQUIRE SSL;
GRANT SELECT ON test_security.* TO 'secure_user'@'%';

-- 7. Utilisateur avec limitations de ressources
CREATE USER 'limited_user'@`%` 
IDENTIFIED BY 'LimitPass123!'
WITH MAX_QUERIES_PER_HOUR 100
     MAX_CONNECTIONS_PER_HOUR 10
     MAX_USER_CONNECTIONS 2;
GRANT SELECT ON test_security.* TO 'limited_user'@'%';

-- Appliquer les changements
FLUSH PRIVILEGES;

-- Vérifier les utilisateurs créés
SELECT User, Host, plugin FROM mysql.user WHERE User LIKE '%_user' OR User LIKE 'admin_%';
```

### Exercice 1.3 : Tests de connexion depuis différentes sources

**Test 1 : Connexion locale**
```bash
# Test avec admin_local
mysql -u admin_local -p -h localhost
# Entrer : AdminPass123!

# Une fois connecté
SELECT USER(), CURRENT_USER();
SHOW GRANTS;
```

**Test 2 : Connexion distante**
```bash
# Depuis une autre machine (remplacer SERVER_IP)
mysql -u app_user -p -h SERVER_IP
# Entrer : AppPass456!

# Test des privilèges
USE test_security;
SELECT * FROM clients;
INSERT INTO clients (nom, email, telephone, solde) 
VALUES ('Test User', 'test@example.com', '0600000000', 100.00);
```

**Test 3 : Vérification des restrictions**
```bash
# Connexion readonly_user
mysql -u readonly_user -p -h SERVER_IP

USE test_security;
SELECT * FROM clients;  -- OK

# Essayer une insertion (devrait échouer)
INSERT INTO clients (nom, email) VALUES ('Fail', 'fail@example.com');
-- ERROR: INSERT command denied

# Essayer une mise à jour (devrait échouer)
UPDATE clients SET solde = 2000 WHERE id = 1;
-- ERROR: UPDATE command denied
```

**Test 4 : Accès limité aux colonnes**
```bash
# Connexion marketing_user
mysql -u marketing_user -p -h SERVER_IP

USE test_security;

# Accès autorisé aux colonnes spécifiées
SELECT id, nom, email FROM test_security.clients;  -- OK

# Accès refusé aux autres colonnes
SELECT solde FROM test_security.clients;
-- ERROR: SELECT command denied
```

### Exercice 1.4 : Test des opérations selon les privilèges

**Créer un script de test SQL :**
```sql
-- test_privileges.sql
USE test_security;

-- Test SELECT
SELECT 'Test SELECT' AS test;
SELECT COUNT(*) FROM clients;

-- Test INSERT
SELECT 'Test INSERT' AS test;
INSERT INTO clients (nom, email, telephone, solde) 
VALUES ('New Client', 'new@example.com', '0611111111', 500.00);

-- Test UPDATE
SELECT 'Test UPDATE' AS test;
UPDATE clients SET solde = solde + 100 WHERE id = 1;

-- Test DELETE
SELECT 'Test DELETE' AS test;
DELETE FROM clients WHERE nom = 'New Client';

-- Test CREATE
SELECT 'Test CREATE' AS test;
CREATE TABLE test_table (id INT);

-- Test DROP
SELECT 'Test DROP' AS test;
DROP TABLE IF EXISTS test_table;
```

**Exécuter avec différents utilisateurs :**
```bash
# Avec app_user (devrait réussir SELECT, INSERT, UPDATE, DELETE)
mysql -u app_user -p -h SERVER_IP < test_privileges.sql

# Avec readonly_user (seul SELECT devrait réussir)
mysql -u readonly_user -p -h SERVER_IP < test_privileges.sql

# Observer les erreurs pour les opérations non autorisées
```

### Exercice 1.5 : Problèmes classiques d'erreurs de connexion

**Problème 1 : Access denied for user**

**Simulation du problème :**
```bash
# Essayer de se connecter avec un mauvais mot de passe
mysql -u app_user -p -h SERVER_IP
# Entrer un mauvais mot de passe
# ERROR 1045 (28000): Access denied for user 'app_user'@'...' (using password: YES)
```

**Diagnostic :**
```sql
-- Vérifier que l'utilisateur existe
SELECT User, Host FROM mysql.user WHERE User = 'app_user';

-- Vérifier depuis quelle IP la connexion arrive
-- Dans les logs : /var/log/mysql/error.log

-- Solution : Réinitialiser le mot de passe
ALTER USER 'app_user'@'%' IDENTIFIED BY 'NewAppPass456!';
```

**Problème 2 : Host not allowed**

**Simulation :**
```bash
# Créer un utilisateur limité à une IP
CREATE USER 'restricted'@'192.168.1.100' IDENTIFIED BY 'RestrictedPass!';

# Essayer de se connecter depuis une autre IP
mysql -u restricted -p -h SERVER_IP
# ERROR 1130 (HY000): Host '192.168.1.50' is not allowed to connect
```

**Solution :**
```sql
-- Option 1 : Créer un compte pour la nouvelle IP
CREATE USER 'restricted'@'192.168.1.50' IDENTIFIED BY 'RestrictedPass!';
GRANT SELECT ON test_security.* TO 'restricted'@'192.168.1.50';

-- Option 2 : Modifier pour accepter le sous-réseau
RENAME USER 'restricted'@'192.168.1.100' TO 'restricted'@'192.168.1.%';
```

**Problème 3 : Can't connect to MySQL server**

**Causes possibles :**
```bash
# 1. Service non démarré
sudo systemctl status mariadb
sudo systemctl start mariadb

# 2. Firewall bloque le port
sudo ufw status
sudo ufw allow 3306/tcp

# 3. bind-address incorrect
sudo grep bind-address /etc/mysql/mariadb.conf.d/50-server.cnf
# Doit être 0.0.0.0 ou l'IP du serveur

# 4. Port incorrect
netstat -tlnp | grep mysql
# Vérifier que MySQL écoute sur le bon port
```

**Problème 4 : Too many connections**

**Simulation :**
```sql
-- Voir le nombre maximum de connexions
SHOW VARIABLES LIKE 'max_connections';

-- Voir les connexions actives
SHOW PROCESSLIST;

-- Compter les connexions par utilisateur
SELECT User, Host, COUNT(*) as nb_connexions
FROM information_schema.PROCESSLIST
GROUP BY User, Host;
```

**Solution :**
```sql
-- Solution temporaire : Augmenter max_connections
SET GLOBAL max_connections = 200;

-- Solution permanente : Modifier my.cnf
-- [mysqld]
-- max_connections = 200

-- Tuer des connexions inactives
SELECT ID, USER, HOST, TIME, COMMAND, STATE 
FROM information_schema.PROCESSLIST 
WHERE COMMAND = 'Sleep' AND TIME > 300;

-- Tuer une connexion spécifique
KILL <process_id>;
```

**Problème 5 : Account locked**

```sql
-- Vérifier si le compte est verrouillé
SELECT User, Host, account_locked FROM mysql.user WHERE User = 'app_user';

-- Déverrouiller
ALTER USER 'app_user'@'%' ACCOUNT UNLOCK;
```

### Exercice 1.6 : Observer les chaînes de connexion

**Exemple PHP avec trace de connexion :**
```php
<?php
// config_db.php
$config = [
    'host' => '192.168.1.10',
    'port' => 3306,
    'database' => 'test_security',
    'username' => 'app_user',
    'password' => 'AppPass456!',
    'charset' => 'utf8mb4'
];

// Afficher la configuration (ATTENTION : Ne jamais faire en production !)
echo "Configuration de connexion :\n";
echo "Hôte : " . $config['host'] . "\n";
echo "Port : " . $config['port'] . "\n";
echo "Base : " . $config['database'] . "\n";
echo "Utilisateur : " . $config['username'] . "\n";
echo "Mot de passe : " . str_repeat('*', strlen($config['password'])) . "\n\n";

try {
    $dsn = sprintf(
        "mysql:host=%s;port=%d;dbname=%s;charset=%s",
        $config['host'],
        $config['port'],
        $config['database'],
        $config['charset']
    );
    
    echo "DSN : " . $dsn . "\n\n";
    
    $pdo = new PDO($dsn, $config['username'], $config['password'], [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC
    ]);
    
    echo "Connexion réussie !\n\n";
    
    // Informations de connexion
    $info = $pdo->query("SELECT 
        USER() as user_connecte, 
        CURRENT_USER() as user_effectif,
        DATABASE() as base_actuelle,
        VERSION() as version_mariadb")->fetch();
    
    echo "Utilisateur connecté : " . $info['user_connecte'] . "\n";
    echo "Utilisateur effectif : " . $info['user_effectif'] . "\n";
    echo "Base de données : " . $info['base_actuelle'] . "\n";
    echo "Version : " . $info['version_mariadb'] . "\n\n";
    
    // Test de requête
    $stmt = $pdo->query("SELECT COUNT(*) as nb FROM clients");
    $result = $stmt->fetch();
    echo "Nombre de clients : " . $result['nb'] . "\n";
    
} catch(PDOException $e) {
    echo "ERREUR DE CONNEXION : " . $e->getMessage() . "\n";
    echo "Code d'erreur : " . $e->getCode() . "\n";
}
?>
```

**Exécuter le script :**
```bash
php config_db.php
```

**Exemple Python :**
```python
import mysql.connector
from mysql.connector import Error

def test_connection():
    config = {
        'host': '192.168.1.10',
        'port': 3306,
        'database': 'test_security',
        'user': 'app_user',
        'password': 'AppPass456!',
        'charset': 'utf8mb4'
    }
    
    print("Configuration de connexion :")
    print(f"Hôte : {config['host']}")
    print(f"Port : {config['port']}")
    print(f"Base : {config['database']}")
    print(f"Utilisateur : {config['user']}")
    print(f"Mot de passe : {'*' * len(config['password'])}\n")
    
    try:
        conn = mysql.connector.connect(**config)
        
        if conn.is_connected():
            print("Connexion réussie !\n")
            
            cursor = conn.cursor(dictionary=True)
            
            # Informations de connexion
            cursor.execute("""
                SELECT 
                    USER() as user_connecte, 
                    CURRENT_USER() as user_effectif,
                    DATABASE() as base_actuelle,
                    VERSION() as version_mariadb
            """)
            
            info = cursor.fetchone()
            print(f"Utilisateur connecté : {info['user_connecte']}")
            print(f"Utilisateur effectif : {info['user_effectif']}")
            print(f"Base de données : {info['base_actuelle']}")
            print(f"Version : {info['version_mariadb']}\n")
            
            # Test de requête
            cursor.execute("SELECT COUNT(*) as nb FROM clients")
            result = cursor.fetchone()
            print(f"Nombre de clients : {result['nb']}")
            
            cursor.close()
            
    except Error as e:
        print(f"ERREUR DE CONNEXION : {e}")
        print(f"Code d'erreur : {e.errno}")
    
    finally:
        if conn and conn.is_connected():
            conn.close()
            print("\nConnexion fermée.")

if __name__ == "__main__":
    test_connection()
```

**Exemple Java :**
```java
import java.sql.*;

public class TestMariaDBConnection {
    public static void main(String[] args) {
        String url = "jdbc:mariadb://192.168.1.10:3306/test_security";
        String user = "app_user";
        String password = "AppPass456!";
        
        System.out.println("Configuration de connexion :");
        System.out.println("URL : " + url);
        System.out.println("Utilisateur : " + user);
        System.out.println("Mot de passe : " + "*".repeat(password.length()) + "\n");
        
        try {
            Connection conn = DriverManager.getConnection(url, user, password);
            System.out.println("Connexion réussie !\n");
            
            // Informations de connexion
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery(
                "SELECT USER() as user_connecte, " +
                "CURRENT_USER() as user_effectif, " +
                "DATABASE() as base_actuelle, " +
                "VERSION() as version_mariadb"
            );
            
            if (rs.next()) {
                System.out.println("Utilisateur connecté : " + rs.getString("user_connecte"));
                System.out.println("Utilisateur effectif : " + rs.getString("user_effectif"));
                System.out.println("Base de données : " + rs.getString("base_actuelle"));
                System.out.println("Version : " + rs.getString("version_mariadb") + "\n");
            }
            
            // Test de requête
            rs = stmt.executeQuery("SELECT COUNT(*) as nb FROM clients");
            if (rs.next()) {
                System.out.println("Nombre de clients : " + rs.getInt("nb"));
            }
            
            rs.close();
            stmt.close();
            conn.close();
            System.out.println("\nConnexion fermée.");
            
        } catch (SQLException e) {
            System.out.println("ERREUR DE CONNEXION : " + e.getMessage());
            System.out.println("Code d'erreur : " + e.getErrorCode());
            e.printStackTrace();
        }
    }
}
```

### Exercice 1.7 : Utilisation des rôles

```sql
-- Créer des rôles par fonction
CREATE ROLE role_lecteur;
CREATE ROLE role_gestionnaire;
CREATE ROLE role_admin_app;

-- Attribuer des privilèges aux rôles
GRANT SELECT ON test_security.* TO role_lecteur;

GRANT SELECT, INSERT, UPDATE, DELETE ON test_security.* TO role_gestionnaire;

GRANT ALL PRIVILEGES ON test_security.* TO role_admin_app;

-- Créer des utilisateurs et attribuer des rôles
CREATE USER 'user_lecture'@'%' IDENTIFIED BY 'LecturePass123!';
GRANT role_lecteur TO 'user_lecture'@'%';
SET DEFAULT ROLE role_lecteur FOR 'user_lecture'@'%';

CREATE USER 'user_gestion'@'%' IDENTIFIED BY 'GestionPass456!';
GRANT role_gestionnaire TO 'user_gestion'@'%';
SET DEFAULT ROLE role_gestionnaire FOR 'user_gestion'@'%';

CREATE USER 'user_admin'@'localhost' IDENTIFIED BY 'AdminPass789!';
GRANT role_admin_app TO 'user_admin'@'localhost';
SET DEFAULT ROLE role_admin_app FOR 'user_admin'@'localhost';

-- Tester les rôles
-- Connexion comme user_lecture
-- Test SELECT : OK
-- Test INSERT : ÉCHEC

-- Connexion comme user_gestion
-- Test SELECT, INSERT, UPDATE, DELETE : OK
-- Test DROP TABLE : ÉCHEC

-- Connexion comme user_admin
-- Toutes les opérations : OK
```

---
