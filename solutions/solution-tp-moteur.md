# ✅ Solution - TP1 : Étude des moteurs de stockage MariaDB

## Objectifs récapitulés
- Comparer InnoDB, MyISAM et MEMORY en termes de transactions, durabilité et performances.
- Montrer les différences de comportement lors d'un rollback et d'opérations de suppression/compaction.

---

## 1) Contrôles et vérifications initiales
Afficher les moteurs :
```sql
SELECT table_name, engine FROM information_schema.tables WHERE table_schema = 'orders_db';
```
Exemple de sortie attendue :
```
orders          InnoDB
order_stats     MyISAM
session_cache   MEMORY
```

---

## 2) Tests transactionnels (InnoDB vs MyISAM)
**Création de deux tables identiques pour comparaison :**
```sql
CREATE TABLE test_innodb (
  id INT AUTO_INCREMENT PRIMARY KEY,
  val INT
) ENGINE=InnoDB;

CREATE TABLE test_myisam (
  id INT AUTO_INCREMENT PRIMARY KEY,
  val INT
) ENGINE=MyISAM;
```
**Test 1 : transaction & rollback (uniquement InnoDB supporte le rollback)**
```sql
-- Session A: ouvrir transaction et insérer
START TRANSACTION;
INSERT INTO test_innodb (val) VALUES (1),(2),(3);
INSERT INTO test_myisam (val) VALUES (1),(2),(3);

-- Annuler
ROLLBACK;
```
**Résultat attendu :**
- `SELECT COUNT(*) FROM test_innodb;` retourne **0** (les insertions ont été annulées).
- `SELECT COUNT(*) FROM test_myisam;` retourne **3** (MyISAM n'est pas transactionnel ; les insertions sont persistées même après ROLLBACK).

**Explication :** InnoDB est transactionnel (ACID) ; MyISAM ne l'est pas. ROLLBACK affecte seulement InnoDB.

---

## 3) Test de suppression et fragmentation
Insérer 10 000 lignes puis supprimer 1 000 dans chaque table :
```sql
INSERT INTO test_innodb (val) SELECT 1 FROM information_schema.columns LIMIT 10000;
INSERT INTO test_myisam (val) SELECT 1 FROM information_schema.columns LIMIT 10000;

DELETE FROM test_innodb WHERE id % 10 = 0;
DELETE FROM test_myisam WHERE id % 10 = 0;
```
**Observation attendue :**
- Taille des fichiers disque : MyISAM peut garder des fichiers fragmentés et présente un fichier `.MYD`/.MYI` ; InnoDB utilise tablespaces (`.ibd`) et le reclaim dépend de `innodb_file_per_table` et d'opérations `OPTIMIZE TABLE` pour récupérer l'espace.
- Pour MyISAM, `OPTIMIZE TABLE test_myisam;` réduit la taille ; pour InnoDB, `OPTIMIZE TABLE` reconstruira la table (si `innodb_file_per_table=ON` cela génère un nouveau .ibd plus compact).

---

## 4) Performance : SELECT COUNT(*) benchmark
Mesurer le temps :
```sql
SELECT BENCHMARK(1000, (SELECT COUNT(*) FROM test_innodb));
SELECT BENCHMARK(1000, (SELECT COUNT(*) FROM test_myisam));
```
**Résultats attendus (indicatifs) :**
- MyISAM peut répondre plus rapidement pour `COUNT(*)` sur tables sans index car stocke le nombre de lignes en interne (dans les anciennes versions), tandis qu'InnoDB calcule en lisant l'index primaire. Sur versions modernes, InnoDB avec index bien configuré est performant mais peut être plus coûteux pour un `COUNT(*)` non indexé.
- Interprète les résultats en fonction de ta machine : l'important est d'observer des différences et d'en expliquer la cause technique.

---

## 5) Use cases & recommandations
- `InnoDB` → tables critiques : commandes, transactions, contraintes FK, durabilité. Recommandé pour `orders`/`order_items`.
- `MyISAM` → lecture intensive, peu d'écritures, ou usages historiques où on recherche des performances brutes de lecture (cas très spécifiques).
- `MEMORY` → cache rapide, données volatiles qui peuvent être recalculées à chaud.
- Toujours privilégier InnoDB pour les nouvelles applications à cause de la gestion des transactions et de la sécurité des données.

---

## 6) Nettoyage (optionnel)
```sql
DROP TABLE IF EXISTS test_innodb, test_myisam;
```
