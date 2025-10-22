# ✅ Solution - TP2 : Configuration, sauvegarde et optimisation MariaDB

## Objectifs récapitulés
- Mesurer l'impact des paramètres (`innodb_buffer_pool_size`, `query_cache_size`, `max_connections`).
- Mettre en place une sauvegarde/restauration et détecter les requêtes lentes.

---

## 1) Vérification initiale des variables
Commandes :
```sql
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW VARIABLES LIKE 'query_cache_size';
SHOW VARIABLES LIKE 'max_connections';
```
Exemple de sortie :
```
innodb_buffer_pool_size | 134217728
query_cache_size        | 1048576
max_connections         | 151
```

Interprétation : si le serveur a 4GB RAM, `innodb_buffer_pool_size` à 128MB est faible pour des données volumineuses.

---

## 2) Ajustements recommandés (exemple pour serveur 4GB RAM)
- `innodb_buffer_pool_size` → ~60–70% de la RAM si serveur dédié : **1.5–2.5 GB** (ici on propose 1GB à titre conservatoire si cohabitation avec OS & autres services).
- `query_cache_size` → souvent **désactivé (0)** pour InnoDB moderne car provoque invalidations et lock contention.
- `max_connections` → dimensionné selon charge ; si 151 est suffisant, garder, sinon augmenter en surveillant la RAM (chaque connexion consomme mémoire).

Application temporaire :
```sql
SET GLOBAL innodb_buffer_pool_size = 1073741824; -- 1GB
SET GLOBAL query_cache_size = 0;
SET GLOBAL max_connections = 300;
```

**Attention :** modifier `innodb_buffer_pool_size` à chaud n'est possible que sur certaines versions et peut nécessiter redémarrage pour prise en compte complète. Toujours valider la compatibilité de la version MariaDB.

---

## 3) Tests avant / après
Mesure : exécuter une requête lourde et mesurer le temps (ou utiliser `\T` dans le client mysql pour journaliser temps) :
```sql
SELECT COUNT(*) FROM sales WHERE amount > 100;
```
Vérifier les compteurs InnoDB :
```sql
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_%';
```
Attendu : augmentation du hit-rate du buffer pool (plus de `read_hits` par rapport à `read_requests`), réduction des lectures disque si buffer pool bien dimensionné.

---

## 4) Sauvegarde et restauration
Dump complet :
```bash
mysqldump -u root -p perf_test_db > /backup/perf_test_db_dump.sql
```
Restauration :
```bash
mysql -u root -p perf_test_db < /backup/perf_test_db_dump.sql
```
Test : supprimer une table (ex. `sales_archive`) puis restaurer et vérifier `COUNT(*)` pour s'assurer de la présence des données.

---

## 5) Requêtes lentes & diagnostic
Activer log requêtes lentes :
```sql
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SHOW VARIABLES LIKE 'slow_query_log_file';
```
Consulter le fichier de log (terminal) :
```bash
sudo tail -n 200 /var/log/mysql/mysql-slow.log
```
Utiliser `EXPLAIN` pour optimiser les requêtes identifiées :  
```sql
EXPLAIN SELECT * FROM sales WHERE amount > 100;
```

---

## 6) Maintenance et recommandations
- Planifier `OPTIMIZE TABLE` pour MyISAM ou tables InnoDB si `innodb_file_per_table=ON` pour récupérer l'espace.
- Mettre en place des dumps réguliers + rotation (7 jours minimum) et activer `log_bin` pour PITR si besoin.
- Surveiller : `Threads_connected`, `Innodb_row_lock_time`, `Innodb_buffer_pool_reads`.

---

## 7) Scripts utiles (récapitulatif)

**Afficher tailles par base :**
```sql
SELECT table_schema AS db,
       ROUND(SUM(data_length + index_length)/1024/1024,2) AS size_MB
FROM information_schema.tables
GROUP BY table_schema
ORDER BY size_MB DESC;
```

**Forcer OPTIMIZE :**
```sql
OPTIMIZE TABLE test_myisam;
OPTIMIZE TABLE sales;
```

**Nettoyage (optionnel) :**
```sql
DROP DATABASE IF EXISTS perf_test_db;
```
