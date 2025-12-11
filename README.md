Raphaël GEAY et Hugo AGUER

# 💻 Projet d'Infrastructure - Réseau Segmenté et PRA

## 🎯 Objectifs du Projet

Ce projet vise à mettre en place une infrastructure réseau complète sur des machines virtuelles en Linux, en respectant les exigences du cahier des charges : segmentation réseau, sécurisation des accès, mise en place de services essentiels et implémentation d'un Plan de Reprise d'Activité (PRA) via un système de sauvegarde automatisé.

## 🗺️ Topologie Réseau et Segmentation

L'infrastructure est composée de 5 machines virtuelles (VMs) isolées en deux segments réseau distincts, reliés par un Routeur central.

Réseau 20.20.20.0 :
 - Client (Ubuntu, 20.20.20.20)

Réseau 10.10.10.0 :
 - Serveur Web (Debian, 10.10.10.3)
 - Serveur Sauvegarde (Debian, 10.10.10.4)
 - Serveur Monitoring (Debian, 10.10.10.5)

Routeur (Debian, 10.10.10.2, 20.20.20.2)

## 🌐 Le site web (http://10.10.10.3)

Site web géré par apache2 et accessible via http://10.10.10.3

## 💾 Sauvegarde et Plan de Reprise d'Activité (PRA)

La capacité de l'infrastructure à être restaurée en cas de défaillance majeure fonctionne comme ceci :

backup-web.sh :
 - Script qui sauvegarde tout le site + toute la config de apache2 de la VM Serveur Web

restauration-web.sh :
 - Script qui remet tous les fichiers de la dernière sauvegarde en place

backup-web.log et restauration-web.log :
 - Fichiers de log qui stocke tous les évènements de l'éxecution de backup-web.sh et de restauration-web.sh

Automatisation avec crontab (Tous les jours à deux heures du matin) :
```
0 2 * * * /home/user/Documents/backup-web.sh
```
