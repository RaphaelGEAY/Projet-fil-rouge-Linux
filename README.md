Raphaël GEAY et Hugo AGUER

# 💻 Projet d'Infrastructure - Réseau Segmenté et PRA

## 🎯 Objectifs du Projet

Ce projet vise à mettre en place une infrastructure réseau complète sur des machines virtuelles en Linux, en respectant les exigences du cahier des charges : segmentation réseau, sécurisation des accès, mise en place de services essentiels et implémentation d'un Plan de Reprise d'Activité (PRA) via un système de sauvegarde automatisé.

## 🗺️ Topologie Réseau et Segmentation

L'infrastructure est composée de 5 machines virtuelles, réparties sur deux réseaux distincts, reliés par un Routeur central.

Le Routeur est la seule machine à posséder une carte NAT, donc toutes les autres machines doivent passer par le Routeur pour accéder à internet.

**Routeur (Debian, NAT, 10.10.10.2, 20.20.20.2)**

Réseau 20.20.20.0 :
 - Client (Ubuntu, 20.20.20.20)

Réseau 10.10.10.0 :
 - Serveur Web (Debian, 10.10.10.3)
 - Serveur Sauvegarde (Debian, 10.10.10.4)
 - Serveur Monitoring (Debian, 10.10.10.5)

**Configuration du routeur**
```
net.ipv4.ip_forward = 1 # Dans le fichier /etc/sysctl.conf
```
```
sudo sysctl -p
```
```
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
```

**Configuration réseau pour le Client (`/etc/netplan/01-network-manager-all.yaml`)**
```
# Let NetworkManager manage all devices on this system
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s8:
      dhcp4: no
      addresses:
        - 20.20.20.20/24
      routes:
        - to: default
          via: 20.20.20.2
      nameservers:
        addresses: [20.20.20.2]
```

**Appliquer**
```
sudo netplan apply
```

**Configuration réseau pour les machines Debian (ex: Routeur) (`/etc/network/interfaces`)**
```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

auto enp0s3
iface enp0s3 inet dhcp

auto enp0s8
iface enp0s8 inet static
    address 10.10.10.2
    netmask 255.255.255.0
    network 10.10.10.0
    broadcast 10.10.10.255
    dns-nameservers 127.0.0.1

auto enp0s9
iface enp0s9 inet static
    address 20.20.20.2
    netmask 255.255.255.0
    network 20.20.20.0
    broadcast 20.20.20.255
    dns-nameservers 127.0.0.1
```

**Configuration DNS pour les machines Debian (ex: Routeur) (`/etc/resolv.conf`)**
```
nameserver 127.0.0.1
```

**Appliquer**
```
sudo systemctl restart networking
```

## 📡 DNS

Le DNS se fait sur le Routeur (10.10.10.2, 20.20.20.2)

Le service **Bind9** gère la zone `monsupersite.com`.

**Fichier de zone (`/etc/bind/db.monsupersite.com`) :**
Ce fichier fait la correspondance entre les noms de machines et leurs IPs respectives.
```
$TTL 604800
@       IN      SOA     ns.monsupersite.com. root.monsupersite.com. (
                              2026012601 ; Serial (Date du jour + version 01)
                              604800     ; Refresh
                              86400      ; Retry
                              2419200    ; Expire
                              604800 )   ; Negative Cache TTL

; --- SERVEURS DE NOMS ---
@       IN      NS      ns.monsupersite.com.
ns      IN      A       10.10.10.2

; --- ENREGISTREMENTS A (LES MACHINES PHYSIQUES/VM) ---
web     IN      A       10.10.10.3
backup  IN      A       10.10.10.4
monitor IN      A       10.10.10.5
client  IN      A       20.20.20.20

; --- ALIAS (POUR LE WEB) ---
@       IN      A       10.10.10.3
www     IN      CNAME   web.monsupersite.com.
```

🌐 Site Web & Base de Données

Le site est un coffre-fort numérique sécurisé déployé la machine Serveur Web (10.10.10.3)

 - Disponible sur tout le réseau via l'URL personnalisée https://monsupersite.com
 - Chiffrement des flux via SSL/TLS (Port 443) avec certificats dédiés
 - Utilisation de Docker Compose pour la conteneurisation
 - Utilisation de MariaDB pour la base de donnée

**`~/MonSuperSite/html/index.php`**
```
<?php
// --- CONFIGURATION DATABASE ---
$host = 'mariadb_site';
$db   = 'monsupersite';
$user = 'root';
$pass = 'SecuVault_2026';

// Connexion à la base
try {
    $pdo = new PDO("mysql:host=$host;dbname=$db;charset=utf8", $user, $pass);
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
} catch (Exception $e) {
    die('<div style="color:red; text-align:center; padding:20px; background:black; border:2px solid red;">
            [ERREUR SYSTEME] : Connexion au Coffre-fort échouée.
         </div>');
}

// --- LOGIQUE PRG (Post-Redirect-Get) ---
// Évite le message "Renvoyer le formulaire" sur Firefox
if (!empty($_POST['codename']) && !empty($_POST['secret'])) {
    $stmt = $pdo->prepare("INSERT INTO vault (codename, secret, date_stored) VALUES (?, ?, NOW())");
    $stmt->execute([$_POST['codename'], $_POST['secret']]);
    
    // Redirection immédiate pour vider le cache d'envoi
    header("Location: " . $_SERVER['PHP_SELF']);
    exit(); 
}

// Récupération des secrets
$secrets = $pdo->query("SELECT * FROM vault ORDER BY date_stored DESC LIMIT 5")->fetchAll();
?>

<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>SECURE VAULT | STORAGE</title>
    
    <meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate">
    <meta http-equiv="Pragma" content="no-cache">
    <meta http-equiv="Expires" content="0">

    <style>
        body {
            /* Fond sombre tech */
            background-color: #050505;
            background-image: 
                linear-gradient(rgba(255, 0, 0, 0.05) 1px, transparent 1px),
                linear-gradient(90deg, rgba(255, 0, 0, 0.05) 1px, transparent 1px);
            background-size: 20px 20px;
            color: #ff3333; /* Rouge "Alerte" */
            font-family: 'Courier New', Courier, monospace; /* Police style terminal */
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
        }
        .vault-panel {
            background: rgba(10, 0, 0, 0.9);
            border: 2px solid #ff3333;
            border-radius: 5px;
            padding: 40px;
            width: 600px;
            box-shadow: 0 0 50px rgba(255, 0, 0, 0.2);
            position: relative;
        }
        /* Effet "TOP SECRET" en haut */
        .vault-panel::before {
            content: "TOP SECRET // CLASSIFIED";
            position: absolute;
            top: -15px;
            left: 20px;
            background: #050505;
            padding: 0 10px;
            color: #ff3333;
            font-weight: bold;
            border: 1px solid #ff3333;
        }

        h1 { 
            text-align: center;
            border-bottom: 1px dashed #ff3333; 
            padding-bottom: 20px;
            letter-spacing: 2px;
            text-shadow: 0 0 10px #ff3333;
        }

        .input-group {
            margin-bottom: 15px;
            display: flex;
            gap: 10px;
        }

        input[type="text"] {
            background: #1a0000;
            border: 1px solid #550000;
            color: #ff8888;
            padding: 15px;
            width: 100%;
            font-family: 'Courier New', monospace;
            outline: none;
            transition: 0.3s;
        }
        input[type="text"]:focus {
            border-color: #ff3333;
            box-shadow: 0 0 15px rgba(255, 51, 51, 0.3);
        }

        button {
            width: 100%;
            padding: 15px;
            background: #330000;
            color: #ff3333;
            border: 1px solid #ff3333;
            font-weight: bold;
            cursor: pointer;
            text-transform: uppercase;
            letter-spacing: 2px;
        }
        button:hover {
            background: #ff3333;
            color: black;
            box-shadow: 0 0 20px #ff3333;
        }

        .data-list {
            margin-top: 30px;
            border-top: 1px solid #330000;
            padding-top: 10px;
        }
        .secret-row {
            padding: 10px;
            border-bottom: 1px dotted #550000;
            display: flex;
            justify-content: space-between;
        }
        .secret-row:hover { background: rgba(255, 0, 0, 0.1); }
        .codename { font-weight: bold; color: #fff; }
    </style>
</head>
<body>

    <div class="vault-panel">
        <h1>STOCKAGE SÉCURISÉ</h1>
        
        <form method="POST">
            <div class="input-group">
                <input type="text" name="codename" placeholder="NOM DE CODE (ex: PROJET X)" required style="flex: 1;">
                <input type="text" name="secret" placeholder="DONNÉE SENSIBLE (ex: MDP ROOT)" required style="flex: 2;">
            </div>
            <button type="submit">🔒 CHIFFRER ET ARCHIVER</button>
        </form>

        <div class="data-list">
            <div style="margin-bottom: 10px; color: #555;">DERNIÈRES ENTRÉES DANS LE COFFRE :</div>
            <?php foreach($secrets as $entry): ?>
                <div class="secret-row">
                    <span class="codename">> <?= htmlspecialchars($entry['codename']) ?></span>
                    <span><?= htmlspecialchars($entry['secret']) ?></span>
                    <span style="font-size: 0.8em; color: #888;">[<?= $entry['date_stored'] ?>]</span>
                </div>
            <?php endforeach; ?>
        </div>
    </div>

</body>
</html>
```

**`~/MonSuperSite/docker-compose.yml`**
```
services:
  # --- SERVEUR WEB (Apache + PHP + SSL) ---
  web-server:
    image: php:8.2-apache
    container_name: site_web
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./html:/var/www/html
      - ./ssl/server.crt:/etc/apache2/ssl/server.crt
      - ./ssl/server.key:/etc/apache2/ssl/server.key
      - /etc/localtime:/etc/localtime:ro
    command: >
      sh -c "a2enmod ssl && a2enmod rewrite &&
             a2ensite default-ssl &&
             sed -i 's|SSLCertificateFile.*|SSLCertificateFile /etc/apache2/ssl/server.crt|g' /etc/apache2/sites-available/default-ssl.conf &&
             sed -i 's|SSLCertificateKeyFile.*|SSLCertificateKeyFile /etc/apache2/ssl/server.key|g' /etc/apache2/sites-available/default-ssl.conf &&
             docker-php-ext-install pdo pdo_mysql &&
             apache2-foreground"
    networks:
      - secure-net
    depends_on:
      - database

  # --- BASE DE DONNÉES (MariaDB) ---
  database:
    image: mariadb:latest
    container_name: mariadb_site
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: SecuVault_2026
      MYSQL_DATABASE: monsupersite
    volumes:
      - db_data:/var/lib/mysql
      - /etc/localtime:/etc/localtime:ro
    networks:
      - secure-net

networks:
  secure-net:
    driver: bridge

volumes:
  db_data:
```

## 💾 Sauvegarde et Plan de Reprise d'Activité (PRA)

La capacité de l'infrastructure à être restaurée en cas de défaillance majeure fonctionne comme ceci :

backup.sh :
 - Script qui sauvegarde l'intégralité du répertoire du site sur la VM Serveur Sauvegarde (Code source, Base de données, Certificats SSL et Docker Compose)
 - Utilisation de rsync + clé SSH

restauration.sh :
 - Script qui remet tous les fichiers de la dernière sauvegarde en place
 - Utilisation de rsync + clé SSH

backup.log et restauration.log :
 - Fichiers de log qui stocke tous les évènements de l'éxecution de backup.sh et de restauration.sh

Automatisation avec crontab (Tous les jours à deux heures du matin) :
```
0 2 * * * /home/user/Scripts/backup.sh
```
```
crontab -e # Modifier cron
```
```
crontab -l # Lister cron
```

## Scripts :

**`~/Scripts/backup.sh`**
```
#!/bin/bash

LOG="$HOME/Scripts/Logs/backup.log"

# --- Configuration ---
SOURCE_DIR="$HOME/MonSuperSite"
DB_CONTAINER="mariadb_site"
DB_NAME="monsupersite"
DB_ROOT_PASS="SecuVault_2026"

# Serveur de destination (Serveur Sauvegarde)
DEST_USER="user"
DEST_HOST="10.10.10.4"
DEST_BASE="/home/user/Sauvegardes"

# --- Début du processus ---
echo "=== Début backup Docker $(date) ===" | tee -a "$LOG"

# 1. Sauvegarde de la Base de Données (Dump à chaud)
echo "--- Export de la base de données MariaDB ---" | tee -a "$LOG"
# On demande au conteneur de créer un fichier SQL dans le dossier du projet
if docker exec $DB_CONTAINER mariadb-dump -u root -p"$DB_ROOT_PASS" $DB_NAME > "$SOURCE_DIR/database.sql"; then
    echo "Dump SQL réussi." | tee -a "$LOG"
else
    echo "ERREUR : Échec du dump SQL. Le conteneur est-il allumé ?" | tee -a "$LOG"
    exit 1
fi

# 2. Préparation du serveur distant
echo "Tentative de connexion à $DEST_HOST..." | tee -a "$LOG"
ssh $DEST_USER@$DEST_HOST "mkdir -p $DEST_BASE"

# 3. Sauvegarde de TOUT le dossier projet (HTML, Config, Docker-compose et SQL)
echo "--- Synchronisation des fichiers vers le serveur de sauvegarde ---" | tee -a "$LOG"
# On envoie tout le dossier Conteneurisation vers le serveur de backup
rsync -avz --no-perms --no-owner --no-group --delete "$SOURCE_DIR/" $DEST_USER@$DEST_HOST:$DEST_BASE/ 2>&1 | tee -a "$LOG"

echo "=== Fin backup $(date) ===" | tee -a "$LOG"
```

**`~/Scripts/restauration.sh`**
```
#!/bin/bash

LOG="$HOME/Scripts/Logs/restauration.log"

# --- Configuration ---
PROJECT_DIR="$HOME/MonSuperSite"
DB_CONTAINER="mariadb_site"
DB_NAME="monsupersite"
DB_ROOT_PASS="SecuVault_2026"

# Serveur Source (Serveur Sauvegarde)
SOURCE_USER="user"
SOURCE_HOST="10.10.10.4"
SOURCE_PATH="/home/user/Sauvegardes"

echo "=== Début de la restauration Docker $(date) ===" | tee -a "$LOG"

# 1. Arrêt des conteneurs (pour éviter les conflits d'écriture)
echo "--- Arrêt des services Docker ---" | tee -a "$LOG"
cd "$PROJECT_DIR" || exit
docker compose down | tee -a "$LOG"

# 2. Récupération des fichiers depuis le serveur de sauvegarde
echo "--- Récupération des fichiers depuis la sauvegarde ---" | tee -a "$LOG"
# On écrase le dossier local par celui de la sauvegarde
if rsync -avz --no-perms --no-owner --no-group --delete $SOURCE_USER@$SOURCE_HOST:$SOURCE_PATH/ "$PROJECT_DIR/" 2>&1 | tee -a "$LOG"; then
    echo "Fichiers restaurés avec succès." | tee -a "$LOG"
else
    echo "ERREUR CRITIQUE lors du rsync." | tee -a "$LOG"
    exit 1
fi

# 3. Redémarrage des conteneurs
echo "--- Redémarrage de l'infrastructure ---" | tee -a "$LOG"
docker compose up -d | tee -a "$LOG"

# 4. Attente et Importation de la base de données
echo "Attente du démarrage de la base de données (15s)..." | tee -a "$LOG"
sleep 15 # On laisse le temps à MariaDB de s'initialiser

if [ -f "$PROJECT_DIR/database.sql" ]; then
    echo "--- Restauration des données SQL dans le conteneur ---" | tee -a "$LOG"
    # On injecte le fichier SQL à l'intérieur du conteneur
    cat "$PROJECT_DIR/database.sql" | docker exec -i $DB_CONTAINER mariadb -u root -p"$DB_ROOT_PASS" $DB_NAME
    
    if [ $? -eq 0 ]; then
        echo "Base de données restaurée : OK" | tee -a "$LOG"
    else
        echo "Erreur lors de l'import SQL." | tee -a "$LOG"
    fi
else
    echo "Aucun fichier SQL trouvé pour la restauration." | tee -a "$LOG"
fi

echo "=== Fin de la restauration $(date). Vérifiez le site. ===" | tee -a "$LOG"
```

**`~/Scripts/vider.sh`** (vide la base de données)
```
docker exec -it mariadb_site mariadb -u root -p"SecuVault_2026" -e "TRUNCATE TABLE monsupersite.vault;"
```
