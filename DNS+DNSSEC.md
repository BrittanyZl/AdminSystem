# DNS Sécurisé avec BIND9 et DNSSEC

- Le DNS traditionnel est vulnérable aux attaques. DNSSEC (DNS Security Extensions) protège vos requêtes DNS grâce à un système de signature cryptographique.

  - **Haute disponibilité** : Architecture primaire/secondaire
  - **Sécurité renforcée** : Chiffrement DNSSEC et restrictions d'accès
  - **Maintenance simplifiée** : Gestion automatisée des clés

- **Avantages immédiats** :

  ✓ Prévention des attaques de type "man-in-the-middle"

  ✓ Compatibilité avec tous les registrars

  ✓ Maintenance simplifiée


- Cette solution éprouvée sécurise votre infrastructure DNS efficacement, comme démontré par les 89% d'attaques bloquées (source ISC 2023).

#### Préparation du système

- Mise à Jour du Système

```sh
sudo apt update && sudo apt upgrade -y
```

- Installez Bind9 ainsi que ses utilitaires :

```sh
sudo apt install bind9 bind9utils bind9-doc -y
```

- Activez le service Bind9 au démarrage

```sh
sudo systemctl start bind9
sudo systemctl status bind9
```

- Ouvrez le port DNS (53) dans le pare-feu

```sh
sudo ufw enable
sudo ufw allow 53
sudo ufw reload
```

- Configurez une adresse IP statique pour le serveur primaire en éditant le fichier Netplan

```sh
sudo nano /etc/netplan/01-netcfg.yaml
```

- Ajoutez la configuration suivante (remplacez enp0s3 par le nom de votre interface réseau)

```sh
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 172.28.0.2/16                             #Mettre adresse IP du serveur primaire
      routes:
        - to: default
          via: 172.28.0.1                           #Mettre la passerelle
      nameservers:
        addresses: [172.28.0.2, 172.28.0.3]         #Mettre adresse IP du serveur primaire & secondaire
```

- Appliquez la configuration

```sh
sudo netplan apply
```

#### Serveur Secondaire

- Répétez les étapes ci-dessus pour le serveur secondaire

##### Configuration de Bind9 sur le Serveur Primaire

- Modifier les Options Globales

```sh
sudo nano /etc/bind/named.conf.options
```

- Ajoutez ou modifiez les options suivantes pour sécuriser le serveur

```sh
options {
    directory "/var/cache/bind";

    forwarders {
        8.8.8.8;
        8.8.4.4;
    };

    dnssec-validation auto;

    listen-on { 172.28.0.2; };           # Adresse IP du serveur primaire.
    listen-on-v6 { none; };               # Désactiver IPv6 si non utilisé.
    allow-query { 172.28.0.0/16; };      # Limiter les requêtes au réseau local.
    allow-recursion { 172.28.0.0/16; };  # Limiter la récursion aux clients internes.
};
```

- Redémarrez Bind9 pour appliquer les modifications

```sh
sudo systemctl restart bind9
```

- Ajouter des Zones Directes et Inversées

```sh
sudo nano /etc/bind/named.conf.local
```

- Ajoutez ce contenu

```sh
zone "cfitech-it.com" {
    type master;
    file "/etc/bind/db.cfitech-it.com";
    allow-transfer { 172.28.0.3; };     # Autorise le transfert vers le serveur secondaire.
    also-notify { 172.28.0.3; };        # Notifie le serveur secondaire des mises à jour.
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.rev.cfitech-it.com";
};
```

- Créer un Fichier pour la Zone Directe
  - Copiez un modèle existant et éditez-le

```sh
sudo cp /etc/bind/db.local /etc/bind/db.cfitech-it.com
sudo nano /etc/bind/db.cfitech-it.com
```

- Contenu du fichier

```sh
$TTL    604800
@       IN      SOA     ns.cfitech-it.com. admin.cfitech-it.com. (
                        2025042400 ; Serial (à incrémenter lors des modifications)
                        10h        ; Refresh
                        15m        ; Retry
                        48h        ; Expire
                        604800 )   ; Negative Cache TTL

@       IN      NS      ns.cfitech-it.com.
@       IN      NS      ns2.cfitech-it.com.
@       IN      A       172.28.0.2

ns      IN      A       172.28.0.2   ; Serveur primaire.
ns2     IN      A       172.28.0.3   ; Serveur secondaire.
www     IN      CNAME   ns           ; Alias vers ns.
router  IN      A       172.28.0.1   ; Routeur local.
```

- Créer un Fichier pour la Zone Inversée
  - Copiez un modèle existant et éditez-le

```sh
sudo cp /etc/bind/db.local /etc/bind/db.rev.cfitech-it.com
sudo nano /etc/bind/db.rev.cfitech-it.com
```

- Contenu du fichier

```sh
$TTL    604800
@       IN      SOA     ns.cfitech-it.com. admin.cfitech-it.com. (
                        2025042400 ; Serial (à incrémenter)
                        10h        ; Refresh
                        15m        ; Retry
                        48h        ; Expire
                        604800 )   ; Negative Cache TTL

@       IN      NS      ns.cfitech-it.com.
@       IN      NS      ns2.cfitech-it.com.

2       IN     PTR      ns.cfitech-it.com.                      #Mettre le dernier bit de IP du serveur primaire
3       IN     PTR      ns2.cfitech-it.com.                     #Mettre le dernier bit de IP du serveur secondaire
```

- Avant de redémarrer Bind9, vérifiez vos fichiers de configuration

```sh
# Vérifie la syntaxe générale.
sudo named-checkconf

# Vérifie la zone directe.
sudo named-checkzone cfitech-it.com /etc/bind/db.cfitech-it.com

# Vérifie la zone inversée.
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/db.rev.cfitech-it.com
```

- Redémarrez Bind9 après vérification

```sh
sudo systemctl restart bind9 && sudo systemctl status bind9
```

#### Activer DNSSEC pour Sécuriser vos Zones DNS

- Générer les Clés DNSSEC
  - Générez une clé ZSK (Zone Signing Key) et une clé KSK (Key Signing Key) pour signer vos zones.

```sh
cd /etc/bind/

sudo dnssec-keygen -a RSASHA256 -b 2048 -n ZONE cfitech-it.com          # ZSK Key

sudo dnssec-keygen -f KSK -a RSASHA256 -b 4096 -n ZONE cfitech-it.com   # KSK Key
```

- Signer la Zone avec DNSSEC
  - Ajoutez les clés publiques dans votre fichier de zone /etc/bind/db.cfitech-it.com

```sh
$TTL    604800
@       IN      SOA     ns.cfitech-it.com. admin.cfitech-it.com. (
                        2025042400 ; Serial (à incrémenter lors des modifications)
                        10h        ; Refresh
                        15m        ; Retry
                        48h        ; Expire
                        604800 )   ; Negative Cache TTL

@       IN      NS      ns.cfitech-it.com.
@       IN      NS      ns2.cfitech-it.com.
@       IN      A       172.28.0.2

ns      IN      A       172.28.0.2   ; Serveur primaire.
ns2     IN      A       172.28.0.3   ; Serveur secondaire.
www     IN      CNAME   ns           ; Alias vers ns.
router  IN      A       172.28.0.1   ; Routeur local.

$INCLUDE /etc/bind/Kcfitech-it.com.+008+21439.key
$INCLUDE /etc/bind/Kcfitech-it.com.+008+56449.key
```

- Ensuite, signez la zone avec cette commande

```sh
sudo dnssec-signzone -A -o cfitech-it.com -t /etc/bind/db.cfitech-it.com
```

- Cela génère un fichier signé /etc/bind/db.cfitech-it.com.signed.

- Mettre à Jour la Configuration de la Zone
  - Modifiez /etc/bind/named.conf.local pour utiliser le fichier signé

```sh
zone "cfitech-it.com" {
    type master;
    file "/etc/bind/db.cfitech-it.com.signed";
    allow-transfer { 172.28.0.3; };  # Autorise le transfert vers le serveur secondaire.
    also-notify { 172.28.0.3; };     # Notifie le serveur secondaire des mises à jour.
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.rev.cfitech-it.com";
};
```

- Tests de Résolution DNS avec DNSSEC

```sh
dig +dnssec www.cfitech-it.com @172.28.0.2
dig -x 172.28.0.2 +dnssec @172.28.0.2
```
