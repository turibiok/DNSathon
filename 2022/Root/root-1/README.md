# Serveur root 1

## Outils ##

`bind9, bind9utils, libbind9-161`<br>
`NB: Faire un sudo apt-cache search libbind9 pour avoir la dernière version de libbind9`

## Configurations ##

### 1 -Installation de bind9 ###

`sudo apt-get install bind9, bind9utils, libbind9-161`

### 2 - Configuration de la zone root ###

Ajouter dans le fichier `/etc/bind/named.config.local` la zone root:

```
zone "." {
    type master;
    file "/etc/bind/zones/root/db.root"
}
```

Créer dans le fichier `db.root` dans `/etc/bind/zones/root/`:
`sudo touch /etc/bind/zones/root/db.root`.

**NB: Créer les dossiers zones et root**

Éditer le fichier `/etc/bind/zones/root/db.root` en ajoutant les configs suivante:

```
;
; BIND data file for local loopback interface
;
$TTL 60
. IN SOA ns1.root. admin.root. (
       2022110904  ; Serial
    	60  ; Refresh
     	60  ; Retry
   	60  ; Expire
    	60 ) ; Negative Cache TTL
;
.    IN NS ns1.root.
.    IN NS ns2.root.
.    IN A 10.10.10.5 ; Adresse IP du serveur
.    IN A 10.10.10.10 ; Adresse IP du serveur root 2
ns1.root    IN A 10.10.10.5
ns2.root    IN A 10.10.10.10
```
Faire une sauvegarde du fichier `/etc/bind/named.conf.default-zones` et en modifier l'existant.

Changer la section

```
zone "." {
    type hint;
    file "/usr/share/dns/root.hints"
} 
```

contenu dans `/etc/bind/named.conf.default-zones` en

```
zone "." {
    type hint;
    file "/etc/bind/root.hints"
} 
```

Créer le fichier `/etc/bind/root.hints` et ajouter le contenu suivant:

```
.                    60      NS    ns1.root.
.                    60      NS    ns2.root.
ns1.root.       60      A     10.10.10.5
ns2.root.       60      A     10.10.10.10 ;Adresse IP du serveur root 2
```

### 4 - Configuration de la Zone Reverse
Ajouter dans le fichier `/etc/bind/named.config.local` la zone root:

```
zone "10.10.10.in-addr.arpa" {
	type master;
	file "/etc/bind/db.10.10.10";
};
```

Créer dans le fichier `db.10.10.10` dans `/etc/bind`:
`sudo touch /etc/bind/db.10.10.10`.

Éditer le fichier `/etc/bind/db.10.10.10` en ajoutant les configs suivante:

```
;
; BIND reverse data file for local loopback interface
;
$TTL    600
@       IN      SOA     ns1.root. contact.root. (
                              2         ; Serial
                        60         ; Refresh
                        60         ; Retry
                        60         ; Expire
                        60 )       ; Negative Cache TTL
;
@       IN      NS      ns1.root.
@       IN      NS      ns2.root.
5       IN      PTR     ns1.root.
10      IN      PTR     ns2.root.
```

### 5 - Configurer la synchronisation avec le serveur root 2 ###
Générer la clé de la tsig en tapant la commande: <br>
ddns-confgen -a hmac-sha256 -k . -z . 

Après l'éxécution de la commande ci-dessus, on a eu les fichiers: <br>
key "." {
        algorithm hmac-sha256;
        secret "CLE";
};

Ajouter ensuite dans le fichier `named.conf.local` la config suivante:

```
key "." {
	algorithm hmac-sha256;
	secret "CLE";
};
```

Modifier ensuite les zone **.** et **10.10.10.in-addr.arpa** de la manière suivante:

```
zone "." {
    type master;
    file  "/etc/bind/zones/root/db.root";
	allow-transfer { key "."; 10.10.10.10;};
	allow-update { key "." ; };
};
```

```
zone "10.10.10.in-addr.arpa" {
	type master;
	file "/etc/bind/db.10.10.10";
    allow-transfer { 10.10.10.10;};
};
```

**NB: l'IP 10.10.10.10 est l'adresse IPv4 du serveur root 2**

### 6 - Configurations des options et activation de dnssec ###

Créer le fichier `/etc/bind/named.conf.options` et ajouter les configurations suivantes:


```
options {
	directory "/var/cache/bind";
	dnssec-validation auto;
	dnssec-enable yes;

	listen-on-v6 { any; };
	listen-on {127.0.0.1; 10.10.10.5; };
	auth-nxdomain no ;
	version none;
	recursion no;
};

```

Créer le dossier `keys` dans `/etc/bind`.

Saisir ensuite les commandes suivantes pour gérer la clé `KSK` et la clé `ZSK`

```
cd /etc/bind/keys
dnssec-keygen -a RSASHA256 -b 2048 -n ZONE . # pour le ZSK
dnssec-keygen -f KSK -a RSASHA256 -b 4096 -n ZONE .
```

Modifier ensuite la zone **.** pour la signature DNSSEC

```
zone "." {
    type master;
    file "/etc/bind/zone/root/db.root";
	allow-transfer { key "."; 10.10.10.10;};
	allow-update { key "." ; };
	
	# look for dnssec keys here:
    key-directory "/etc/bind/keys";
    
    # publish and activate dnssec keys:
    auto-dnssec maintain;
    
    # use inline signing:
    inline-signing yes;
};
```

Réinitialiser les configurations du serveur en saisissant la commande:

`rndc reconfig`

Pour vérifier que le DNSSEC est bien activé exécuter la commande suivante:

`dig @10.10.10.5 SOA . +dnssec`

Si dans les records, vous avez le `RRSIG (RRset Signature)` c'est OK.  

### 7 - Tester les configurations ###

Pour tester les configurations et vérifier si tout vas bien exécuter les commandes suivantes:
`named-checkedzone` <br>
`named-checkedconf`

### 8 - Redémarrer bind ###

`sudo systemctl restart bind9`

### 9 - Configurer les résolveurs ###

Modifier le fichier `/etc/resolv.conf`

```
#nameserver 10.10.40.5
nameserver 10.10.40.10
```
et exécuter la commande `sudo systemctl restart bind9`.

### 10- Configurer les TLD ###

Modifier le fichier zone du seveur root pour déclarer les TLD .cotonou et .benin

```
;
; BIND data file for local loopback interface
;
$TTL 60
. IN SOA ns1.root. admin.root. (
        2022110904 ; Serial
        60 ; Refresh
        60 ; Retry
        60 ; Expire
        60 ) ; Negative Cache TTL
;
.          IN NS ns1.root.
.          IN NS ns2.root.
.          IN A 10.10.10.5 ; Adresse IP du serveur
.          IN A 10.10.10.10 ; Adresse IP du serveur root 2
ns1.root   IN A 10.10.10.5
ns2.root   IN A 10.10.10.10
;
cotonou.      IN NS  ns1.cotonou.
parakou.      IN NS  ns1.parakou.
ns1.cotonou.  IN A   10.10.20.5
ns1.parakou.  IN A   10.10.20.10 
```


**NB: A chaque modification d'un fichier de zone il faut incrémenter le Serial dans le SOA.**

