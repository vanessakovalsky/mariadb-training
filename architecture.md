# Atelier : Architecture des tables et partitionnement

**Objectif :** Comprendre le cycle de vie d'une requête, créer des tables avec différentes structures et implémenter le partitionnement.

## Création des tables nécessaires

**Créer une base de données de test :**
```sql
CREATE DATABASE IF NOT EXISTS formation_db 
DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

USE formation_db;
```

**Créer des tables avec différents moteurs :**
```sql
-- Table InnoDB (transactionnelle)
CREATE TABLE departements (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nom VARCHAR(100) NOT NULL UNIQUE,
    localisation VARCHAR(100),
    budget DECIMAL(12,2),
    date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- Table InnoDB avec clés étrangères
CREATE TABLE employes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    matricule VARCHAR(20) NOT NULL UNIQUE,
    nom VARCHAR(100) NOT NULL,
    prenom VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE,
    telephone VARCHAR(20),
    date_naissance DATE,
    date_embauche DATE NOT NULL,
    salaire DECIMAL(10,2),
    departement_id INT,
    poste VARCHAR(100),
    actif BOOLEAN DEFAULT TRUE,
    INDEX idx_nom (nom),
    INDEX idx_departement (departement_id),
    INDEX idx_actif_dept (actif, departement_id),
    FOREIGN KEY (departement_id) REFERENCES departements(id)
        ON DELETE SET NULL
        ON UPDATE CASCADE
) ENGINE=InnoDB;

-- Table MyISAM pour archives (lecture seule)
CREATE TABLE logs_archives (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    date_log DATETIME NOT NULL,
    niveau ENUM('INFO', 'WARNING', 'ERROR', 'CRITICAL'),
    module VARCHAR(50),
    message TEXT,
    INDEX idx_date (date_log),
    INDEX idx_niveau (niveau)
) ENGINE=MyISAM;

-- Table MEMORY pour données temporaires
CREATE TABLE sessions_actives (
    session_id VARCHAR(64) PRIMARY KEY,
    user_id INT NOT NULL,
    ip_address VARCHAR(45),
    dernier_acces TIMESTAMP,
    INDEX idx_user (user_id)
) ENGINE=MEMORY;
```

**Insérer des données de test :**
```sql
-- Départements
INSERT INTO departements (nom, localisation, budget) VALUES
('Informatique', 'Paris', 500000),
('Ressources Humaines', 'Lyon', 200000),
('Commercial', 'Marseille', 350000),
('Finance', 'Paris', 400000),
('Logistique', 'Lille', 250000);

-- Employés
INSERT INTO employes (matricule, nom, prenom, email, date_naissance, date_embauche, salaire, departement_id, poste) VALUES
('EMP001', 'Dupont', 'Jean', 'jean.dupont@societe.com', '1985-03-15', '2015-01-10', 45000, 1, 'Développeur'),
('EMP002', 'Martin', 'Sophie', 'sophie.martin@societe.com', '1990-07-22', '2018-03-15', 42000, 1, 'Développeur'),
('EMP003', 'Bernard', 'Pierre', 'pierre.bernard@societe.com', '1982-11-08', '2012-06-01', 55000, 1, 'Chef de projet'),
('EMP004', 'Dubois', 'Marie', 'marie.dubois@societe.com', '1988-05-30', '2016-09-12', 38000, 2, 'Chargée RH'),
('EMP005', 'Leroy', 'Thomas', 'thomas.leroy@societe.com', '1992-01-18', '2019-02-20', 48000, 3, 'Commercial'),
('EMP006', 'Moreau', 'Emma', 'emma.moreau@societe.com', '1987-09-25', '2014-11-03', 52000, 4, 'Contrôleur de gestion'),
('EMP007', 'Simon', 'Lucas', 'lucas.simon@societe.com', '1991-12-12', '2017-04-18', 41000, 5, 'Responsable logistique');
```

### Exercice 3.1 : Mise à plat du cycle d'exécution d'une requête

**Schéma du cycle d'une requête SELECT :**

1. **Connexion** : Le client établit une connexion TCP/IP ou via socket
2. **Authentification** : Vérification des identifiants (user, host, password)
3. **Réception de la requête** : Le daemon mysqld reçoit la requête SQL
4. **Parsing** : Analyse syntaxique de la requête
5. **Vérification des privilèges** : Contrôle des droits d'accès
6. **Optimisation** : Le query optimizer génère un plan d'exécution
7. **Accès au cache** : Recherche dans le buffer pool (InnoDB)
8. **Lecture des données** :
   - Si en cache : lecture en mémoire (rapide)
   - Si non en cache : lecture depuis le disque (plus lent)
9. **Exécution** : Application des filtres, jointures, tris
10. **Construction du résultat** : Formatage des données
11. **Retour au client** : Envoi via le réseau
12. **Affichage** : Le client reçoit et affiche les résultats

**Exercice pratique :**
```sql
-- Activer le profiling
SET profiling = 1;

-- Exécuter une requête
SELECT * FROM employes WHERE departement_id = 5;

-- Analyser le profil d'exécution
SHOW PROFILES;
SHOW PROFILE FOR QUERY 1;

-- Voir le plan d'exécution
EXPLAIN SELECT * FROM employes WHERE departement_id = 5;
```

### Exercice 3.2 : Création  et de vues

**Créer des vues :**
```sql
-- Vue simple
CREATE VIEW vue_employes_actifs AS
SELECT 
    e.id,
    e.matricule,
    e.nom,
    e.prenom,
    e.email,
    e.poste,
    d.nom AS departement
FROM employes e
LEFT JOIN departements d ON e.departement_id = d.id
WHERE e.actif = TRUE;

-- Vue avec agrégation
CREATE VIEW vue_stats_departements AS
SELECT 
    d.id,
    d.nom AS departement,
    d.localisation,
    COUNT(e.id) AS nb_employes,
    ROUND(AVG(e.salaire), 2) AS salaire_moyen,
    MIN(e.salaire) AS salaire_min,
    MAX(e.salaire) AS salaire_max,
    SUM(e.salaire) AS masse_salariale
FROM departements d
LEFT JOIN employes e ON d.id = e.departement_id AND e.actif = TRUE
GROUP BY d.id, d.nom, d.localisation;

-- Vue complexe avec calculs
CREATE VIEW vue_anciennete_employes AS
SELECT 
    e.id,
    e.nom,
    e.prenom,
    d.nom AS departement,
    e.date_embauche,
    TIMESTAMPDIFF(YEAR, e.date_embauche, CURDATE()) AS anciennete_annees,
    TIMESTAMPDIFF(MONTH, e.date_embauche, CURDATE()) AS anciennete_mois,
    e.salaire,
    CASE 
        WHEN TIMESTAMPDIFF(YEAR, e.date_embauche, CURDATE()) < 2 THEN 'Junior'
        WHEN TIMESTAMPDIFF(YEAR, e.date_embauche, CURDATE()) BETWEEN 2 AND 5 THEN 'Confirmé'
        WHEN TIMESTAMPDIFF(YEAR, e.date_embauche, CURDATE()) BETWEEN 5 AND 10 THEN 'Senior'
        ELSE 'Expert'
    END AS niveau_experience
FROM employes e
LEFT JOIN departements d ON e.departement_id = d.id
WHERE e.actif = TRUE;

-- Tester les vues
SELECT * FROM vue_employes_actifs;
SELECT * FROM vue_stats_departements;
SELECT * FROM vue_anciennete_employes ORDER BY anciennete_annees DESC;
```

### Exercice 3.3 : Création de tables partitionnées

**Table partitionnée par RANGE (année) :**
```sql
CREATE TABLE commandes (
    id BIGINT AUTO_INCREMENT,
    numero_commande VARCHAR(20) NOT NULL,
    date_commande DATE NOT NULL,
    client_id INT NOT NULL,
    montant_total DECIMAL(12,2),
    statut ENUM('En attente', 'Validée', 'Expédiée', 'Livrée', 'Annulée'),
    PRIMARY KEY (id, date_commande),
    INDEX idx_client (client_id),
    INDEX idx_statut (statut)
) ENGINE=InnoDB
PARTITION BY RANGE (YEAR(date_commande)) (
    PARTITION p_2022 VALUES LESS THAN (2023),
    PARTITION p_2023 VALUES LESS THAN (2024),
    PARTITION p_2024 VALUES LESS THAN (2025),
    PARTITION p_2025 VALUES LESS THAN (2026),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

**Table partitionnée par HASH :**
```sql
CREATE TABLE clients (
    id INT AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    prenom VARCHAR(100),
    email VARCHAR(255) NOT NULL UNIQUE,
    telephone VARCHAR(20),
    date_inscription DATE,
    PRIMARY KEY (id)
) ENGINE=InnoDB
PARTITION BY HASH(id)
PARTITIONS 8;
```

**Table partitionnée par LIST :**
```sql
CREATE TABLE produits (
    id INT AUTO_INCREMENT,
    reference VARCHAR(50) NOT NULL UNIQUE,
    nom VARCHAR(200),
    categorie VARCHAR(50) NOT NULL,
    prix DECIMAL(10,2),
    stock INT DEFAULT 0,
    PRIMARY KEY (id, categorie),
    INDEX idx_stock (stock)
) ENGINE=InnoDB
PARTITION BY LIST COLUMNS(categorie) (
    PARTITION p_electronique VALUES IN ('Ordinateurs', 'Téléphones', 'Tablettes', 'Accessoires électroniques'),
    PARTITION p_electromenager VALUES IN ('Gros électroménager', 'Petit électroménager'),
    PARTITION p_multimedia VALUES IN ('TV', 'Audio', 'Photo', 'Vidéo'),
    PARTITION p_informatique VALUES IN ('Composants', 'Périphériques', 'Réseaux', 'Stockage'),
    PARTITION p_autres VALUES IN ('Divers', 'Services')
);
```

**Table partitionnée par mois (TO_DAYS) :**
```sql
CREATE TABLE logs_application (
    id BIGINT AUTO_INCREMENT,
    timestamp_log DATETIME NOT NULL,
    niveau VARCHAR(20),
    application VARCHAR(50),
    utilisateur_id INT,
    message TEXT,
    donnees_supplementaires JSON,
    PRIMARY KEY (id, timestamp_log),
    INDEX idx_niveau (niveau),
    INDEX idx_utilisateur (utilisateur_id)
) ENGINE=InnoDB
PARTITION BY RANGE (TO_DAYS(timestamp_log)) (
    PARTITION p_202409 VALUES LESS THAN (TO_DAYS('2024-10-01')),
    PARTITION p_202410 VALUES LESS THAN (TO_DAYS('2024-11-01')),
    PARTITION p_202411 VALUES LESS THAN (TO_DAYS('2024-12-01')),
    PARTITION p_202412 VALUES LESS THAN (TO_DAYS('2025-01-01')),
    PARTITION p_202501 VALUES LESS THAN (TO_DAYS('2025-02-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

**Insérer des données de test :**
```sql
-- Données pour la table commandes
INSERT INTO commandes (numero_commande, date_commande, client_id, montant_total, statut) VALUES
('CMD-2022-001', '2022-03-15', 101, 1250.50, 'Livrée'),
('CMD-2022-002', '2022-08-22', 102, 890.00, 'Livrée'),
('CMD-2023-001', '2023-01-10', 103, 2340.75, 'Livrée'),
('CMD-2023-002', '2023-06-18', 104, 1560.25, 'Livrée'),
('CMD-2024-001', '2024-02-05', 105, 3200.00, 'Expédiée'),
('CMD-2024-002', '2024-05-12', 106, 1870.50, 'Validée'),
('CMD-2024-003', '2024-09-28', 107, 4125.90, 'En attente'),
('CMD-2025-001', '2025-01-03', 108, 2450.00, 'Validée');

-- Données pour la table produits
INSERT INTO produits (reference, nom, categorie, prix, stock) VALUES
('PC-001', 'Ordinateur portable 15 pouces', 'Ordinateurs', 899.99, 25),
('TEL-001', 'Smartphone 5G', 'Téléphones', 649.99, 50),
('TAB-001', 'Tablette 10 pouces', 'Tablettes', 329.99, 30),
('TV-001', 'TV LED 55 pouces', 'TV', 599.99, 15),
('COMP-001', 'Processeur 8 cœurs', 'Composants', 299.99, 40),
('ELEC-001', 'Réfrigérateur', 'Gros électroménager', 799.99, 10);

-- Données pour la table logs_application
INSERT INTO logs_application (timestamp_log, niveau, application, utilisateur_id, message) VALUES
('2024-10-01 10:30:00', 'INFO', 'WebApp', 1, 'Connexion utilisateur'),
('2024-10-15 14:22:33', 'WARNING', 'API', 2, 'Tentative de connexion échouée'),
('2024-11-05 09:15:42', 'ERROR', 'WebApp', 3, 'Erreur de base de données'),
('2024-12-20 18:45:12', 'INFO', 'Batch', NULL, 'Traitement nocturne terminé'),
('2025-01-10 11:20:30', 'INFO', 'WebApp', 4, 'Commande créée');
```

### Exercice 3.4 : Utilisation détaillée de SHOW TABLE STATUS

**Afficher le statut complet d'une table :**
```sql
-- Format étendu pour une table
SHOW TABLE STATUS FROM formation_db LIKE 'employes'\G

-- Format tabulaire
SHOW TABLE STATUS FROM formation_db WHERE Name = 'employes';
```

**Analyser les informations retournées :**
```sql
-- Créer une requête d'analyse détaillée
SELECT 
    'Nom de la table' AS Information,
    Name AS Valeur
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'formation_db' AND TABLE_NAME = 'employes'

UNION ALL

SELECT 
    'Moteur de stockage',
    Engine
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'formation_db' AND TABLE_NAME = 'employes'

UNION ALL

SELECT 
    'Version',
    CAST(Version AS CHAR)
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'formation_db' AND TABLE_NAME = 'employes'

UNION ALL

SELECT 
    'Format des lignes',
    Row_format
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'formation_db' AND TABLE_NAME = 'employes'

UNION ALL

SELECT 
    'Nombre de lignes',
    CAST(TABLE_ROWS AS CHAR)
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'formation_db' AND TABLE_NAME = 'employes'

UNION ALL

SELECT 
    'Longueur moyenne ligne',
    CAST(AVG_ROW_LENGTH AS CHAR)
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'formation_db' AND TABLE_NAME = 'employes'

UNION ALL

SELECT 
    'Taille données (Mo)',
    CAST(ROUND(DATA_LENGTH / 1024 / 1024, 2) AS CHAR)
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'formation_db' AND TABLE_NAME = 'employes'

UNION ALL

SELECT 
    'Taille index (Mo)',
    CAST(ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS CHAR)
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'formation_db' AND TABLE_NAME = 'employes'

UNION ALL

SELECT 
    'Espace libre (Mo)',
    CAST(ROUND(DATA_FREE / 1024 / 1024, 2) AS CHAR)
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'formation_db' AND TABLE_NAME = 'employes'

UNION ALL

SELECT 
    'Auto increment',
    CAST(AUTO_INCREMENT AS CHAR)
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'formation_db' AND TABLE_NAME = 'employes';
```

**Analyser toutes les tables de la base :**
```sql
-- Rapport complet sur toutes les tables
SELECT 
    TABLE_NAME AS 'Table',
    ENGINE AS 'Moteur',
    ROW_FORMAT AS 'Format',
    TABLE_ROWS AS 'Nb lignes',
    ROUND(AVG_ROW_LENGTH, 2) AS 'Long. moy.',
    ROUND(DATA_LENGTH / 1024 / 1024, 2) AS 'Données (Mo)',
    ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS 'Index (Mo)',
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS 'Total (Mo)',
    ROUND(DATA_FREE / 1024 / 1024, 2) AS 'Libre (Mo)',
    ROUND((DATA_FREE / (DATA_LENGTH + INDEX_LENGTH)) * 100, 2) AS 'Frag. %',
    CREATE_TIME AS 'Créée le',
    UPDATE_TIME AS 'MAJ le',
    TABLE_COLLATION AS 'Collation'
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'formation_db'
  AND TABLE_TYPE = 'BASE TABLE'
ORDER BY (DATA_LENGTH + INDEX_LENGTH) DESC;
```

**Analyser les tables partitionnées :**
```sql
-- Détails des partitions
SELECT 
    TABLE_NAME AS 'Table',
    PARTITION_NAME AS 'Partition',
    PARTITION_METHOD AS 'Méthode',
    PARTITION_EXPRESSION AS 'Expression',
    PARTITION_DESCRIPTION AS 'Description',
    TABLE_ROWS AS 'Lignes',
    ROUND(DATA_LENGTH / 1024 / 1024, 2) AS 'Données (Mo)',
    ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS 'Index (Mo)',
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS 'Total (Mo)',
    CREATE_TIME AS 'Créée le',
    UPDATE_TIME AS 'MAJ le'
FROM information_schema.PARTITIONS
WHERE TABLE_SCHEMA = 'formation_db'
  AND TABLE_NAME = 'commandes'
ORDER BY PARTITION_ORDINAL_POSITION;
```

**Identifier les problèmes potentiels :**
```sql
-- Tables fragmentées (> 10% de fragmentation)
SELECT 
    TABLE_NAME,
    ENGINE,
    ROUND((DATA_FREE / (DATA_LENGTH + INDEX_LENGTH)) * 100, 2) AS fragmentation_pct,
    ROUND(DATA_FREE / 1024 / 1024, 2) AS espace_libre_mb,
    CONCAT('OPTIMIZE TABLE ', TABLE_NAME, ';') AS solution
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'formation_db'
  AND DATA_FREE > 0
  AND (DATA_LENGTH + INDEX_LENGTH) > 0
  AND (DATA_FREE / (DATA_LENGTH + INDEX_LENGTH)) > 0.10
ORDER BY fragmentation_pct DESC;

-- Tables sans clé primaire
SELECT 
    TABLE_NAME,
    ENGINE,
    TABLE_ROWS,
    'Ajouter une clé primaire' AS recommandation
FROM information_schema.TABLES t
WHERE TABLE_SCHEMA = 'formation_db'
  AND NOT EXISTS (
      SELECT 1 
      FROM information_schema.TABLE_CONSTRAINTS c
      WHERE c.TABLE_SCHEMA = t.TABLE_SCHEMA
        AND c.TABLE_NAME = t.TABLE_NAME
        AND c.CONSTRAINT_TYPE = 'PRIMARY KEY'
  );

-- Tables volumineuses sans partitionnement
SELECT 
    TABLE_NAME,
    ENGINE,
    TABLE_ROWS,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS taille_mb,
    'Envisager le partitionnement' AS recommandation
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'formation_db'
  AND TABLE_ROWS > 100000
  AND (DATA_LENGTH + INDEX_LENGTH) > 100 * 1024 * 1024
  AND PARTITION_NAME IS NULL
ORDER BY TABLE_ROWS DESC;
```

### Exercice 3.5 : Comparaison des moteurs de stockage

**Créer des tables identiques avec différents moteurs :**
```sql
-- Table de test InnoDB
CREATE TABLE test_innodb (
    id INT AUTO_INCREMENT PRIMARY KEY,
    donnee VARCHAR(100),
    valeur INT,
    date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- Table de test MyISAM
CREATE TABLE test_myisam (
    id INT AUTO_INCREMENT PRIMARY KEY,
    donnee VARCHAR(100),
    valeur INT,
    date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=MyISAM;

-- Table de test MEMORY
CREATE TABLE test_memory (
    id INT AUTO_INCREMENT PRIMARY KEY,
    donnee VARCHAR(100),
    valeur INT,
    date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=MEMORY;
```

**Insérer des données identiques :**
```sql
-- Procédure pour insérer des données de test
DELIMITER //

CREATE PROCEDURE inserer_donnees_test(IN nb_lignes INT)
BEGIN
    DECLARE i INT DEFAULT 1;
    
    WHILE i <= nb_lignes DO
        INSERT INTO test_innodb (donnee, valeur) 
        VALUES (CONCAT('Donnée ', i), FLOOR(RAND() * 1000));
        
        INSERT INTO test_myisam (donnee, valeur) 
        VALUES (CONCAT('Donnée ', i), FLOOR(RAND() * 1000));
        
        INSERT INTO test_memory (donnee, valeur) 
        VALUES (CONCAT('Donnée ', i), FLOOR(RAND() * 1000));
        
        SET i = i + 1;
    END WHILE;
END //

DELIMITER ;

-- Insérer 10000 lignes
CALL inserer_donnees_test(10000);
```

**Comparer les performances :**
```sql
-- Activer le profiling
SET profiling = 1;

-- Test de lecture InnoDB
SELECT COUNT(*), AVG(valeur) FROM test_innodb WHERE valeur > 500;

-- Test de lecture MyISAM
SELECT COUNT(*), AVG(valeur) FROM test_myisam WHERE valeur > 500;

-- Test de lecture MEMORY
SELECT COUNT(*), AVG(valeur) FROM test_memory WHERE valeur > 500;

-- Voir les résultats
SHOW PROFILES;

-- Comparer les tailles
SELECT 
    TABLE_NAME,
    ENGINE,
    TABLE_ROWS,
    ROUND(DATA_LENGTH / 1024, 2) AS data_kb,
    ROUND(INDEX_LENGTH / 1024, 2) AS index_kb,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024, 2) AS total_kb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'formation_db'
  AND TABLE_NAME LIKE 'test_%'
ORDER BY TABLE_NAME;
```

**Tester le comportement transactionnel :**
```sql
-- Test InnoDB (supporte les transactions)
START TRANSACTION;
UPDATE test_innodb SET valeur = valeur + 100 WHERE id <= 100;
SELECT COUNT(*) FROM test_innodb WHERE valeur > 600;
ROLLBACK;
-- Les modifications sont annulées

-- Test MyISAM (ne supporte pas les transactions)
START TRANSACTION;
UPDATE test_myisam SET valeur = valeur + 100 WHERE id <= 100;
SELECT COUNT(*) FROM test_myisam WHERE valeur > 600;
ROLLBACK;
-- Les modifications sont conservées malgré le ROLLBACK
```

### Exercice 3.6 : Optimisation et maintenance

**Analyser les tables :**
```sql
-- Analyser une table
ANALYZE TABLE employes;

-- Analyser toutes les tables de la base
SELECT CONCAT('ANALYZE TABLE ', TABLE_NAME, ';') AS commande
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'formation_db';
```

**Optimiser les tables :**
```sql
-- Optimiser une table (défragmente et reconstruit les index)
OPTIMIZE TABLE employes;

-- Optimiser une partition spécifique
ALTER TABLE commandes OPTIMIZE PARTITION p_2024;
```

**Réparer une table (MyISAM principalement) :**
```sql
-- Vérifier l'intégrité
CHECK TABLE test_myisam;

-- Réparer si nécessaire
REPAIR TABLE test_myisam;
```

**Reconstruire les index :**
```sql
-- Supprimer et recréer un index
ALTER TABLE employes DROP INDEX idx_nom;
ALTER TABLE employes ADD INDEX idx_nom (nom);

-- Ou simplement
ALTER TABLE employes DROP INDEX idx_nom, ADD INDEX idx_nom (nom);
```
