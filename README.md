

# NSD + DNSSEC

### Conditions préalables

* Deux noms de domaine seront utilisés:

| Nom de domaine | Serveur DNS |
| --- | --- |
| awayfrom.network | ns1.awayfrom.network |
|  | ns2.awayfrom.network |
| pulvere.space | ns1.awayfrom.network |
|  | ns2.awayfrom.network |

* Les deux VPS suivantes utiliseront NSD:

| Hostname | Adresse IP |
| --- | --- |
| ns1.awayfrom.network | 1.1.1.1 |
| ns2.awayfrom.network | 2.2.2.2 |

 - 3.3.3.3 étant l'ip de votre serveur web

### Première étape - Installer et configurer NSD sur les deux serveurs


#### Serveur maître

En plus du package du serveur NSD, le serveur maître nécessite les packages suivants:

**ldnsutils** : Pour la génération de clé DNSSEC et la signature de zone.
**haveged** : L'installation de ce package accélère le processus de génération de clé.

Créez un user nommé:

```
useradd -r nsd
```

Mettez à jour les repos et installez NSD, ldnsutils et haveged.

```
apt-get update
apt-get install nsd ldnsutils haveged
```

Le transfert de zone DNS du serveur maître au serveur esclave est sécurisé par un secret partagé. On va donc générer une clé:

```
dd if=/dev/random count=1 bs=32 2> /dev/null | base64
```

Créez un répertoire séparé pour les fichiers de zone:

```
mkdir /etc/nsd/zones
```

Editer le fichier de configuration de NSD:

```
vim /etc/nsd/nsd.conf
```

La première section *"serveur"* spécifie les emplacements des fichiers de zone, des journaux et des fichiers PID (ID de processus):

```
server:
   username: nsd
   hide-version: yes
   zonesdir: "/etc/nsd/zones"
   logfile: "/var/log/nsd.log"
   pidfile: "/run/nsd/nsd.pid"
```
Dans la section *"key"*, nous définissons une clé nommée **mykey** et introduisons le secret généré précédemment.

```
key:
   name: "mykey"
   algorithm: hmac-sha256
   secret: ""
```

Chaque section *"zone"* contiendra le nom de domaine, le nom de fichier de la zone et les détails de son serveur esclave:

```
zone:
   name: awayfrom.network
   zonefile: awayfrom.network.zone
   notify: 2.2.2.2 mykey
   provide-xfr: 2.2.2.2 mykey
zone:
   name: pulvere.space
   zonefile: pulvere.space.zone
   notify: 2.2.2.2 mykey
   provide-xfr: 2.2.2.2 mykey
```

Les lignes *notify:* et *provide-xfr:* doivent avoir l'adresse IP du serveur esclave. Enregistrez le fichier et créez un fichier de zone pour **awayfrom.network**.

```
vim /etc/nsd/zones/awayfrom.network.zone
```

Nous allons ajouter les données suivantes dans le fichier de zone.

```
$ORIGIN awayfrom.network.
$TTL 1800
@       IN      SOA    ns1.awayfrom.network.    email.awayfrom.network. (
                       2020280201
                       3600
                       900
                       1209600
                       1800
                       )
@       IN      NS      ns1.awayfrom.network.
@       IN      NS      ns2.awayfrom.network.
ns1     IN      A       1.1.1.1
ns2     IN      A       2.2.2.2
@       IN      A       3.3.3.3
www     IN      CNAME   awayfrom.network.
```

Enregistrez ce fichier et créez un fichier de zone pour **pulvere.space**.

```
vim /etc/nsd/zones/pulvere.space.zone
```

Le deuxième fichier de zone:

```
$ORIGIN pulvere.space.
$TTL 1800
@       IN      SOA    ns1.awayfrom.network.    email.awayfrom.network. (
                       2020280201
                       3600
                       900
                       1209600
                       1800
                       )
@       IN      NS      ns1.awayfrom.network.
@       IN      NS      ns2.awayfrom.network.
@       IN      A       3.3.3.3
www     IN      CNAME   pulvere.space.
```

Enregistrez le fichier et vérifiez les erreurs de configuration à l'aide de la commande suivante:

```
nsd-checkconf /etc/nsd/nsd.conf
```

Une configuration valide ne doit rien sortir. Redémarrez le serveur NSD:

```
service nsd restart
```

Vérifiez si les enregistrements DNS sont en vigueur pour les domaines à l'aide de la commande suivante :

```
dig ANY awayfrom.network. @localhost +norec +short
```

Un exemple de sortie de cette commande:

```
ns1.awayfrom.network. email.awayfrom.network. 2020280201 3600 900 1209600 1800
ns1.awayfrom.network.
ns2.awayfrom.network.
3.3.3.3
```

Répétez la commande pour le deuxième domaine:

```
dig ANY pulvere.space. @localhost +norec +short
```

Nous avons correctement installé et configuré NSD sur le serveur maître et avons également créé deux zones.

### Serveur Esclave

Le serveur esclave n'a besoin que du package NSD, aucune génération de clé ni signature n'étant effectuée dessus.

Créez un user nommé:

```
useradd -r nsd
```

Mettez à jour les repos et installez NSD:

```
apt-get update
apt-get install nsd
```

Créez un répertoire pour les fichiers de zone:

```
mkdir /etc/nsd/zones
```

Editez le fichier de configuration NSD:

```
vim /etc/nsd/nsd.conf
```

Ajouter des directives de configuration:

```
server:
   username: nsd
   hide-version: yes
   zonesdir: "/etc/nsd/zones"
   logfile: "/var/log/nsd.log"
   pidfile: "/run/nsd/nsd.pid"

key:
   name: "mykey"
   algorithm: hmac-sha256
   secret: ""

zone:
   name: awayfrom.network
   zonefile: awayfrom.network.zone
   allow-notify: 1.1.1.1 mykey
   request-xfr: 1.1.1.1 mykey
zone:
   name: pulvere.space
   zonefile: pulvere.space.zone
   allow-notify: 1.1.1.1 mykey
   request-xfr: 1.1.1.1 mykey
```

La clé "**mykey**" doit être exactement le même que celle saisi dans le serveur maître. Utilisez l'adresse du serveur maître dans les lignes *allow-notify* et *request-xfr*.

Recherchez les erreurs de configuration:

```
nsd-checkconf /etc/nsd/nsd.conf
```

Redémarrez le service NSD:

```
service nsd restart
```

Forcer un transfert de zone pour les deux domaines avec la commande:

```
nsd-control force_transfer awayfrom.network
nsd-control force_transfer pulvere.space
```

Maintenant, vérifiez si ce serveur peut répondre aux requêtes du domaine **awayfrom.network**.

```
dig ANY . @localhost +norec +short
```

Si cela renvoie le même résultat que le maître, cette zone est configurée correctement. Répétez la commande pour le domaine * foorbar.org * pour vérifier si sa zone est correctement configurée.

Nous avons maintenant une paire de serveurs DNS NSD qui font autorité pour les domaines **awayfrom.network** et **pulvere.space**.

À ce stade, vous devriez pouvoir visiter vos domaines dans votre navigateur Web.

### Deuxième étape - générer les clés et signer la zone

Dans cette étape, nous allons générer une paire (privée et publique) de clés de signature de zone (ZSK) et de clés de signature de clé (KSK) pour chaque domaine. _Les commandes de la section doivent être exécutées sur le serveur maître._

On ce déplace dans le répertoire des zone de NSD:

```
cd /etc/nsd/zones
```

Générez la ZSK dans l'algorithme:

```
export ZSK=`ldns-keygen -a RSASHA1-NSEC3-SHA1 -b 1024 awayfrom.network`
```

Générez ensuite une KSK en ajoutant l'option *-k* à la même commande:

```
export KSK=`ldns-keygen -k -a RSASHA1-NSEC3-SHA1 -b 2048 awayfrom.network`
```

Ce répertoire aura maintenant les six fichiers supplémentaires suivants:

* 2 clés privées avec une extension *.private*.
* 2 clés publiques avec une extension *.key*.
* 1 enregistrements DS avec une extension *.ds*.

Dans la *Troisième étape*, nous allons générer des enregistrements DS d'un type différent *. Pour éviter toute confusion, supprimez le fichiers d'enregistrement DS.

```
rm $KSK.ds
```

Répétez les commandes pour le domaine *pulvere.space*:

```
export ZSK2=`ldns-keygen -a RSASHA1-NSEC3-SHA1 -b 1024 pulvere.space`
export KSK2=`ldns-keygen -k -a RSASHA1-NSEC3-SHA1 -b 2048 pulvere.space`
rm $KSK2.ds
```

La commande *ldns-signzone* est utilisée pour signer la zone DNS.

```
ldns-signzone -n -p -s $(head -n 1000 /dev/random | sha1sum | cut -b 1-16) awayfrom.network.zone $ZSK $KSK
```

Un nouveau fichier nommé **exemple.com.zone.signed** est créé.

Exécutez la commande pour le domaine **pulvere.space**:

```
ldns-signzone -n -p -s $(head -n 1000 /dev/random | sha1sum | cut -b 1-16) pulvere.space.zone $ZSK2 $KSK2
```

NSD doit être configuré pour utiliser les fichiers de zone. Editez le fichier de configuration:

```
vim /etc/nsd/nsd.conf
```

Modifiez l'option sous la section pour les deux domaines.

```
zone:
   name: awayfrom.network
   zonefile: awayfrom.network.zone.signed
   notify: 2.2.2.2 mykey
   provide-xfr: 2.2.2.2 mykey
zone:
   name: pulvere.space
   zonefile: pulvere.space.zone.signed
   notify: 2.2.2.2 mykey
   provide-xfr: 2.2.2.2 mykey
```

Pour appliquer les modifications et recharger le fichier de zone, exécutez les commandes suivantes:

```
nsd-control reconfig
nsd-control reload awayfrom.network
nsd-control reload pulvere.space
```

Recherchez les enregistrements DNSKEY en effectuant une requête DNS:

```
dig DNSKEY awayfrom.network. @localhost +multiline +norec
```

Cela devrait imprimer les clés publiques de ZSK et KSK comme suit:

```
; <<>> DiG 9.9.5-3-Ubuntu <<>> DNSKEY awayfrom.network. @localhost +norec +multiline
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 14231
;; flags: qr aa; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;awayfrom.network.                IN DNSKEY

;; ANSWER SECTION:
awayfrom.network.            300 IN DNSKEY 256 3 7 (
                                AwEAAZZ/AhrWgj9bbgUVVS+nPNYTPQtwo/5PNZmiI5h1
                                1lY3up1EAc2t7Az7pYCEbKY3glOHS2iV3abl0t+XmbNV
                                FUotKMORoat/gNt5Fwdi7WMc331MZfCVEKL60F1d8t6K
                                7/lZu6aQ9jUc0P4jw4bLvc5HGJNu5ccRWi1IZS2qvbGd
                                ) ; ZSK; alg = NSEC3RSASHA1 ; key id = 27245
awayfrom.network.            300 IN DNSKEY 257 3 7 (
                                AwEAAc5MCkEcD+R/rxOA2IDBhSb9peaBRLdnKEguECem
                                PIgOC263odaoZNKzcJHP8h4YdYZyOt+AlqwIb0cA9MxK
                                4PMmWsxgcW7h8x4CR2uE+gABBgVpXMaTCzLmJD0qczDv
                                IinMSnwKfvBrd6Io+2CPVJEFRofnyx0zWHXlMhDDLZQe
                                Iaa9jiUttZDgaNNcb6VOjSBt1NJ7pHI171L2dqAyqjlb
                                FfWVAywy6L6cKlRMIJuOm5uJ9OXpdEEqQ9Ek6ubU2ylW
                                z83UKVAfAbi+N1FD4dDLy8URS4USjQ0E1VZBr3dezWDf
                                7KVHOm/w3gjWviqiCnuVXdM7HE8EauqSX/Tm7/k=
                                ) ; KSK; alg = NSEC3RSASHA1 ; key id = 26241

;; Query time: 5 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Thu Sep 04 01:37:18 IST 2014
;; MSG SIZE  rcvd: 467
```

Répétez la commande pour le deuxième domaine et vérifiez la réponse:

```
dig DNSKEY pulvere.space. @localhost +multiline +norec
```

Le serveur maître fournit maintenant des réponses DNS *signées*.

#### Esclave

Cette zone doit maintenant être transférée sur le serveur esclave. Connectez-vous au serveur esclave et forcez le transfert des deux zones.

```
nsd-control force_transfer awayfrom.network
nsd-control force_transfer pulvere.space
```

Requête pour les enregistrements DNSKEY sur ce serveur:

```
dig DNSKEY . @localhost +multiline +norec
```

Cela devrait retourner le même DNSKEY que nous avons vu sur le serveur maître.

### Troisième étape - Générer des enregistrements DS

Dans cette étape, nous allons générer deux enregistrements DS que vous entrerez dans le panel du registraire de domaine. Les enregistrements DS auront les spécifications suivantes:

|  | Algorithme | Type de hashage |
| --- | --- | --- |
| DS record 1 | RSASHA1-NSEC3-SHA1 | SHA1 |
| DS record 2 | RSASHA1-NSEC3-SHA1 | SHA256 |

* Les commandes suivantes doivent être exécutées sur le serveur maître.

La commande génère des enregistrements DS à partir du fichier de zone signé. On ce rend dans le répertoire des fichiers de zone et exécutez les commandes:

```
cd /etc/nsd/zones
ldns-key2ds -n -1 .zone.signed && ldns-key2ds -n -2 awayfrom.network.zone.signed
```

Ceci retourne deux lignes de sortie:

```
awayfrom.network. 1800 IN DS 25050 7 1 db227c6dc0cf5d0a618f61714a4194ed72cf441a
awayfrom.network. 1800 IN DS 25050 7 2 0380981d351d4c7882eef78be98e5ebc2cf2fcb9a7ac45deca8a580bac7679f6
```

Le tableau suivant montre chaque champ de ces enregistrements DS:

|  | Key tag | Algorithme | Type de Hashage | Resultat |
| --- | --- | --- | --- | --- |
| Enregistrements DS #1 | 25050 | 7 | 1 | c1b9f7f1[...] |
| Enregistrements DS #2 | 25050 | 7 | 2 | 98216f4d[..] |

Générez des enregistrements DS pour le **pulvere.space**:

```
cd /etc/nsd/zones
ldns-key2ds -n -1 .zone.signed && ldns-key2ds -n -2 pulvere.space.zone.signed
```

Notez les quatre enregistrements DS (deux par domaine). Nous aurons besoin d'eux dans la prochaine étape.

### Quatrième étape - Configurer les enregistrements DS avec le registraire

Dans cette section, nous ajouterons les enregistrements DS dans le panel du registraire de domaine. *Dans mon cas OVH*

Connectez-vous à panel et choisissez votre nom de domaine.

On ce dirige vers la section GLUE du nom de domaine *"maitre"* pour ajouter nos serveur DNS : 

![enter image description here](https://files.legroupedamis.best/oPVeAH.png)

Puis on rajoute nos entrées DS respective pour nos 2 domaines : 

![enter image description here](https://files.legroupedamis.best/rG0HBO.png)

Après quelques minutes, recherchez les enregistrements DS.

```
dig DS awayfrom.network. +trace +short | egrep '^DS'
```

La sortie doit contenir les deux enregistrements DS.

```
DS 25050 7 1 db227c6dc0cf5d0a618f61714a4194ed72cf441a server x.x.x.x in 1 ms.
DS 25050 7 2 0380981d351d4c7882eef78be98e5ebc2cf2fcb9a7ac45deca8a580bac7679f6 from server x.x.x.x in 1 ms.
```

### Cinquième étape - Vérifier le fonctionnement de DNSSEC

DNSSEC peut être vérifié sur les sites suivants:

* [http://dnssec-debugger.verisignlabs.com/](https://dnssec-debugger.verisignlabs.com/)
* [http://dnsviz.net/](http://dnsviz.net/)
