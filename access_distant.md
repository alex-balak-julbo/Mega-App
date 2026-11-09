# Guide d'Accès Réseau Distant et Maintenance pour Serveur Déployé chez un Client

Ce guide explique comment configurer votre vieux PC pour pouvoir vous y connecter en SSH depuis chez vous, faire des mises à jour, et permettre à votre client d'accéder à l'application web, même si vous êtes à des centaines de kilomètres.

---

## 1. Les 2 méthodes pour accéder au serveur à distance

Pour vous connecter au serveur situé chez votre client, vous avez deux approches. La **Méthode B** est la plus moderne, la plus sûre et la plus simple.

### Méthode A : Le Redirection de Port (Port Forwarding) — La méthode classique
Elle consiste à configurer la box internet du client pour ouvrir un port sur l'extérieur.
* **Comment faire :** Vous attribuez une IP locale fixe au serveur chez le client (ex: `192.168.1.50`). Sur la box du client, vous redirigez le port externe `22` (SSH) vers le port `22` du serveur, et le port `80` (Web) vers le port `80` du serveur. Vous vous connectez ensuite via l'IP publique de la box du client.
* **Inconvénients :** Si le client n'a pas une IP publique fixe, son IP change tout le temps (il faut utiliser un service comme DynDNS). De plus, ouvrir le port SSH directement sur internet expose le serveur à des attaques de robots.

### Méthode B : Le VPN de Réseau Virtuel (Recommandé) — La méthode moderne
Vous installez un outil de réseau maillé (Mesh VPN) gratuit comme **Tailscale** ou **ZeroTier** sur le serveur et sur votre propre ordinateur de travail.
* **Comment ça marche :** Tailscale crée un réseau privé sécurisé entre vos machines. Peu importe où se trouve le serveur dans le monde et peu importe la box internet (même derrière une connexion 4G/5G ou un pare-feu d'entreprise strict), le serveur recevra une IP privée unique (ex: `100.11.22.33`).
* **Avantages :** 
  * Zéro configuration sur la box du client (pas de ports à ouvrir).
  * Sécurité maximale (seuls vos appareils autorisés peuvent voir le serveur).
  * L'IP fournie par Tailscale ne change **jamais**.

---

## 2. Configuration des outils de maintenance à distance

Avant d'envoyer le PC chez votre client, installez et configurez ces trois piliers sur le serveur :

### Outil 1 : SSH (Secure Shell) — Pour les mises à jour de code
C'est votre terminal de commande à distance. Il vous permettra de faire des `git pull`, de relancer les scripts Python ou de modifier la configuration.
* **Sous Linux (Ubuntu Server) :** SSH est installé par défaut. Vous vous connectez depuis votre PC via : `ssh utilisateur@ip_tailscale_du_serveur`.
* **Sous Windows :** Activez le serveur OpenSSH intégré (Paramètres > Applications > Fonctionnalités facultatives > Ajouter OpenSSH Server) et assurez-vous que le service est configuré sur "Automatique".

### Outil 2 : Git — Pour déployer vos mises à jour proprement
Ne transférez pas vos fichiers à la main en FTP. 
1. Créez un dépôt privé pour votre projet sur **GitHub** ou **GitLab**.
2. Sur le serveur, clonez ce dépôt.
3. Lorsque vous modifiez votre code chez vous, faites un `git push`. Pour mettre à jour le serveur du client à distance, connectez-vous en SSH sur le serveur et tapez simplement :
   ```bash
   cd /chemin/vers/votre/projet
   git pull
   ```
4. Redémarrez ensuite le script Python pour appliquer les changements.

### Outil 3 : Accès distant graphique (Optionnel mais rassurant)
Si vous choisissez de laisser Windows sur le vieux PC et que vous voulez voir l'écran à distance en cas de blocage :
* Installez un outil comme **RustDesk** (alternative open-source et gratuite à TeamViewer). Configurez un mot de passe fixe pour vous connecter en un clic.

---

## 3. Stratégie de sauvegarde à distance automatique (La Redondance)

Pour envoyer vos sauvegardes de base de données du serveur local (le vieux PC chez le client) vers votre serveur de secours à distance, planifiez une tâche automatisée (Cron sous Linux, ou Planificateur de tâches sous Windows).

### Exemple de script Python de sauvegarde automatique (`sauvegarde.py`)
Ce script extrait les données locales et les envoie sur votre base de données distante. Vous pouvez le programmer pour s'exécuter toutes les nuits à 2h du matin.

```python
import os
import subprocess
import datetime
# Si vous utilisez MySQL, vous pouvez automatiser un dump (export de sécurité)
# et l'envoyer vers un stockage Cloud (comme un bucket AWS S3 ou un serveur SFTP tiers)

def executer_sauvegarde():
    date_du_jour = datetime.datetime.now().strftime("%Y-%m-%d")
    fichier_backup = f"backup_stocks_{date_du_jour}.sql"
    
    # 1. Création de l'export de la base locale (Exemple avec MySQL)
    # (Sous Windows, assurez-vous que mysqldump est dans le PATH)
    commande_dump = f"mysqldump -u root -pMotDePasse BDD_Stocks > {fichier_backup}"
    os.system(commande_dump)
    
    # 2. Envoyer ce fichier vers votre propre serveur ou un Cloud sécurisé
    # Exemple conceptuel via protocole SFTP ou API de stockage
    print(f"Fichier {fichier_backup} créé et prêt à être envoyé à distance.")

if __name__ == "__main__":
    executer_sauvegarde()
```
