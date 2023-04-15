# Deroulement du projet

## Etape 1

- Routage static pour assurer la connectivité depuis/vers l’Internet. Cette tâche est déjà réalisée dans certains projets

for the provided network topology, most of the static routing configuration was done, altghou most of it was not correctly configured and the devices could not connect to the internet. to further debug and inspect the problem we followed this procedure

## Site 2

Pour le Site 2 on a suivuit les etapes suivantes:

- Verification des configuration de la table de routage:

```bash
Switch#sh ip route

Gateway of last resort is 200.168.2.2 to network 0.0.0.0

C    192.168.2.0/24 is directly connected, Vlan1
     200.168.2.0/29 is subnetted, 1 subnets
C       200.168.2.0 is directly connected, Vlan2
S*   0.0.0.0/0 [1/0] via 200.168.2.2
```

cette resultat declare une configuration d'un default gateway 200.168.2.2. c'est l'address du firewale pour le site 2.

- Verification de la configuration du routeur R1_internet

Le routeur R1_internet et le premier routeur connecter au firewale du site 2. donc il est conciderer comme un ISP pour le site deux. Le bon fonctionement de ce routeur est cle pour l'accecibilite au internet pour le site 2.

```bash
Router#sh ip route

Gateway of last resort is 20.20.20.2 to network 0.0.0.0

     18.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C       18.18.18.0/30 is directly connected, GigabitEthernet0/1
L       18.18.18.1/32 is directly connected, GigabitEthernet0/1
     20.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C       20.20.20.0/24 is directly connected, GigabitEthernet0/2
L       20.20.20.1/32 is directly connected, GigabitEthernet0/2
     21.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C       21.21.21.0/30 is directly connected, GigabitEthernet0/0
L       21.21.21.1/32 is directly connected, GigabitEthernet0/0
     200.168.1.0/29 is subnetted, 1 subnets
S       200.168.1.0/29 [1/0] via 21.21.21.2
     200.168.2.0/29 is subnetted, 1 subnets
S       200.168.2.0/29 [1/0] via 18.18.18.2
S*   0.0.0.0/0 [1/0] via 20.20.20.2
```

le point essentiel de cette configuration est la presence d'une route pour le reseaux 200.168.2.0/29, c'est le reseaux entre le  switch L3 du site 2 et le firewall du site 2. Le routeur n'as auccun connaissance du reseaux prive 192.168.2.0/24. On peux deduire qu'une configuration du NAT est naicessaire pour que le reseqxu 192.168.2.0/24 peux avoire access a internet.

3. Dans cette etape on peux revenir vers le switch du site 2 pour verifier la configuration du NAT

```bash
Router#sh run 

..
...
..

interface Vlan1
 ip address 192.168.2.254 255.255.255.0
 ip nat inside
!
interface Vlan2
 mac-address 00e0.f9d9.e601
 ip address 200.168.2.1 255.255.255.248
 ip access-group 110 in
 ip nat outside
!
ip nat pool SITE1 200.168.2.3 200.168.2.6 netmask 255.255.255.248
ip nat inside source list LAN pool SITE1
ip classless
ip route 0.0.0.0 0.0.0.0 200.168.2.2 
!
....

```

cette configuration declare la presence de deux interface Vlan le premier avec un address 192.168.2.254/24 et le deuxieme avec un eaddress 200.168.2.1/29. Une poul des address pour la translation des address nomme SITE1 est configurer avec des address et des mask correct. le premier vlan est declarer comme le inside de translation nat et le deuxieme est declarer comme le outside. Une translation NAT est configurer avec l'access list ***LAN*** et le pool des address SITE1. quellque chose cloch ici, une ACL avec le nom ***LAN*** ?.

l'etape suivante est de verifier la configuration des ACL dans ce switch. La meme commande precedente n'affiche aucune configuration d'un ACL. on peux declarer que la translation NAT est male configurer avec un ACL qui n'existe pas. donc on peux le reconfigurer de meme facon mais avec un ACL qui existe. pour les test initiaux on configure une ACL qui permit any pour assurer l'accebilite de l'internet pour les poste de ce reseaux.

```
Switch(config)# access-list 10 permit any

Switch(config)#no ip nat inside source list LAN pool SITE1

Switch(config)#ip nat inside source list 10 pool SITE1

```

Pour verife le fonctionement de notre configuration on va tester un ping du PC-PT (PC2) vert l'address 18.18.18.1

le ping ne fonctionne pas... plus d'investigation !!

au debut on doi verifier le fonctionement de la translation NAT au niveau du switch.

```bash
Switch#sh ip nat statistics 
Total translations: 1 (0 static, 1 dynamic, 1 extended)
Outside Interfaces: Vlan2
Inside Interfaces: Vlan1
Hits: 0  Misses: 3
Expired translations: 2
Dynamic mappings:
-- Inside Source
access-list 10 pool SITE1 refCount 1
 pool SITE1: netmask 255.255.255.248
       start 200.168.2.3 end 200.168.2.6
       type generic, total addresses 4 , allocated 1 (25%), misses 0

Switch#sh ip nat translations 
Pro  Inside global     Inside local       Outside local      Outside global
icmp 200.168.2.3:3     192.168.2.1:3      18.18.18.1:3       18.18.18.1:3

```

On a verifier que la translation NAT a ete fait pour l'address 192.168.2.1 vers 200.168.2.3. Donc le probleme n'est pas la translation NAT.

Dans un reseaux reel dans ce cas on doit utiliser `traceroute` pour suivre lacheminement des packet mais dans le cas de Cisco Packet tracer.

### SITE 1

Pour el site 1 c'etait generalement la meme procedure.

### DMZ

Pour la DMZ on a premierement tester la connectivite entre les hosts de la DMZ et le firewall du site 1.

```bash
C:\>ping 192.168.3.254

Pinging 192.168.3.254 with 32 bytes of data:

Reply from 192.168.3.254: bytes=32 time<1ms TTL=255
Reply from 192.168.3.254: bytes=32 time<1ms TTL=255
Reply from 192.168.3.254: bytes=32 time<1ms TTL=255
Reply from 192.168.3.254: bytes=32 time<1ms TTL=255

Ping statistics for 192.168.3.254:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms
```

On a assure la connectivite entre les hots de la DMZ et le port du firewall qui est connecter au meme reseaux que ces hots (Gig1/3). L'etape suivante est de tester la connectivite entre les hotes et routeur internet R2.

On a essaye de pinger l'interface du routeur avec l'addresse 19.19.19.1 mais ne pass passe, pour mieu debuguer le problem on cherche le point d'echec

```bash
C:\>tracert 19.19.19.1

Tracing route to 19.19.19.1 over a maximum of 30 hops: 

  1   0 ms      0 ms      0 ms      192.168.3.254
  2   *         *         *         Request timed out.
```

On remarque que le packet ne pass pas du firewall donc on a conclue que le problem est dans le niveau de securite du porte du firewall. Pour le moment on va changer le niveau de securite du port pour assurer la connectivite est eliminer les friction. Dans les prochaine etape du projet on vas restorer le niveau de securite est configurer des ACL bien filtrer le trafic.

- Inspection et modification du configuration du firewall site 1:

     la commade show running config affiche cette configuration pour le port Gig1/3

     ```bash
     interface GigabitEthernet1/3
     nameif DMZ
     security-level 50
     ip address 192.168.3.254 255.255.255.0
     ```

     On vas renitialiser le niveau de securite a 0

     ```bash
     ciscoasa#conf t
     ciscoasa(config)#int g
     ciscoasa(config)#int gigabitEthernet 1/3
     ciscoasa(config-if)#se
     ciscoasa(config-if)#security-level 0
     ```

     Apres cette modification on a assurer la connectivite entre les hotes de la DMZ et l'internet.

## ETAPE 3

Etablir un tunnel IP sec entre les deux firewall. Ce firewall cert a lier les deux reseaux prive a travert un tunnel securise. Les entite entre ces deux reseaux seront capable de communiquer entre eux avec leur address prive a travert le tunnel sans besoin de configurer le routage entre c'est deux reseaux prive dans les routeur de l'internet, qui sera pas possible du debut. Le defit qu'on a face est pour que les deux sous reseaux peux connecter a l'intrnet il etait neccessaire de configurer une translation NAT, cette configuration ete etablis au niveau du switch L3 qui est situe avant le firewall dans le deux site. donc on est besoin de reconfigurer la translation NAT pour faire la translation pour tous les packet sauf les packets qui sont on la source et la destination dans les subnet des deux sites.

Pour mieux explique la situation on voir la configuration d'un des switch, dans cette exemple on s'interes a la configuration du switch 2 mais ca change pas du grand chause entre les deux.

```bash
access-list 10 permit any
ip nat pool SITE1 200.168.2.3 200.168.2.6 netmask 255.255.255.248
ip nat inside source list 10 pool SITE1
```

Dans cette configuration on peux voire que la translation NAT est applique sur l'ACL 10 qui autorise tout. on doit la changer pour autoriser touts sauf les packet qui on l'address source dans `192.168.2.0/24` et l'address destination dans `192.168.1.0/24`. Pour cela on doit cree une nouvelle ACL qui `deny` les packet IP de source `192.168.2.0/24` et destination `192.168.1.0/24` et autorise tous le rest. C'est important de note que l'ordre de ces deux regle est important.

```bash
access-list 100 deny ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255
access-list 100 permit ip any any
```

L'etape suivante est de suprimer l'ancienne configuration du NAT est applique une nouvelle configuration avec la nouvelle ACL

```bash
no ip nat inside source list 10 pool SITE2
ip nat inside source list 100 pool SITE2
```

On fait de la meme facont dans le commutateur du SITE1

```bash
access-list 100 deny ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255
access-list 100 permit ip any any

no ip nat inside source list 10 pool SITE2
ip nat inside source list 100 pool SITE2

```

Maintenant qu'on assurer la bon configuration de la translation NAT. On peux configurer le tunnel ipsec entre les deux ASA.

On gros la configuration du vpn est constitue de deux phase. La premiere pahse est d'etablir un tunnel securise et authentifie pour echanger les message de negociation du ISAKMP. Pour configurer cette phase on doit cree une policy pour IKEV1 et definir l'algorithm du chifrememnt et le group de `Diffie-Hellman`, l'algorithm de hashage et la durre de vie de l'association. Cette configuration `doit` obligatoirement etre identic entre les deux ASA.

Apres on doit configurer le tunnel-group de type ipsec Lan to Lan et cree une cle pre-partager qui sera utiliser pour l'authentification entre les firewall. On note que le nom du tunnel doit etre identique au nom du peer dans les etapes suivante.

crypto ipsec ikev1 transform-set ESP-AES-SHA esp-aes esp-sha-hmac

La deuxieme phase est la phase de ipsec, cette phase sera securise par la SA de a premiere phase (IKE).  
Pour configurer cette phase on doit cree un `ipsec ike transform set` et definit l'algorithm d'authentification et de chifrement.

Apres on definit un crypto map pour la dexieme phase. on doit configurer le peer ( doit etre le meme que le tunnel-group ), applique le transform set et finalement applique la map sur l'interface sotant.

Voici la configuration du ipsec sur un des deux ASA

```bash
access-list SITE2_TO_SITE1 extended permit ip 192.168.2.0 255.255.255.0 192.168.1.0 255.255.255.0
crypto ipsec ikev1 transform-set ESP-AES-SHA esp-aes esp-sha-hmac
!
crypto map outside_map 10 match address SITE2_TO_SITE1
crypto map outside_map 10 set peer 19.19.19.2 
crypto map outside_map 10 set ikev1 transform-set ESP-AES-SHA 
crypto map outside_map interface outside
crypto ikev1 enable outside
crypto ikev1 policy 5
 encr aes
 authentication pre-share
 group 5
 lifetime 3600
!
tunnel-group 19.19.19.2 type ipsec-l2l
tunnel-group 19.19.19.2 ipsec-attributes
 ikev1 pre-shared-key vpnkey
!
```

Dans la configuration presedente on a: 
     - un ACL qui autorise les packet avec la destination 192.168.1.0/24 et la source 192.168.1.0/24
     - un policy de ikev avec:
          - authentification avec cle pre partager
          - un group Diffie-Hellman 5
          - algorithm de chifrement aes
          - une dure de vie de 3600 seconds
     - On a definit un ipsec transform set avec l'algorithme d'authentification SHA-1 HMAC et l'algorith de chifrement aes.
     - un cle ipsec pre partager vpnkey
     - un crypto map avec:
          - le peer 19.19.19.2
          - le transform set declarer precedament 
          - le acl declarer precedament

et de la meme facon on a cette configuration dans l'autre ASA

```bash
access-list SITE1_TO_SITE_2 extended permit ip 192.168.1.0 255.255.255.0 192.168.2.0 255.255.255.0
crypto ipsec ikev1 transform-set ESP-AES-SHA esp-aes esp-sha-hmac
!
crypto map outside_map 10 match address SITE1_TO_SITE_2
crypto map outside_map 10 set peer 18.18.18.2 
crypto map outside_map 10 set ikev1 transform-set ESP-AES-SHA 
crypto map outside_map interface outside
crypto ikev1 enable outside
crypto ikev1 policy 5
 encr aes
 authentication pre-share
 group 5
 lifetime 3600
!
tunnel-group 18.18.18.2 type ipsec-l2l
tunnel-group 18.18.18.2 ipsec-attributes
 ikev1 pre-shared-key vpnkey
!
```
