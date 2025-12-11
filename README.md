Raphaël GEAY et Hugo AGUER

# 💻 Projet d'Infrastructure - Réseau Segmenté et PRA

## 🎯 Objectifs du Projet

Ce projet vise à mettre en place une infrastructure réseau complète sur des machines virtuelles en Linux, en respectant les exigences du cahier des charges : segmentation réseau, sécurisation des accès, mise en place de services essentiels et implémentation d'un Plan de Reprise d'Activité (PRA) via un système de sauvegarde automatisé.

## 🗺️ Topologie Réseau et Segmentation

L'infrastructure est composée de 5 machines virtuelles (VMs) isolées en deux segments réseau distincts, reliés par un Routeur central.

**Routeur (Debian, 10.10.10.2, 20.20.20.2)**

Réseau 20.20.20.0 :
 - Client (Ubuntu, 20.20.20.20)

Réseau 10.10.10.0 :
 - Serveur Web (Debian, 10.10.10.3)
 - Serveur Sauvegarde (Debian, 10.10.10.4)
 - Serveur Monitoring (Debian, 10.10.10.5)

## 🌐 Le site web (https://10.10.10.3)

Site web géré par apache2 et accessible via https://10.10.10.3

## 💾 Sauvegarde et Plan de Reprise d'Activité (PRA)

La capacité de l'infrastructure à être restaurée en cas de défaillance majeure fonctionne comme ceci :

backup-web.sh :
 - Script qui sauvegarde tout le site + toute la config de apache2 sur la VM Serveur Sauvegarde
 - Utilisation de rsync + clé SSH

restauration-web.sh :
 - Script qui remet tous les fichiers de la dernière sauvegarde en place
 - Utilisation de rsync + clé SSH

backup-web.log et restauration-web.log :
 - Fichiers de log qui stocke tous les évènements de l'éxecution de backup-web.sh et de restauration-web.sh

Automatisation avec crontab (Tous les jours à deux heures du matin) :
```
0 2 * * * /home/user/Documents/backup-web.sh
```

## Scripts :

**```backup.sh```**
```
#!/bin/bash

LOG="$HOME/Documents/backup-web.log"

# Serveur de destination
DEST_USER="user"
DEST_HOST="10.10.10.4"  # Assurez-vous que c'est bien l'IP du Serveur Sauvegarde !
DEST_BASE="/home/user/Documents/backup/web"
DEST_HTML="$DEST_BASE/html"
DEST_APACHE="$DEST_BASE/apache2"

# --- Début du processus ---
echo "=== Début backup $(date) ===" | tee -a "$LOG"
echo "Tentative de connexion à $DEST_HOST et création des répertoires..."

# Création des dossiers sur le serveur distant
# On affiche la commande ssh dans le terminal pour l'effet "stylé"
if ssh $DEST_USER@$DEST_HOST "mkdir -p $DEST_HTML $DEST_APACHE"; then
    echo "Répertoires distants créés ou déjà existants." | tee -a "$LOG"
else
    echo "Erreur lors de la création des répertoires distants. Vérifiez SSH et les permissions." | tee -a "$LOG"
fi

echo "--- Démarrage de la sauvegarde du site web (/var/www/html) ---" | tee -a "$LOG"
# Sauvegarde du site web - Utilisation de tee pour afficher et loguer
rsync -avz /var/www/html/ $DEST_USER@$DEST_HOST:$DEST_HTML/ 2>&1 | tee -a "$LOG"

echo "--- Démarrage de la sauvegarde de la configuration Apache (/etc/apache2) ---" | tee -a "$LOG"
# Sauvegarde de la configuration Apache - Utilisation de tee pour afficher et loguer
rsync -avz /etc/apache2/ $DEST_USER@$DEST_HOST:$DEST_APACHE/ 2>&1 | tee -a "$LOG"

echo "=== Fin backup $(date) ===" | tee -a "$LOG"
```

**```restauration.sh```**
```
#!/bin/bash

# --- Configuration (à adapter au chemin où vous exécutez le script) ---
LOG="/home/user/Documents/restauration-web.log" # Nouveau fichier log pour la restauration
SOURCE_USER="user"
SOURCE_HOST="10.10.10.4" # Serveur de Sauvegarde
SOURCE_BASE="/home/user/Documents/backup/web"
SOURCE_HTML="$SOURCE_BASE/html"
SOURCE_APACHE="$SOURCE_BASE/apache2"

# --- Vérification et Nettoyage (optionnel mais conseillé pour la restauration) ---
echo "=== Début de la restauration $(date) ===" | tee -a "$LOG"
echo "Vérification des accès SSH sans mot de passe vers le Serveur Sauvegarde ($SOURCE_HOST)..." | tee -a "$LOG"

if ! ssh -q $SOURCE_USER@$SOURCE_HOST exit; then
    echo "ERREUR CRITIQUE : La connexion SSH sans mot de passe au Serveur Sauvegarde a échoué." | tee -a "$LOG"
    echo "Veuillez vérifier les clés SSH et la connectivité." | tee -a "$LOG"
    exit 1
fi

# --- Restauration du site web ---
echo "--- Démarrage de la restauration du site web vers /var/www/html/ ---" | tee -a "$LOG"
# Utilisation de 'sudo' car /var/www/html/ nécessite généralement des droits root pour écrire
# Attention : rsync inversé (Source distante -> Destination locale)
if sudo rsync -avz $SOURCE_USER@$SOURCE_HOST:$SOURCE_HTML/ /var/www/html/ 2>&1 | tee -a "$LOG"; then
    echo "Restauration du site web : OK" | tee -a "$LOG"
else
    echo "Restauration du site web : ÉCHEC. Voir le log pour les détails." | tee -a "$LOG"
fi

# --- Restauration de la configuration Apache ---
echo "--- Démarrage de la restauration de la configuration Apache vers /etc/apache2/ ---" | tee -a "$LOG"
# Utilisation de 'sudo' car /etc/apache2/ nécessite des droits root pour écrire
if sudo rsync -avz $SOURCE_USER@$SOURCE_HOST:$SOURCE_APACHE/ /etc/apache2/ 2>&1 | tee -a "$LOG"; then
    echo "Restauration de la configuration Apache : OK" | tee -a "$LOG"
else
    echo "Restauration de la configuration Apache : ÉCHEC. Voir le log pour les détails." | tee -a "$LOG"
fi

# --- Finalisation et redémarrage du service ---
echo "--- Redémarrage du service Apache2 pour appliquer les nouvelles configurations ---" | tee -a "$LOG"
if sudo systemctl restart apache2 2>&1 | tee -a "$LOG"; then
    echo "Service Apache2 redémarré avec succès !" | tee -a "$LOG"
else
    echo "ÉCHEC du redémarrage d'Apache2. Vérifiez la syntaxe des fichiers restaurés." | tee -a "$LOG"
fi

echo "=== Fin de la restauration $(date). Veuillez vérifier l'accessibilité du site. ===" | tee -a "$LOG"
```
