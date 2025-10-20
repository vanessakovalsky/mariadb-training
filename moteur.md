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
