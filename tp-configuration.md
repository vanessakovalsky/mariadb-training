# 🧩 TP — Ajustement de la configuration MariaDB

## ⏱ Durée : 1 heure  
## 🎯 Objectifs :
- Identifier les paramètres clés dans la configuration (`my.cnf`).  
- Ajuster la mémoire et le cache pour améliorer les performances.  
- Évaluer les effets sur la charge et la stabilité du serveur.

---

## 🔎 Contexte

Le serveur MariaDB d’une application interne est souvent lent à certaines heures.  
Les logs montrent beaucoup d’attente sur les verrous et des requêtes lentes.  
Tu dois **analyser la configuration**, **proposer des ajustements** et **observer leur impact**.

---

## 🧠 Travail demandé

### 1. Observation initiale
Affiche les valeurs actuelles de paramètres clés :
```sql
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW VARIABLES LIKE 'query_cache_size';
SHOW VARIABLES LIKE 'max_connections';
```
Compare-les à la mémoire disponible :
```bash
free -m
```

### 2. Analyse
- Quelle part de la RAM est réservée au buffer pool ?  
- Le cache de requêtes est-il activé ?  
- Le nombre de connexions max est-il adapté à la charge ?  

### 3. Expérimentation
- Modifie temporairement un ou deux paramètres (ex. mémoire ou cache) :
  ```sql
  SET GLOBAL innodb_buffer_pool_size = 256000000;
  SET GLOBAL query_cache_size = 32000000;
  ```
- Relance une requête lourde (par ex.) :
  ```sql
  SELECT COUNT(*) FROM sales WHERE amount > 100;
  ```

### 4. Observation
- Consulte les statistiques :
  ```sql
  SHOW STATUS LIKE 'Innodb_buffer_pool_%';
  SHOW STATUS LIKE 'Threads_connected';
  ```
- Note les différences avant / après.

### 5. Rendu attendu
Un mini-rapport contenant :
- Les paramètres modifiés.  
- Les effets observés sur les performances.  
- Une recommandation pour le fichier `my.cnf`.  

---

## 📄 Script SQL de départ : `tp2_configuration.sql` (à enregistrer dans un fichier et à importer dans mariadb)

```sql
DROP DATABASE IF EXISTS perf_test_db;
CREATE DATABASE perf_test_db;
USE perf_test_db;

CREATE TABLE sales (
    id INT AUTO_INCREMENT PRIMARY KEY,
    customer VARCHAR(100),
    amount DECIMAL(10,2),
    sale_date DATETIME,
    INDEX (sale_date)
) ENGINE=InnoDB;

CREATE TABLE sales_archive (
    id INT AUTO_INCREMENT PRIMARY KEY,
    customer VARCHAR(100),
    amount DECIMAL(10,2),
    sale_date DATETIME
) ENGINE=MyISAM;

DELIMITER //
CREATE PROCEDURE insert_sales()
BEGIN
  DECLARE i INT DEFAULT 1;
  WHILE i <= 10000 DO
    INSERT INTO sales (customer, amount, sale_date)
    VALUES (
      CONCAT('Client_', FLOOR(RAND() * 5000)),
      ROUND(RAND() * 200 + 5, 2),
      NOW() - INTERVAL FLOOR(RAND() * 100) DAY
    );
    SET i = i + 1;
  END WHILE;
END //
DELIMITER ;

CALL insert_sales();
DROP PROCEDURE insert_sales;

INSERT INTO sales_archive (customer, amount, sale_date)
SELECT customer, amount, sale_date
FROM sales
WHERE sale_date < NOW() - INTERVAL 30 DAY;
```
