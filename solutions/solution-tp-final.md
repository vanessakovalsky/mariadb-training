
# ✅ Correction - TP Final MariaDB : Mise en production

## 1. Moteurs de stockage
| Table             | Moteur  | Justification |
|-------------------|---------|----------------|
| `orders`          | InnoDB  | Transactions, contraintes, durabilité |
| `order_items`     | InnoDB  | Cohérence référentielle avec `orders` |
| `product_cache`   | MEMORY  | Accès rapide, données temporaires |
| `reporting_orders`| MyISAM  | Lecture intensive, peu de modifications |

Exemples SQL :
```sql
CREATE DATABASE ecom_db;
USE ecom_db;

CREATE TABLE orders (
  id INT AUTO_INCREMENT PRIMARY KEY,
  order_date DATETIME,
  amount DECIMAL(10,2)
) ENGINE=InnoDB;

CREATE TABLE order_items (
  id INT AUTO_INCREMENT PRIMARY KEY,
  order_id INT,
  product_name VARCHAR(100),
  quantity INT,
  price DECIMAL(10,2),
  FOREIGN KEY (order_id) REFERENCES orders(id)
) ENGINE=InnoDB;

CREATE TABLE product_cache (
  product_id INT PRIMARY KEY,
  product_name VARCHAR(100),
  stock INT
) ENGINE=MEMORY;

CREATE TABLE reporting_orders AS SELECT * FROM orders;
ALTER TABLE reporting_orders ENGINE=MyISAM;
```

---

## 2. Utilisateurs et droits
```sql
CREATE USER 'app_user'@'%' IDENTIFIED BY 'App2025!';
GRANT SELECT, INSERT, UPDATE ON ecom_db.orders TO 'app_user'@'%';
GRANT SELECT, INSERT, UPDATE ON ecom_db.order_items TO 'app_user'@'%';

CREATE USER 'report_user'@'%' IDENTIFIED BY 'Report2025!';
GRANT SELECT ON ecom_db.reporting_orders TO 'report_user'@'%';

CREATE USER 'audit_user'@'localhost' IDENTIFIED BY 'Audit2025!';
GRANT SELECT ON *.* TO 'audit_user'@'localhost';

FLUSH PRIVILEGES;
```

---

## 3. Configuration et optimisation
### Variables observées :
```sql
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW VARIABLES LIKE 'query_cache_size';
SHOW VARIABLES LIKE 'max_connections';
```

### Ajustements proposés :
- `innodb_buffer_pool_size` : passer à 1 Go sur serveur 4 Go RAM.
- `query_cache_size` : désactiver si usage majoritaire d’InnoDB.

Application temporaire :
```sql
SET GLOBAL innodb_buffer_pool_size = 1073741824;
SET GLOBAL query_cache_size = 0;
```

Test de requête lourde :
```sql
SELECT COUNT(*) FROM orders WHERE order_date > NOW() - INTERVAL 30 DAY;
```

Vérification :
```sql
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read_requests';
```

---

## 4. Sauvegarde et restauration
```bash
mysqldump -u root -p ecom_db > /backup/ecom_db_dump.sql
mysql -u root -p ecom_db < /backup/ecom_db_dump.sql
```

Stratégie recommandée :
- Dump quotidien des bases critiques.
- Rotation sur 7 jours.
- Utilisation des journaux binaires (`log_bin`) pour restauration point-in-time.

---

## 5. Rapport de synthèse (exemple résumé)
| Élément | Décision / Résultat |
|----------|---------------------|
| Moteurs | InnoDB / MEMORY / MyISAM selon usage |
| Sécurité | Accès restreint par utilisateur et rôle |
| Performance | Buffer pool ajusté, cache désactivé |
| Sauvegarde | Dump quotidien + binlogs activés |
| Préconisations | Surveiller espace disque, vérifier verrouillages InnoDB |
