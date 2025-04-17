# Configuration d'un Serveur DNS Primaire et Secondaire avec Bind9 sur Ubuntu

- Cette configuration met en place un serveur DNS primaire et secondaire avec zones directes et inversées. Elle inclut des bonnes pratiques pour garantir une configuration propre, sécurisée et fonctionnelle.

- Mise à jour du système

```sh
sudo apt update && sudo apt upgrade -y
```
 
- Installez Bind9 et ses utilitaires

```sh 
sudo apt install bind9 bind9utils bind9-doc -y
```

- Activez le service Bind9 au démarrage

```sh
sudo systemctl enable bind9
sudo systemctl start bind9
```

- Ouvrez le port DNS (53) dans le pare-feu

```sh
sudo ufw allow 53
sudo ufw reload
```

- Configuration réseau (IP statique)
  - Configuration du serveur primaire

```sh
sudo nano /etc/netplan/01-netcfg.yaml
```

- Ajoutez la configuration suivante

```sh
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.1.2/24                # Mettre IP du serveur primaire  
      routes:
        - to: default
          via: 192.168.1.1              # Mettre la passerelle
      nameservers:
        addresses: [192.168.1.2]        # Mettre IP du serveur primaire
```

- Appliquez la configuration

```sh
sudo netplan apply
```

#### Configuration du serveur secondaire

- Répétez les étapes ci-dessus pour le serveur secondaire en remplaçant l’adresse IP par celle du second serveur

- Configuration de Bind9 sur le Serveur Primaire

  - Modifiez le fichier pour configurer les redirecteurs DNS

```sh
sudo nano /etc/bind/named.conf.options
```

- Ajoutez ou modifiez les options suivantes

```sh
options {
    directory "/var/cache/bind";

    forwarders {
        8.8.8.8;
        8.8.4.4;
    };

    dnssec-validation auto;

    listen-on { any; };
    listen-on-v6 { any; };
    allow-query { any; };
};
```

- Redémarrez Bind9 pour appliquer les modifications

```sh
sudo systemctl restart bind9
```

- Ajoutez la configuration des zones directe et inversée

```sh
sudo nano /etc/bind/named.conf.local
```

- Contenu du fichier

```sh
zone "cfitech-it.com" {
    type master;
    file "/etc/bind/db.cfitech-it.com";
    allow-transfer { 192.168.1.3; }; # Autorise le transfert vers le serveur secondaire.
    also-notify { 192.168.1.3; };   # Notifie le serveur secondaire des mises à jour.
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.rev.cfitech-it.com";
};
```

- Créez un fichier pour la zone directe

```sh
sudo cp /etc/bind/db.local /etc/bind/db.cfitech-it.com
sudo nano /etc/bind/db.cfitech-it.com
```

- Contenu du fichier

```sh
$TTL    604800
@       IN      SOA     ns.cfitech-it.com. admin.cfitech-it.com. (
                        2025040200 ; Serial (à incrémenter lors des modifications)
                        10h        ; Refresh
                        15m        ; Retry
                        48h        ; Expire
                        604800 )   ; Negative Cache TTL

@       IN      NS      ns.cfitech-it.com.
@       IN      NS      ns2.cfitech-it.com.
@       IN      A       192.168.1.2                             # Mettre IP du serveur primaire

ns      IN      A       192.168.1.2   ; Serveur primaire.
ns2     IN      A       192.168.1.3   ; Serveur secondaire.
www     IN      CNAME   ns            ; Alias vers ns.
router  IN      A       192.168.1.1   ; Routeur local.
```

- Créez un fichier pour la zone inversée

```sh
sudo cp /etc/bind/db.local /etc/bind/db.rev.cfitech-it.com
sudo nano /etc/bind/db.rev.cfitech-it.com
```

- Contenu du fichier

```sh
$TTL    604800
@       IN      SOA     ns.cfitech-it.com. admin.cfitech-it.com. (
                        2025040200 ; Serial (à incrémenter)
                        10h        ; Refresh
                        15m        ; Retry
                        48h        ; Expire
                        604800 )   ; Negative Cache TTL

@       IN      NS      ns.cfitech-it.com.
@       IN      NS      ns2.cfitech-it.com.

2       IN PTR ns.cfitech-it.com.                   # Mettre dernier bit de IP du serveur primaire
3       IN PTR ns2.cfitech-it.com.                  # Mettre dernier bit de IP du serveur secondaire
```

- Avant de redémarrer Bind9, vérifiez les fichiers de configuration

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

#### Configuration du Serveur Secondaire

- Ajoutez une zone esclave sur le serveur secondaire

```sh
sudo nano /etc/bind/named.conf.local

zone "cfitech-it.com" {
    type slave;
    file "/var/cache/bind/db.cfitech-it.com";
    masters { 192.168.1.2; }; # Mettre IP du serveur primaire.
};

zone "1.168.192.in-addr.arpa" {
    type slave;
    file "/var/cache/bind/db.rev";
    masters { 192.168.1.2; };   # Mettre IP du serveur primaire
};
```

- Redémarrez Bind9 sur le serveur secondaire

```sh
sudo systemctl restart bind9 && sudo systemctl status bind9
```

- Depuis une machine cliente ou un autre serveur, testez la résolution DNS directe

```sh
nslookup www.cfitech-it.com 192.168.1.x   # Remplacez x par l'IP du serveur primaire ou secondaire.
```

- Testez la résolution DNS inversée (PTR)

```sh
nslookup 192.x.x.x   # Remplacez par une adresse IP configurée dans votre zone inversée.
```