# Atelier 1 : Requêtes avec plage temporelle (versioning)

```sql
-- Créer une base de test
CREATE DATABASE atelier_versioning;
USE atelier_versioning;

-- Table de prix avec historique
CREATE TABLE prix_produits (
    id INT PRIMARY KEY AUTO_INCREMENT,
    produit VARCHAR(100),
    prix DECIMAL(10,2),
    date_modification TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) WITH SYSTEM VERSIONING;

-- Insérer des prix initiaux
INSERT INTO prix_produits (produit, prix) VALUES
('Ordinateur', 899.99),
('Souris', 29.99),
('Clavier', 79.99);

-- Attendre et modifier
SELECT SLEEP(2);
UPDATE prix_produits SET prix = 849.99 WHERE produit = 'Ordinateur';

SELECT SLEEP(2);
UPDATE prix_produits SET prix = 24.99 WHERE produit = 'Souris';

SELECT SLEEP(2);
UPDATE prix_produits SET prix = 899.99 WHERE produit = 'Ordinateur';

-- Voir l'historique complet
SELECT 
    produit,
    prix,
    row_start AS valide_de,
    row_end AS valide_jusque
FROM prix_produits FOR SYSTEM_TIME ALL
ORDER BY produit, row_start;

-- Voir les prix à un moment spécifique
SET @moment = DATE_SUB(NOW(), INTERVAL 5 SECOND);
SELECT * FROM prix_produits FOR SYSTEM_TIME AS OF @moment;

-- Comparer les prix sur deux périodes
SELECT 
    p1.produit,
    p1.prix AS prix_avant,
    p2.prix AS prix_maintenant,
    p2.prix - p1.prix AS evolution
FROM prix_produits FOR SYSTEM_TIME AS OF @moment p1
JOIN prix_produits p2 ON p1.id = p2.id;
```

### Atelier 2 : Mise en œuvre des transactions InnoDB

```sql
-- Base de test pour transactions
CREATE TABLE comptes_bancaires (
    id INT PRIMARY KEY AUTO_INCREMENT,
    numero_compte VARCHAR(20) UNIQUE,
    titulaire VARCHAR(100),
    solde DECIMAL(12,2) CHECK (solde >= 0)
) ENGINE=InnoDB;

INSERT INTO comptes_bancaires (numero_compte, titulaire, solde) VALUES
('FR001', 'Alice Martin', 5000.00),
('FR002', 'Bob Dupont', 3000.00);

-- Test 1 : Transaction simple avec COMMIT
START TRANSACTION;
UPDATE comptes_bancaires SET solde = solde - 500 WHERE numero_compte = 'FR001';
SELECT * FROM comptes_bancaires WHERE numero_compte = 'FR001';
COMMIT;

-- Test 2 : Transaction avec ROLLBACK
START TRANSACTION;
UPDATE comptes_bancaires SET solde = solde - 1000 WHERE numero_compte = 'FR002';
SELECT * FROM comptes_bancaires WHERE numero_compte = 'FR002';
ROLLBACK;
SELECT * FROM comptes_bancaires WHERE numero_compte = 'FR002';  -- Inchangé

-- Test 3 : Virement (transaction complexe)
DELIMITER //
CREATE PROCEDURE virement_securise(
    IN compte_src VARCHAR(20),
    IN compte_dst VARCHAR(20),
    IN montant DECIMAL(12,2)
)
BEGIN
    DECLARE solde_actuel DECIMAL(12,2);
    
    START TRANSACTION;
    
    -- Vérifier le solde
    SELECT solde INTO solde_actuel
    FROM comptes_bancaires
    WHERE numero_compte = compte_src
    FOR UPDATE;
    
    IF solde_actuel < montant THEN
        ROLLBACK;
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Solde insuffisant';
    ELSE
        -- Débiter
        UPDATE comptes_bancaires 
        SET solde = solde - montant 
        WHERE numero_compte = compte_src;
        
        -- Créditer
        UPDATE comptes_bancaires 
        SET solde = solde + montant 
        WHERE numero_compte = compte_dst;
        
        COMMIT;
    END IF;
END //
DELIMITER ;

-- Tester le virement
CALL virement_securise('FR001', 'FR002', 1000.00);
SELECT * FROM comptes_bancaires;

-- Tester avec solde insuffisant
CALL virement_securise('FR002', 'FR001', 10000.00);
-- ERROR: Solde insuffisant
```


# Bonus - Columnstore

* Installer column Store a partir du docker compose ici : https://hub.docker.com/r/mariadb/columnstore#docker-compose-instructions-cluster

* Procédure pour créer les tables (une en innodb et une en column store) pour comparer

```sql
CREATE TABLE ventes_analytique_innodb (
id BIGINT,
date_vente DATE,
produit_id INT,
client_id INT,
quantite INT,
montant DECIMAL(12,2),
region VARCHAR(50)
);


DELIMITER //

CREATE PROCEDURE inserer_ventes_analytique()
BEGIN
  DECLARE i INT DEFAULT 1;
  DECLARE regions CHAR(5);
  DECLARE region_val VARCHAR(50);

  -- Boucle de 100 000 insertions
  WHILE i <= 100000 DO
    -- Sélection aléatoire d'une région
    SET regions = ELT(FLOOR(1 + (RAND() * 4)), 'Nord', 'Sud', 'Est', 'Ouest');

    INSERT INTO ventes_analytique_innodb (
      id, date_vente, produit_id, client_id, quantite, montant, region
    )
    VALUES (
      i,
      DATE_ADD('2020-01-01', INTERVAL FLOOR(RAND() * 2000) DAY), -- dates aléatoires sur plusieurs années
      FLOOR(1 + RAND() * 1000),  -- produit_id entre 1 et 1000
      FLOOR(1 + RAND() * 5000),  -- client_id entre 1 et 5000
      FLOOR(1 + RAND() * 10),    -- quantite entre 1 et 10
      ROUND(RAND() * 1000, 2),   -- montant entre 0.00 et 1000.00
      regions                    -- région choisie aléatoirement
    );

    SET i = i + 1;
  END WHILE;
END //

DELIMITER ;

// On crée la table identique avec columnstore

CREATE TABLE ventes_analytique_columnstore LIKE ventes_analytique_innodb;

ALTER TABLE ventes_analytique_columnstore ENGINE = ColumnStore;

// On duplique les données

INSERT INTO ventes_analytique_columnstore
SELECT * FROM ventes_analytique_innodb;


// Requête d'agregations à passer sur les deux tables pour comparer 

SELECT region,
COUNT(*) as nb_ventes,
SUM(quantite) as total_quantite,
SUM(montant) as total_montant,
AVG(montant) as montant_moyen
FROM ventes_analytique_innodb
GROUP BY region;

SELECT
YEAR(date_vente) as annee,
MONTH(date_vente) as mois,
SUM(montant) as chiffre_affaires
FROM ventes_analytique_innodb
GROUP BY annee, mois
ORDER BY annee, mois;

SELECT
v.region,
DATE_FORMAT(v.date_vente, '%Y-%m') as mois,
COUNT(DISTINCT v.client_id) as nb_clients,
SUM(v.montant) as ca
FROM ventes_analytique_innodb v
WHERE v.date_vente >= '2024-01-01'
GROUP BY v.region, mois
HAVING ca > 1000
ORDER BY ca DESC;
```
