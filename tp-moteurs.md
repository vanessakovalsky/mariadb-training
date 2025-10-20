# 🧩 TP — Étude des moteurs de stockage MariaDB

## ⏱ Durée : 1 heure  
## 🎯 Objectifs :
- Comprendre les différences entre les moteurs de stockage (`InnoDB`, `MyISAM`, `MEMORY`).  
- Expérimenter leurs comportements (transactions, performances, durabilité).  
- Être capable de recommander un moteur selon un besoin métier.

---

## 🔎 Contexte

Tu travailles pour une application web de gestion de commandes.  
Certaines tables perdent leurs données après redémarrage, d’autres sont très lentes lors de grosses opérations.  
On te demande d’**analyser et de choisir les moteurs de stockage adaptés** à chaque type de table.

---

## 🧠 Travail demandé

### 1. Analyse du moteur de stockage actuel
- Affiche le moteur de chaque table de la base :
  ```sql
  SHOW TABLE STATUS;
  ```
- Ou bien :
  ```sql
  SELECT table_name, engine FROM information_schema.tables WHERE table_schema = 'orders_db';
  ```

### 2. Expérimentation transactionnelle
- Crée deux tables identiques, l’une en `InnoDB`, l’autre en `MyISAM`.  
- Insère quelques lignes dans les deux tables.  
- Supprime certaines lignes puis fais un `ROLLBACK`.  
  → Que constates-tu ? Pourquoi ?

### 3. Test de performance
- Exécute un `SELECT COUNT(*)` sur chaque table et compare les temps.  
- Observe la taille des fichiers dans `/var/lib/mysql/orders_db`.  

### 4. Réflexion
- Quels avantages et inconvénients as-tu observés pour chaque moteur ?  
- Dans quels cas utiliserais-tu :
  - `InnoDB` ?  
  - `MyISAM` ?  
  - `MEMORY` ?  

### 5. Rendu attendu
Un court rapport (5 à 10 lignes) avec :
- Les moteurs choisis pour chaque table.  
- Les raisons techniques et métiers.  
- Les conséquences sur la durabilité et les performances.  

---

## 📄 Script SQL de départ : `tp1_moteurs_stockage.sql` (à enregistrer et à importer dans mariadb : mysql -u root -p < tp1_moteurs_stockage.sql )

```sql
DROP DATABASE IF EXISTS orders_db;
CREATE DATABASE orders_db;
USE orders_db;

CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    customer_name VARCHAR(100),
    product_name VARCHAR(100),
    quantity INT,
    total_price DECIMAL(10,2),
    order_date DATETIME DEFAULT NOW()
) ENGINE=InnoDB;

CREATE TABLE order_stats (
    stat_id INT AUTO_INCREMENT PRIMARY KEY,
    product_name VARCHAR(100),
    total_sold INT,
    last_update DATETIME
) ENGINE=MyISAM;

CREATE TABLE session_cache (
    session_id CHAR(36) PRIMARY KEY,
    user_name VARCHAR(100),
    last_access DATETIME
) ENGINE=MEMORY;

INSERT INTO orders (customer_name, product_name, quantity, total_price)
SELECT 
  CONCAT('Client_', FLOOR(RAND() * 1000)),
  CASE FLOOR(RAND() * 4)
    WHEN 0 THEN 'Pizza Margherita'
    WHEN 1 THEN 'Burger Classic'
    WHEN 2 THEN 'Burger Bacon'
    ELSE 'Pizza 4 Fromages'
  END,
  FLOOR(RAND() * 5) + 1,
  ROUND(RAND() * 40 + 10, 2)
FROM information_schema.tables
LIMIT 200;

INSERT INTO order_stats (product_name, total_sold, last_update)
VALUES
('Pizza Margherita', 350, NOW()),
('Burger Classic', 275, NOW()),
('Burger Bacon', 420, NOW()),
('Pizza 4 Fromages', 150, NOW());

INSERT INTO session_cache (session_id, user_name, last_access)
VALUES
(UUID(), 'admin', NOW()),
(UUID(), 'user1', NOW() - INTERVAL 10 MINUTE),
(UUID(), 'user2', NOW() - INTERVAL 1 HOUR);
```
