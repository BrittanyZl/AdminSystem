
# GLPI (Gestionnaire Libre de Parc Informatique) 

GLPI est une solution open source de gestion de parc informatique et de services d’assistance (helpdesk). C’est un outil très complet, utilisé principalement dans les entreprises, administrations et structures IT pour gérer efficacement leurs ressources informatiques, suivre les demandes d’assistance, et organiser les interventions des techniciens.

## Utilisation GLPI avec serveur Ubuntu 22.04

## INSTALLATION ET CONFIGURATION


- Mise à jour du serveur
```sh
sudo -i
apt update && apt -y dist-upgrade
```

- Installation et autorisation du ssh pour l'accés à distance
```sh
sudo apt install ssh
sudo ufw allow ssh
```

- Connexion à distance dans l'invite de commande
```sh
ssh NomDuServeur@IpServeur
```

- Installation d'Apache2
```sh
apt install apache2
```

- Installation de PHP 8.2
```sh
sudo apt install software-properties-common -y
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
```

- Installation des modules PHP 8.2
```sh
sudo apt install -y \
    php8.2-cli \
    php8.2-fpm \
    php8.2-common \
    php8.2-mysql \
    php8.2-curl \
    php8.2-gd \
    php8.2-intl \
    php8.2-mbstring \
    php8.2-xml \
    php8.2-zip \
    php8.2-bcmath \
    php8.2-gmp \
    php8.2-opcache \
    php8.2-apcu
```

- Vérification de PHP
```sh
php8.2 -v
php8.2 -m
```

- Installation des dépendances supplémentaires
```sh
sudo apt update && sudo apt install -y \
    libapache2-mod-php8.2 \
    hunspell \
    certbot \
    imagemagick \
    unzip \
    php8.2-opcache
```

- Vérification de PHP
```sh
php -v
```

- Configuration de PHP
```sh
ls /etc/php
```

- Création d'un fichier test
```sh
nano /var/www/html/main.php
```
```sh
    - son contenu
<?php phpinfo(); ?>     
```

- Redémarrer et vérifier le service apache2
```sh
systemctl restart apache2
systemctl status apache2
```

- Vérification de la page test
```sh
http://<IP_SERVER>/main.php
```

- Installation de MariaDB(ou MySQL)
```sh
apt -y install mariadb-server
systemctl restart mariadb
```

- Sécurisation de MariaDB
```sh
mysql_secure_installation
```

- Réponses à la sécurisation
```sh
Switch to unix_socket authentication [Y/n] n
Change the root password? [Y/n] n
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
```

- Rédemarrer MariaDB
```sh
systemctl restart mariadb
```

- Création de la base de données GLPI
```sh
mysql -u root
```

- A faire dans MySQL
```sh
    - Liste les base de données existantes
    show databases;
```
```sh
    - Création d'une base de données         
    create database glpi; 
```
```sh
    - Création d'un utilisateur avec son mot de passe    
    create user 'admin'@localhost identified by 'cfitech'; 
```
```sh
    - Attributions des permissions 
    grant all privileges on glpi.* to admin@localhost;
    flush privileges;
```
```sh
    - Sortir
    exit
```

- Rédemarrer MariaDB
```sh
systemctl restart mariadb
```

- Téléchargement et Extraction de GLPI
    - Depuis : https://github.com/glpi-project/glpi/releases?page=2
```sh 
wget https://github.com/glpi-project/glpi/releases/download/10.0.6/glpi-10.0.6.tgz
tar -xzf glpi-10.0.6.tgz -C /var/www/html/
ls /var/www/html/
```
- Configuration d'Apache pour GLPI
```sh
nano /etc/apache2/sites-available/000-default.conf
    - Son contenu
<VirtualHost *:80>
    DocumentRoot /var/www/html/glpi
</VirtualHost>
```
- Attribution des droits d'accès
```sh
chown -R www-data:www-data /var/www/html/glpi
ls -l /var/www/html
```

- Installation et Activation de SELInux
```sh
sudo apt install selinux-basics selinux-policy-default auditd -y
sudo selinux-activate
nano /etc/selinux/config
```

- Modification du fichier de configuration de SELinux
```sh
SELINUX=Enforcing
```

- Installation de modules PHP supplémentaires
```sh 
sudo apt install php8.2-ldap php8.2-imap php8.2-xmlrpc -y
```
- Redémmarrer le service Apache
```sh
systemctl restart apache2
```

- Installation de GLPI via l'interface web
    - Accès à : http://<IP_SERVER>/install/install.php
    - Sélectionner la langue et suivre l’assistant
    - Entrer les informations de la base de données
    - Valider l’installation

    - Entrer ces données
```sh
        Serveur : localhost
        Utilisateur : admin
        Mot de passe : cfitech
        Base de données : glpi
```

- Sécurisation de GLPI
    - Supprimer le script d’installation
```sh
rm -fr /var/www/html/glpi/install/install.php
```

- Modifier php.ini pour renforcer la sécurité
```sh
nano /etc/php/8.2/apache2/php.ini
    - Ajouter/Modifier
session.cookie_httponly = on
```

- Redémarrer Apache
```sh
systemctl restart apache2
```

- Accès à GLPI
    - L’interface est disponible à l’adresse http://<IP_SERVER>
    - Identifiant par défaut : glpi
    - Mot de passe : glpi  

- Installation du Plugin FusionInventory
```sh
    - Se placer dans le dossier des plugins GLPI
    cd /var/www/html/glpi/plugins
```
```sh
- Télécharger la dernière version du plugin
wget https://github.com/fusioninventory/fusioninventory-for-glpi/releases/download/glpi10.0.6%2B1.1/fusioninventory-10.0.6+1.1.tar.bz2
```
```sh
- Extraire l'archive
tar -xjf fusioninventory-10.0.6+1.1.tar.bz2
```

```sh
- Configurer les permissions
chown -R www-data:www-data fusioninventory
```

- Configuration d'Apache
```sh
    - Créer la configuration virtuelle
    nano /etc/apache2/sites-available/glpi.conf
```
    - Ajouter la configuration suivante
```sh
<VirtualHost *:80>
    DocumentRoot /var/www/html/glpi
    <Directory /var/www/html/glpi>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

- Exécuter ceci
```sh
a2ensite glpi.conf
a2dissite 000-default.conf
systemctl restart apache2
curl -I http://localhost/plugins/fusioninventory/front/plugin_fusioninventory.communication.php
```

- Configurez le cron pour les tâches automatique (optionnel mais recommandé)
```sh
crontab -e
```
    - Ajouter
```sh
*/1 * * * * php /var/www/html/glpi/front/cron.php
```

- Redémarrer le service Cron
```sh
/etc/init.d/cron restart
systemctl restart cron
```
- Activation dans GLPI
    - Se connecter à GLPI (http://)
    - Aller dans Configuration > Plugins
    - Activer FusionInventory
    - Aller dans Configuration > Action automatique
    - Rechercher taskscheduler et cliquer sur Exécuter

- Installation des Agents
    - Pour Windows
        - Télécharger l'agent depuis https://github.com/fusioninventory/fusioninventory-agent/releases/tag/2.6
        - Installer l'agent
        - Configurer C:\Program Files\FusionInventory-Agent\agent.cfg avec :
Mode servers = http://<IP_SERVER>/plugins/fusioninventory/front/plugin_fusioninventory.communication.php

- Redémarrer le service
´´´sh
net stop FusionInventory-Agent && net start FusionInventory-Agent
´´´
- Pour Linux (Debian/Ubuntu)
```sh
wget https://github.com/fusioninventory/fusioninventory-agent/releases/download/2.6/fusioninventory-agent_2.6-1_all.deb
sudo apt --fix-broken install
sudo apt install hwdata


sudo apt install libnet-cups-perl libnet-ip-perl libwww-perl libparse-edid-perl \
libproc-daemon-perl libuniversal-require-perl libfile-which-perl libxml-treepp-perl \
libxml-xpath-perl libyaml-perl libtext-template-perl libhttp-daemon-perl \
libyaml-tiny-perl libsocket-getaddrinfo-perl

sudo dpkg -i fusioninventory-agent_2.6-1_all.deb
sudo apt install -f
```

- Configuration
```sh
sudo nano /etc/fusioninventory/agent.cfg
```

- Ajouter 
```sh
<server url="http://<IP_SERVER>/plugins/fusioninventory/front/plugin_fusioninventory.communication.php"/>
```

- Exécuter
```sh
sudo systemctl enable fusioninventory-agent
sudo systemctl start fusioninventory-agent
```

- Vérification dans GLPI
    - Dans GLPI, aller dans Plugins > FusionInventory > Inventaire
    - Les équipements devraient apparaître dans les 5-10 minutes

- Pour gérer le Marketplace
    - Aller dans Configuration > General > GLPI Network
    - Activer/désactiver selon besoins

- Pour activer le marketplace
    - Aller dans Configuration > Plugins > Marketlace
    - on voit message avec lien pour GLPI Network
    - Cliquer sur ce lien
    - S'enregistrer
    - Enregistrement
    - Copie la clef 
    - Retour dans interface GLPI
    - Coller dans Configuration > Général > GLPI Network
