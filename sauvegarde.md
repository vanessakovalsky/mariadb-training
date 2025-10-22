# 🛠️ ATELIER SAUVEGARDE STRUCTURE UNIQUEMENT

### Objectif
Sauvegarder uniquement la structure (schéma) sans les données, puis avec les données.

### Partie A : Structure seule

```bash
# Sauvegarde structure uniquement
mysqldump -u root -p --no-data atelier_backup > structure_only.sql

# Examiner le fichier
cat structure_only.sql
# On voit les CREATE TABLE mais pas d'INSERT
```

### Partie B : Structure + Données d'un utilisateur

```bash
# Créer d'abord un utilisateur
mysql -u root -p << EOF
CREATE USER IF NOT EXISTS 'utilisateur_test'@'localhost' IDENTIFIED BY 'password123';
GRANT ALL PRIVILEGES ON atelier_backup.* TO 'utilisateur_test'@'localhost';
FLUSH PRIVILEGES;
EOF

# Sauvegarde du schéma de cet utilisateur
mysqldump -u root -p \
  --no-data \
  --routines \
  --triggers \
  atelier_backup > schema_user.sql

# Variante : avec les données
mysqldump -u root -p \
  --single-transaction \
  --routines \
  --triggers \
  atelier_backup > schema_et_data_user.sql
```

### Utilisation

```bash
# Restaurer uniquement la structure (autre serveur, autre base)
mysql -u root -p -e "CREATE DATABASE atelier_backup_structure;"
mysql -u root -p atelier_backup_structure < structure_only.sql
```



# 🛠️ ATELIER : AUTOMATISATION AVEC SCRIPT ET SFTP 

### Objectif
Créer un script qui sauvegarde automatiquement une base toutes les heures, la transfère vers un serveur distant via SFTP, puis la restaure localement sans écraser le backup.

### Partie 1 : Script de sauvegarde (Linux)

**Créer le fichier** `/opt/scripts/backup_mariadb.sh`

```bash
#!/bin/bash

##############################################
# Script de sauvegarde automatique MariaDB
# Sauvegarde horaire avec transfert SFTP
##############################################

# Configuration
DB_USER="root"
DB_PASS="votre_mot_de_passe"  # ⚠️ Utiliser un fichier .my.cnf en production
DB_NAME="atelier_backup"
BACKUP_DIR="/var/backups/mariadb"
REMOTE_USER="backup_user"
REMOTE_HOST="serveur-backup.domaine.com"
REMOTE_DIR="/backups/mariadb"
LOG_FILE="/var/log/backup_mariadb.log"
RETENTION_DAYS=7

# Créer le répertoire si nécessaire
mkdir -p "$BACKUP_DIR"

# Nom du fichier avec horodatage
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${DB_NAME}_${TIMESTAMP}.sql"
BACKUP_PATH="${BACKUP_DIR}/${BACKUP_FILE}"

# Fonction de log
log_message() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log_message "=== Début de la sauvegarde ==="

# Sauvegarde avec mysqldump
log_message "Sauvegarde de la base ${DB_NAME}..."
mysqldump -u"$DB_USER" -p"$DB_PASS" \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    "$DB_NAME" > "$BACKUP_PATH" 2>> "$LOG_FILE"

if [ $? -eq 0 ]; then
    log_message "✓ Sauvegarde réussie : ${BACKUP_FILE}"
    
    # Compression
    log_message "Compression du fichier..."
    gzip "$BACKUP_PATH"
    BACKUP_PATH="${BACKUP_PATH}.gz"
    BACKUP_FILE="${BACKUP_FILE}.gz"
    
    # Transfert SFTP
    log_message "Transfert vers ${REMOTE_HOST}..."
    sftp -o "StrictHostKeyChecking=no" "${REMOTE_USER}@${REMOTE_HOST}" << EOF
cd ${REMOTE_DIR}
put ${BACKUP_PATH}
bye
EOF
    
    if [ $? -eq 0 ]; then
        log_message "✓ Transfert SFTP réussi"
    else
        log_message "✗ Erreur lors du transfert SFTP"
    fi
    
    # Nettoyage des anciennes sauvegardes locales
    log_message "Nettoyage des sauvegardes de plus de ${RETENTION_DAYS} jours..."
    find "$BACKUP_DIR" -name "${DB_NAME}_*.sql.gz" -mtime +${RETENTION_DAYS} -delete
    
else
    log_message "✗ Échec de la sauvegarde"
    exit 1
fi

log_message "=== Fin de la sauvegarde ===\n"
```

**Rendre le script exécutable**
```bash
chmod +x /opt/scripts/backup_mariadb.sh
```

**Sécuriser le mot de passe avec .my.cnf** (recommandé)
```bash
# Créer /root/.my.cnf
cat > /root/.my.cnf << EOF
[client]
user=root
password=votre_mot_de_passe
[mysqldump]
user=root
password=votre_mot_de_passe
EOF

chmod 600 /root/.my.cnf
```


### Partie 2 : Configuration de la tâche planifiée

**Linux - Crontab (toutes les heures)**
```bash
# Éditer le crontab
crontab -e

# Ajouter la ligne (exécution à chaque heure)
0 * * * * /opt/scripts/backup_mariadb.sh

# Vérifier
crontab -l
```


### Partie 3 : Script de restauration (sans écraser le backup)

**Linux** : `/opt/scripts/restore_mariadb.sh`

```bash
#!/bin/bash

##############################################
# Script de restauration depuis backup distant
# Sans écraser le fichier original
##############################################

# Configuration
DB_USER="root"
DB_NAME="atelier_backup_restored"  # Nouvelle base pour ne pas écraser
REMOTE_USER="backup_user"
REMOTE_HOST="serveur-backup.domaine.com"
REMOTE_DIR="/backups/mariadb"
LOCAL_TEMP_DIR="/tmp/restore_mariadb"
LOG_FILE="/var/log/restore_mariadb.log"

# Fichier à restaurer (le plus récent ou spécifié)
BACKUP_FILE="$1"  # Passé en paramètre

log_message() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Créer répertoire temporaire
mkdir -p "$LOCAL_TEMP_DIR"

log_message "=== Début de la restauration ==="

# Si pas de fichier spécifié, prendre le plus récent
if [ -z "$BACKUP_FILE" ]; then
    log_message "Recherche du backup le plus récent..."
    BACKUP_FILE=$(sftp "${REMOTE_USER}@${REMOTE_HOST}" << EOF | grep "atelier_backup_" | tail -1 | awk '{print $NF}'
cd ${REMOTE_DIR}
ls -t atelier_backup_*.sql.gz
bye
EOF
)
    log_message "Fichier trouvé : ${BACKUP_FILE}"
fi

# Télécharger le fichier (COPIE, pas déplacement)
log_message "Téléchargement de ${BACKUP_FILE}..."
sftp "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_DIR}/${BACKUP_FILE}" "${LOCAL_TEMP_DIR}/"

if [ $? -eq 0 ]; then
    log_message "✓ Téléchargement réussi"
    
    # Décompresser
    log_message "Décompression..."
    gunzip "${LOCAL_TEMP_DIR}/${BACKUP_FILE}"
    UNCOMPRESSED_FILE="${LOCAL_TEMP_DIR}/${BACKUP_FILE%.gz}"
    
    # Créer la nouvelle base
    log_message "Création de la base ${DB_NAME}..."
    mysql -u"$DB_USER" -e "DROP DATABASE IF EXISTS ${DB_NAME};"
    mysql -u"$DB_USER" -e "CREATE DATABASE ${DB_NAME};"
    
    # Restaurer
    log_message "Restauration en cours..."
    mysql -u"$DB_USER" "$DB_NAME" < "$UNCOMPRESSED_FILE"
    
    if [ $? -eq 0 ]; then
        log_message "✓ Restauration réussie dans ${DB_NAME}"
        
        # Vérification
        ROW_COUNT=$(mysql -u"$DB_USER" -N -e "SELECT SUM(TABLE_ROWS) FROM information_schema.TABLES WHERE TABLE_SCHEMA='${DB_NAME}';")
        log_message "Nombre total d'enregistrements : ${ROW_COUNT}"
    else
        log_message "✗ Erreur lors de la restauration"
    fi
    
    # Nettoyage du fichier temporaire local
    rm -f "$UNCOMPRESSED_FILE"
    log_message "Fichier temporaire supprimé (backup distant préservé)"
    
else
    log_message "✗ Échec du téléchargement"
    exit 1
fi

log_message "=== Fin de la restauration ===\n"
```

### Partie 4 : Test complet

**1. Configuration SSH sans mot de passe (clés SSH)**
```bash
# Sur le serveur source
ssh-keygen -t rsa -b 4096
ssh-copy-id backup_user@serveur-backup.domaine.com

# Tester
ssh backup_user@serveur-backup.domaine.com "ls -la"
```

**2. Test manuel du script**
```bash
# Exécuter la sauvegarde
/opt/scripts/backup_mariadb.sh

# Vérifier les logs
tail -f /var/log/backup_mariadb.log

# Vérifier sur le serveur distant
ssh backup_user@serveur-backup.domaine.com "ls -lh /backups/mariadb/"
```

**3. Tester la restauration**
```bash
# Restaurer depuis le dernier backup
/opt/scripts/restore_mariadb.sh

# Ou restaurer un fichier spécifique
/opt/scripts/restore_mariadb.sh atelier_backup_20251019_140000.sql.gz

# Vérifier
mysql -u root -p -e "USE atelier_backup_restored; SELECT * FROM clients;"
```

---
