# INSTALLATION ET CONFIGRATION DU SERVEUR Dns 

### INSTALLATION :

 #### Mettre en place le serveur DNS master
  #### Installer les paquets Bind
  La première étape consiste à réaliser l'installation de BIND sur le DNS maître : 192.168.0.172. Il faut donc installer l’ensemble des packages bind* sur le serveur maître : 
   * yum install –y bind*  

 #### Configuration de Bind
  Les paquets étant installés, nous pouvons configurer les options de BIND sur le DNS maître (192.168.0.172).

 * Le fichier /etc/named.conf contient principalement les options de notre serveur DNS maître. Il faut préciser quelle(s) adresse(s) va (ou vont) être autorisé et suivie(s). On doit alors ajouter l’adresse de notre serveur DNS primaire :

 * options {
        listen-on port 53 { 192.168.1.172; localhost; };
   On peut laisser les mentions du répertoire et des fichiers de cache et de statistiques (sauf si l’on souhaite changer leur emplacement) :

        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";  

 #### On peut ensuite préciser que l’on souhaite utiliser le mode récursif lorsque des requêtes sont émises et autoriser les réseaux ou zones sur lesquelles on peut envoyer ces requêtes en cas de réponse vide :

        recursion yes;
        allow-query     { 192.168.1.0/24; 127.0.0.1/8; localhost; };
#### Ensuite, si l’on envisage (c’est notre cas), de transférer les requêtes circulant vers un serveur secondaire, on peut le déclarer de la manière suivante :

        allow-transfer { 192.168.1.173; localhost; };
#### En dernier lieu, si l’on connaît d’autres autorités DNS, on peut également leur transférer les requêtes provenant des clients en déclarant la ligne suivante :

        forwarders { 12.27.2.102; 13.27.2.102; };
#### Le reste peut être conservé à l’identique du fichier d’origine. Bien évidemment, la dernière ligne n’est absolument pas nécessaire si l’on souhaite rester en circuit fermé. Mais, dans le cas où l’on souhaite aussi interroger les serveurs DNS 12.27.2.102 et 13.27.2.102, alors il faut ajouter cette option.  

### Création des fichiers de zones: 

  *  On va maintenant créer les deux fichiers de zones : l’un pour la résolution standard et l’autre pour la résolution inverse, le tout pour notre domaine "mydmn.org". Dans le répertoire /var/named on va devoir créer les deux fichiers suivants :

     * /var/named/named.mydmn.org
     * /var/named/named .1.168.192.in-addr.arpa 

 #### On devra donc déclarer les lignes suivantes, dans le fichier named.mydmn.org :

    $TTL 3600

@ IN SOA masterdns.mydmn.org. root.mydmn.org. (
        20170423         ; serial
        10800              ; refresh
        3600                ; retry
        604800             ; expire
        86400              ; minimum
)

@  IN          NS     masterdns.mydmn.org.
@  IN          NS     slavedns.mydmn.org.
@  IN          A           192.168.1.150
@  IN          A           192.168.1.151
@  IN          A           192.168.1.152
@  IN          A           192.168.1.153  

 #### Si l’on veut aller jusqu’au bout de la déclaration, on peut également ajouter les lignes déclaratives littérales (sans utiliser le caractère de substitution @, qui substitue le nom complet du domaine) :

masterdns                   IN          A      192.168.1.172
slavedns                    IN          A     192.168.1.173
client1                     IN          A     192.168.1.150
client2                     IN          A     192.168.1.151
client3                     IN          A     192.168.1.152
client4                     IN          A     192.168.1.153  

 #### Pour le fichier de zone reverse, on peut également prendre celui appelé named.loopback et le copier en named.1.168.192.in-addr-arpa. On peut ensuite y déclarer les lignes suivantes :

$TTL 86400

@       IN SOA  masterdns.mydmn.org. root.mydmn.org. (
                                20170423 ; serial
                                    3600 ; refresh
                                    1800 ; retry
                                  604800 ; expire
                                   86400 ; minimum
)
@               IN      NS      masterdns.mydmn.org.
@               IN      NS      slavedns.mydmn.org.
@               IN      PTR     mydmn.org.
masterdns       IN      A       192.168.1.172
slavedns        IN      A       192.168.1.173
client1         IN      A       192.168.1.150
client2         IN      A       192.168.1.151
client3         IN      A       192.168.1.152
client4         IN      A       192.168.1.153
172             IN      PTR     masterdns.mydmn.org.
173             IN      PTR     slavedns.mydmn.org.
150             IN      PTR     client1.mydmn.org.
151             IN      PTR     client2.mydmn.org.
152             IN      PTR     client3.mydmn.org.
153             IN      PTR     client4.mydmn.org.  


#### ATTENTION : normalement un utilisateur named et un groupe sont créés pour la gestion des zones et des enregistrements DNS. Mais, si les fichiers que l’on vient de créer utilisent le compte ou le groupe root, il faut changer cela. Dans l’idéal, les fichiers devraient avoir les propriétés suivantes :

-rw-r--r--   1 named named 2035 Mar 27 13:39 named.1.168.192.in-addr.arpa
-rw-r-----   1 named named 1892 Feb 18  2008 named.ca
-rw-r--r--   1 named named 4167 Mar 27 13:38 named.mydmn.org
-rw-r-----   1 named named  152 Dec 15  2009 named.empty
-rw-r-----   1 named named  152 Jun 21  2007 named.localhost
-rw-r-----   1 named named  168 Dec 15  2009 named.loopback  

 ### Déclaration des zones :
 
 #### Pour finir, on peut déclarer les zones à utiliser directement dans le fichier /etc/named.rfc1912.zones en l’éditant et lui ajoutant les lignes suivantes :

…
zone "mydmn.org" {
        type master;
        file "named.mydmn.org";
        notify yes;
        allow-transfer { 192.168.1.173; };
        allow-update { key rndc.key; };
};

zone "1.168.192.in-addr.arpa" {
        type master;
        file "named.1.168.192.in-addr.arpa";
        notify yes;
        allow-transfer { 192.168.1.173; };
        allow-update { key rndc.key; };
};  

### Vérification de la configuration

   Avant de lancer le service et de commencer à utiliser notre serveur DNS, il convient de s’assurer que l’on n’a rien oublié et que le paramétrage ne contient aucune erreur :

 * named-checkconf /etc/named.conf  

 #### Ensuite, on peut passer à la vérification de la configuration, zone par zone, en exécutant les commandes suivantes :

 * named-checkzone mydmn.org /var/named/named.mydmn.org
zone mydmn.org/IN: loaded serial 20170423
OK

 * named-checkzone 1.168.192.in-addr.arpa /var/named/1.168.192.in-addr.arpa
Zone 0.168.192.in-addr.arpa/IN: loaded serial 20170423
OK  

#### Lorsqu’il y a une erreur, généralement le texte en est suffisamment explicite pour corriger et relancer les commandes de vérifications, jusqu’à ce qu’il n’y ait plus aucun problème. On peut alors démarrer le service DNS et le rendre automatique lors des redémarrages du système :

* service named start
* chkconfig named on  

#### ATTENTION : si l’on a activé le service de pare-feu local, via les tables netfilter, il faut alors autoriser le flux TCP/53 et UDP/53 et redémarrer le service iptables:

  iptables -A INPUT -p tcp -m state --state NEW -m tcp --dport 53 -j ACCEPT
  iptables -A INPUT -p udp -m state --state NEW -m udp --dport 53 -j ACCEPT   

  #### Il existe les règles équivalentes pour SystemD. La seule différence c’est qu’il faut bien sûr intégrer ce genre de règles dans la zone public du pare-feu. En effet, SystemD fonctionne via des zones :

* drop
* block
* public
* external
* internal
* dmz
* work
* home
* trusted
Le plus simple étant d’intégrer ces règles de façon permanente via l’option --permanent :

* firewall-cmd --zone=public --permanent --add-service=dns
* firewall-cmd --reload  

### Configuration du serveur DNS esclave  

 #### On peut ensuite configurer le serveur DNS esclave. En règle générale, on peut reprendre le même paramétrage que pour le serveur maître à quelques exceptions près. En commençant par le fichier /etc/named.conf :

options {
        listen-on port 53 { 127.0.0.1; 192.168.1.173; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursion yes;
        allow-query     { 192.168.1.0/24;  localhost; };
        forwarders { 12.27.2.102; 13.27.2.102; };  

#### On peut ensuite configurer le serveur DNS esclave. En règle générale, on peut reprendre le même paramétrage que pour le serveur maître à quelques exceptions près. En commençant par le fichier /etc/named.conf :

options {
        listen-on port 53 { 127.0.0.1; 192.168.1.173; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursion yes;
        allow-query     { 192.168.1.0/24;  localhost; };
        forwarders { 12.27.2.102; 13.27.2.102; };  

### Configuration du serveur DNS esclave  

#### On peut ensuite configurer le serveur DNS esclave. En règle générale, on peut reprendre le même paramétrage que pour le serveur maître à quelques exceptions près. En commençant par le fichier /etc/named.conf :

options {
        listen-on port 53 { 127.0.0.1; 192.168.1.173; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursion yes;
        allow-query     { 192.168.1.0/24;  localhost; };
        forwarders { 12.27.2.102; 13.27.2.102; };  

### Pour le reste, on peut recopier les fichiers de configuration se trouvant dans le répertoire /var/named et démarrer le service DNS :

* service named start
* chkconfig named on
REMARQUE : en réalité, on n’a même pas besoin de créer les fichiers de zones, ceux-ci sont résolus par le service DNS du serveur maître. Mais, pour que ce mécanisme de synchronisation fonctionne, il est essentiel que le n° de série (serial) des fichiers de zones soient les mêmes et que dans le fichier /etc/named.rfc1912.zones on ait bien déclaré la ligne du cache  ci-dessous, pour chacune des zones à gérer :

        allow-update { key rndc.key; };


### Déployer les serveurs DNS sur les postes clients

Lorsque les deux serveurs sont opérationnels, il faut alors mettre à jour le fichier '/etc/resolv.conf' des clients, en renseignant les lignes suivantes:

search mydmn.org
nameserver 192.168.1.172
nameserver 192.168.1.173
Il faut également modifier le fichier /etc/sysconfig/network afin de renseigner le champ HOSTNAME du client avec son nom FQDN. Toutefois, si le fichier contient la ligne suivante :

HOSTNAME=localhost.localdomain
Les différents champs seront interprétés et remplacés par leur valeur contextuelle. Par exemple, sur la machine client1, la valeur interprétée avec la commande d’interrogation hostname, sera : client1.mydmn.org :

* hostname
client1.mydmn.org
Les principaux outils d’interrogation d’un serveur DNS sont nslookup (il s’agit d’un des plus anciens clients d’interrogation) et dig (outil beaucoup plus récent). On les détaillera dans le troisième module.

Exemple : interrogation avec nslookup :

* nslookup client1
Server:         192.168.1.172
Address:        192.168.1.172#53
Name:   client1.mydmn.org
Address: 192.168.1.150
Félicitations le service DNS est en place ! Dans le chapitre suivant, nous verrons comment sécuriser nos serveurs DNS, en plusieurs étapes.