# DNS (Domain Name Service)
DNS signifie Domain Name System (ou système de noms de domaine en français).
C’est comme l’annuaire téléphonique d’Internet.

Il sert à traduire les noms de domaine faciles à retenir (comme google.com) en adresses IP que les ordinateurs utilisent pour se connecter entre eux (comme 142.250.190.78).

### Configuration d'un Serveur DNS Primaire et Secondaire avec Bind9 sur Serveru Ubuntu 22.04

- Cette configuration met en place un serveur DNS primaire et secondaire avec zones directes et inversées. Elle inclut des bonnes pratiques pour garantir une configuration propre, sécurisée et fonctionnelle.

#### Serveur Primaire 

- Mise à jour du système

```sh
sudo apt update && sudo apt upgrade -y
```
- Ajout des guest agent

```sh
sudo apt-get install qemu-guest-agent
sudo systemctl start qemu-guest-agent
sudo systemctl enable qemu-guest-agent
```

- Autorisation de l'accès à distance

```sh
sudo ufw enable
sudo ufw allow ssh
sudo ufw reload
```

- Installation de Bind9

```sh
sudo apt install bind9 bind9utils bind9-doc -y
```

- Activation du service

```sh
sudo systemctl enable bind9
sudo systemctl start bind9
```

- Configuration du pare-feu

```sh
sudo ufw allow 53
sudo ufw reload
```

- Configuration réseau (IP statique)
  - Configuration du serveur primaire

```sh
sudo nano /etc/netplan/01-netcfg.yaml
```

- Configuration IP statique

```sh
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [192.168.1.2/24]                   # Mettre adresse IP du serveur primaire
      routes:
        - to: default
          via: 192.168.1.1                          # Mettre adresse IP de la passerelle par défaut
      nameservers:
        addresses: [192.168.1.2, 192.168.1.3]       # Mettre adresse IP du serveur primaire & secondaire
```

- Appliquez la configuration

```sh
sudo netplan apply
```

- Configuration des options Bind9

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
    allow-transfer { 192.168.1.3; };    # Mettre adresse IP du serveur secondaire
    recursion yes;
};
```

- Redémarrez Bind9 pour appliquer les modifications

```sh
sudo systemctl restart bind9
```

- Configuration des zones

```sh
sudo nano /etc/bind/named.conf.local
```

- Contenu du fichier

```sh
zone "cfitech-it.com" {
    type master;
    file "/etc/bind/db.cfitech-it.com";
    allow-transfer { 192.168.1.3; };  # Autorise le transfert vers le serveur secondaire.
    also-notify { 192.168.1.3; };     # Notifie le serveur secondaire des mises à jour.
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.rev.cfitech-it.com";
    allow-transfer { 192.168.1.3; };    # Mettre adresse IP du serveur secondaire
};
```

- Créez un fichier pour la zone directe

```sh
sudo cp /etc/bind/db.local /etc/bind/db.cfitech-it.com
```

```sh
sudo nano /etc/bind/db.cfitech-it.com
```

- Contenu du fichier

```sh
$TTL    604800
@       IN      SOA     ns.cfitech-it.com. admin.cfitech-it.com. (
                        2025040201 ; Serial       #YYYYMMDDnn (nn=à incrémenter lors des modifications)
                        10h        ; Refresh
                        15m        ; Retry
                        48h        ; Expire
                        604800 )   ; Negative Cache TTL

; Serveurs DNS
@       IN      NS      ns.cfitech-it.com.
@       IN      NS      ns2.cfitech-it.com.

; Enregistrements A
@       IN      A       192.168.1.2  ; Serveur primaire.
ns      IN      A       192.168.1.2
ns2     IN      A       192.168.1.3  ; Serveur secondaire.
www     IN      CNAME   ns           ; Alias vers ns.
router  IN      A       192.168.1.1  ; Routeur local.
```

- Créez un fichier pour la zone inversée

```sh
sudo cp /etc/bind/db.local /etc/bind/db.rev.cfitech-it.com
```

- Zone inverse

```sh
sudo nano /etc/bind/db.rev.cfitech-it.com
```

```sh
$TTL    604800
@       IN      SOA     ns.cfitech-it.com. admin.cfitech-it.com. (
                        2025040201 ; Serial         #YYYYMMDDnn (nn=à incrémenter lors des modifications)
                        10h        ; Refresh
                        15m        ; Retry
                        48h        ; Expire
                        604800 )   ; Negative Cache TTL

@       IN      NS      ns.cfitech-it.com.
@       IN      NS      ns2.cfitech-it.com.

2       IN      PTR     ns.cfitech-it.com.          #mettre le dernier bit de IP du serveur primaire
3       IN      PTR     ns2.cfitech-it.com.         #mettre le dernier bit de IP du serveur secondaire
```

- Avant de redémarrer Bind9, vérifiez les fichiers de configuration

- Vérifie la syntaxe générale.

```sh
sudo named-checkconf
```

- Vérifie la zone directe.

```sh
sudo named-checkzone cfitech-it.com /etc/bind/db.cfitech-it.com
```

- Vérifie la zone inversée.

```sh
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/db.rev.cfitech-it.com     #Attention changement de IP 
```

- Redémarrez Bind9 après vérification

```sh
sudo systemctl restart bind9 && sudo systemctl status bind9
```
- Faire test de la résolution zone directe
```sh
nslookup <NomDomain>    
```    
si affiche ne trouve pas le serveur, faire:
    - aller dans 
    ```sh
    sudo nano /etc/resolv.conf
    ```
    - changer nameservers par celui de notre serveur primaire

- Redémarrer (si modification du fichier resolv.conf)

```sh 
sudo systemctl restart bind9
```


#### Configuration du Serveur Secondaire

- Mise à jour du système

```sh
sudo apt update && sudo apt upgrade -y
```

- Ajout des guest agent

```sh
sudo apt-get install qemu-guest-agent
sudo systemctl start qemu-guest-agent
sudo systemctl enable qemu-guest-agent
```

- Autorisation de l'accès à distance

```sh
sudo ufw enable
sudo ufw allow ssh
sudo ufw reload
```

- Installation de Bind9

```sh
sudo apt install bind9 bind9utils bind9-doc -y
```

- Activation du service

```sh
sudo systemctl enable bind9
sudo systemctl start bind9
```

- Configuration du pare-feu

```sh
sudo ufw allow 53
sudo ufw reload
```

- Configuration IP statique

```sh
  sudo nano /etc/netplan/01-netcfg.yaml
```

```sh
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [192.168.1.3/24]               #Mettre adresse IP du serveur secondaire
      routes:
        - to: default
          via: 192.168.1.1                      #Mettre adresse IP da la passerelle par défaut
      nameservers:
        addresses: [192.168.1.2, 192.168.1.3]   #Mettre adresse IP du serveur primaire & secondaire
```

```sh
sudo netplan apply
```

- Configuration des options Bind9

```sh
sudo nano /etc/bind/named.conf.options
```

```sh
options {
    directory "/var/cache/bind";
    dnssec-validation auto;
    listen-on { any; };
    listen-on-v6 { any; };
    allow-query { any; };
    recursion yes;
};
```

- Configuration des zones esclaves

```sh
sudo nano /etc/bind/named.conf.local
```

```sh
zone "cfitech-it.com" {
    type slave;
    file "/var/cache/bind/db.cfitech-it.com";
    masters { 192.168.1.2; };  # Adresse IP du serveur primaire.
};

zone "1.168.192.in-addr.arpa" {
    type slave;
    file "/var/cache/bind/db.rev.cfitech-it.com";
    masters { 192.168.1.2; };  # Adresse IP du serveur primaire.
};
```

- Vérification

```sh
sudo named-checkconf
```

- Redémarrage

```sh
sudo systemctl restart bind9
sudo systemctl status bind9
```

- Vérification du transfert de zone

```sh
sudo ls -l /var/cache/bind/
```

- Faire test de la résolution zone directe
```sh
nslookup <NomDomain>    
```    

- Changer serveur utilisé par défaut si le test ne reconnais pas le serveur secondaire

```sh
sudo nano /etc/resolv.conf
```
  //Mettre IP du serveur secondaire

- Redémarrer (si modification du fichier resolv.conf)

```sh 
sudo systemctl restart bind9
```

- Tests depuis un client

```sh
# Test de résolution directe
nslookup ns.cfitech-it.com 192.168.1.2
nslookup www.cfitech-it.com 192.168.1.3

# Test de résolution inverse
nslookup 192.168.1.2 192.168.1.2
nslookup 192.168.1.3 192.168.1.3

# Test de transfert de zone
dig @192.168.1.2 cfitech-it.com AXFR
dig @192.168.1.3 cfitech-it.com AXFR
```