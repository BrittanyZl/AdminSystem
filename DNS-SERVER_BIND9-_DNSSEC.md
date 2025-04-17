# Mise à jour
sudo apt update && sudo apt upgrade -y

# Installer ssh
sudo apt install ssh -y

# Se connecter avec ssh dans le terminal
ssh NomUser@ipAddress

# installer service bind9 et ses utilitaires
sudo apt install bind9 bind9utils bind9-doc -y

# Activer le service bind9 
sudo systemctl start bind9
sudo systemctl status bind

# Autoriser le port 53 dans les pare-feu
sudo ufw allow 53
sudo ufw allow 53/tcp
sudo ufw reload
sudo ufw status
sudo ufw enable

# Configurez une adresse IP statique pour le serveur primaire en éditant le fichier Netplan
sudo nano /etc/netplan/01-netcfg.yaml

# Mettre la configuration suivante dans le fichier Netplan (vérifier votre carte réseau et le modifier si besoin)
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.1.2/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [192.168.1.2]

# Appliquer la configuration
sudo netplan apply

# Faire les étapes précédentes sur le serveur secondaire

# Retour sur serveur primaire
# Configuration de bind9 

sudo nano /etc/bind/named.conf.options

# Modifier et ajouter les options dans le fichier /etc/bind/named.conf.options
options {
    directory "/var/cache/bind";

    forwarders {
        8.8.8.8;
        8.8.4.4;
    };

    dnssec-validation auto;

    listen-on { 192.168.1.2; };           # Adresse IP du serveur primaire.
    listen-on-v6 { none; };               # Désactiver IPv6 si non utilisé.
    allow-query { 192.168.1.0/24; };      # Limiter les requêtes au réseau local.
    allow-recursion { 192.168.1.0/24; };  # Limiter la récursion aux clients internes.
};

# Redémarrer le service bind9
sudo systemctl restart bind9

# Ajouter des Zones Directes et Inversées
sudo nano /etc/bind/named.conf.local

# Ajouter ceci (mettre ip du serveur secondaire)
zone "cfitech-it.com" {
    type master;
    file "/etc/bind/db.cfitech-it.com";
    allow-transfer { 192.168.1.3; };     # Autorise le transfert vers le serveur secondaire.
    also-notify { 192.168.1.3; };        # Notifie le serveur secondaire des mises à jour.
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.rev.cfitech-it.com";
};

# 