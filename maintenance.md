# 🛠️ ATELIER : OBSERVATION DE L'ACTIVITÉ 

### Objectif
Activer et analyser les logs pour observer l'activité du serveur.

### Étape 1 : Activer les logs

```sql
-- Activer le slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SET GLOBAL log_queries_not_using_indexes = 'ON';

-- Activer le general log (⚠️ temporaire)
SET GLOBAL general_log = 'ON';

-- Vérifier les chemins
SHOW VARIABLES LIKE '%log_file';
```

### Étape 2 : Créer des données de test

```sql
CREATE DATABASE atelier_logs;
USE atelier_logs;

CREATE TABLE clients (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nom VARCHAR(100),
    prenom VARCHAR(100),
    email VARCHAR(100),
    INDEX idx_nom (nom)
);

-- Insérer 10000 lignes
INSERT INTO clients (nom, prenom, email)
SELECT 
    CONCAT('Client', n),
    CONCAT('Prenom', n),
    CONCAT('client', n, '@email.com')
FROM (
    SELECT a.N + b.N * 10 + c.N * 100 + 1 AS n
    FROM 
        (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4) a,
        (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4) b,
        (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4) c
    LIMIT 10000
) numbers;
```

### Étape 3 : Générer des requêtes lentes

```sql
-- Requête sans index
SELECT * FROM clients WHERE email LIKE '%@email.com';

-- Requête avec sous-requête
SELECT * FROM clients
WHERE id IN (SELECT id FROM clients WHERE nom LIKE 'Client%');
```

### Étape 4 : Observer les processus

```sql
-- Voir les processus actifs
SHOW FULL PROCESSLIST;

-- Vue détaillée
SELECT Id, User, db, Command, Time, State, LEFT(Info, 100) AS Requete
FROM information_schema.PROCESSLIST
WHERE Command != 'Sleep'
ORDER BY Time DESC;
```

### Étape 5 : Analyser les logs

```bash
# Suivre le general log
tail -f /var/log/mysql/general.log

# Analyser le slow query log
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log
```

### Étape 6 : Désactiver les logs

```sql
-- IMPORTANT : Désactiver le general log
SET GLOBAL general_log = 'OFF';
```

## 🛠️ ATELIER : CONFIGURATION UTILISATEUR (15 min)

### Objectif
Configurer des paramètres par défaut pour les utilisateurs.

### Étape 1 : Configuration globale (my.cnf)

```ini
[mysqld]
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
wait_timeout = 600
interactive_timeout = 600
```

### Étape 2 : Configuration utilisateur (.my.cnf)

```bash
# Créer /home/user/.my.cnf
cat > ~/.my.cnf << 'EOF'
[client]
user = utilisateur
password = MotDePasse123
default-character-set = utf8mb4

[mysql]
prompt = '\u@\h [\d]> '
show-warnings
EOF

chmod 600 ~/.my.cnf
```

### Étape 3 : Vérifier la configuration

```sql
-- Se connecter
mysql

-- Vérifier les paramètres
SELECT @@character_set_client, 
       @@character_set_connection, 
       @@character_set_results;

SELECT @@wait_timeout;
```

## 🛠️ ATELIER : CHARGEMENT MULTI-SOURCES 

### Objectif
Charger des données depuis différents formats.

### Étape 1 : Créer une table

```sql
CREATE DATABASE atelier_import;
USE atelier_import;

CREATE TABLE clients_import (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nom VARCHAR(100),
    prenom VARCHAR(100),
    email VARCHAR(100)
) CHARACTER SET utf8mb4;
```

### Étape 2 : Créer un fichier CSV

```bash
# Créer /var/lib/mysql-files/clients.csv
cat > /var/lib/mysql-files/clients.csv << 'EOF'
nom,prenom,email
Dupont,Jean,jean.dupont@email.fr
Martin,Sophie,sophie.martin@email.fr
Bernard,Paul,paul.bernard@email.fr
EOF
```

### Étape 3 : Charger avec LOAD DATA

```sql
LOAD DATA INFILE '/var/lib/mysql-files/clients.csv'
INTO TABLE clients_import
CHARACTER SET utf8mb4
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 LINES
(nom, prenom, email);

SELECT * FROM clients_import;
```

### Étape 4 : Export

```sql
SELECT * INTO OUTFILE '/var/lib/mysql-files/export.csv'
FIELDS TERMINATED BY ','
FROM clients_import;
```

## 🛠️ ATELIER : CORRECTION D'ENCODAGE

### Objectif
Corriger une table mal encodée.

### Étape 1 : Simuler le problème

```sql
USE atelier_import;

-- Table en latin1
CREATE TABLE clients_latin1 (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nom VARCHAR(100),
    commentaire TEXT
) CHARACTER SET latin1;

INSERT INTO clients_latin1 (nom, commentaire) VALUES
('François', 'Client très satisfait'),
('Stéphanie', 'Demande des devis personnalisés');
```

### Étape 2 : Diagnostic

```sql
-- Vérifier l'encodage
SHOW CREATE TABLE clients_latin1;

SELECT TABLE_NAME, TABLE_COLLATION
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'atelier_import';

-- Voir les colonnes
SELECT COLUMN_NAME, CHARACTER_SET_NAME, COLLATION_NAME
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'atelier_import'
  AND TABLE_NAME = 'clients_latin1';
```

### Étape 3 : Conversion simple

```sql
-- Si les données sont correctement encodées
ALTER TABLE clients_latin1 
CONVERT TO CHARACTER SET utf8mb4 
COLLATE utf8mb4_unicode_ci;

-- Vérifier
SHOW CREATE TABLE clients_latin1;
SELECT * FROM clients_latin1;
```

### Étape 4 : Double conversion (pour données mal stockées)

```sql
-- Créer une table avec problème simulé
CREATE TABLE clients_probleme LIKE clients_latin1;

SET NAMES latin1;
INSERT INTO clients_probleme SELECT * FROM clients_latin1;
SET NAMES utf8mb4;

-- On voit des caractères bizarres
SELECT * FROM clients_probleme;

-- SOLUTION : Double conversion
-- Étape 1 : En binaire (préserve les bytes)
ALTER TABLE clients_probleme 
MODIFY nom VARBINARY(100),
MODIFY commentaire VARBINARY(65535);

-- Étape 2 : En UTF8MB4
ALTER TABLE clients_probleme 
MODIFY nom VARCHAR(100) CHARACTER SET utf8mb4,
MODIFY commentaire TEXT CHARACTER SET utf8mb4;

ALTER TABLE clients_probleme 
CHARACTER SET utf8mb4 
COLLATE utf8mb4_unicode_ci;

-- Vérifier : les caractères sont corrects
SELECT * FROM clients_probleme;
```

### Étape 5 : Script de conversion automatique

```sql
DELIMITER //

CREATE PROCEDURE convert_all_to_utf8mb4(IN db_name VARCHAR(64))
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE tbl_name VARCHAR(64);
    DECLARE cur CURSOR FOR 
        SELECT TABLE_NAME 
        FROM information_schema.TABLES 
        WHERE TABLE_SCHEMA = db_name 
          AND TABLE_TYPE = 'BASE TABLE'
          AND TABLE_COLLATION NOT LIKE 'utf8mb4%';
    
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    
    OPEN cur;
    
    read_loop: LOOP
        FETCH cur INTO tbl_name;
        IF done THEN
            LEAVE read_loop;
        END IF;
        
        SET @sql = CONCAT('ALTER TABLE ', db_name, '.', tbl_name, 
                          ' CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci');
        
        SELECT CONCAT('Converting: ', tbl_name) AS status;
        PREPARE stmt FROM @sql;
        EXECUTE stmt;
        DEALLOCATE PREPARE stmt;
    END LOOP;
    
    CLOSE cur;
    
    -- Convertir la base
    SET @sql = CONCAT('ALTER DATABASE ', db_name, 
                      ' CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
END//

DELIMITER ;

-- Utiliser
CALL convert_all_to_utf8mb4('atelier_import');

-- Supprimer
DROP PROCEDURE convert_all_to_utf8mb4;
```

### Étape 6 : Vérification finale

```sql
-- Vérifier qu'il n'y a plus de tables non UTF8MB4
SELECT 
    TABLE_NAME,
    TABLE_COLLATION
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'atelier_import'
  AND TABLE_COLLATION NOT LIKE 'utf8mb4%';

-- Vérifier les colonnes
SELECT 
    TABLE_NAME,
    COLUMN_NAME,
    CHARACTER_SET_NAME
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'atelier_import'
  AND CHARACTER_SET_NAME IS NOT NULL
  AND CHARACTER_SET_NAME != 'utf8mb4';
```

---
