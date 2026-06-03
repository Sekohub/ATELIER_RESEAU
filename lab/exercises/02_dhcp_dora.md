# Exercice 2 — DHCP : observer DORA paquet par paquet

**Durée estimée :** 45 min
**Objectif :** capturer un échange DHCP complet (Discover / Offer / Request /
ACK), identifier les options portées par chaque message, et comprendre
pourquoi DHCP utilise un broadcast L2 alors qu'IP n'est pas encore
configuré.

## Manipulation

Côté `dhcp-server`, démarrez une capture filtrée sur les ports DHCP (67/68)&nbsp;:

```bash
docker exec -it lab_dhcp_server tcpdump -i eth0 -nn -e -v port 67 or port 68
```

> Note&nbsp;: `-e` affiche les adresses MAC, indispensables pour comprendre
> le broadcast L2.

Côté `client`, déclenchez une nouvelle demande de bail&nbsp;:

```bash
docker exec lab_client bash -c "dhclient -r eth0 2>/dev/null; dhclient -v eth0"
```

Observez les **4 paquets** DORA dans la capture, puis arrêtez tcpdump (Ctrl+c).

Affichez aussi les journaux applicatifs du serveur&nbsp;:

```bash
docker logs --tail 40 lab_dhcp_server
```

## À rendre — répondez directement dans ce fichier

### 1. Tableau DORA

Complétez en vous appuyant sur **votre propre capture**&nbsp;:

| Étape       | Émetteur (IP src) | Destinataire (IP dst) |               MAC src / dst               | Options DHCP notables |
| ----------- | ----------------- | --------------------- | ----------------------------------------- | --------------------- |
| 1. Discover | `0.0.0.0`         | `255.255.255.255`     | `6a:7e:b7:2d:0b:9e` > `ff:ff:ff:ff:ff:ff` | option 53 = …, option 55 = … |
| 2. Offer    | `172.20.1.2`      | `172.20.1.176`        | `d2:6f:84:6f:7f:1c` > `6a:7e:b7:2d:0b:9e` | … |
| 3. Request  | `0.0.0.0`         | `255.255.255.255`     | `6a:7e:b7:2d:0b:9e` > `ff:ff:ff:ff:ff:ff` | … |
| 4. ACK      | `172.20.1.2`      | `172.20.1.176`        | `d2:6f:84:6f:7f:1c` > `6a:7e:b7:2d:0b:9e` | … |

### 2. Configuration finale du client

```bash
docker exec lab_client ip -4 addr show eth0
docker exec lab_client ip route
docker exec lab_client cat /etc/resolv.conf   # peut être vide si non géré par dhclient
```

Notez **l'IP attribuée, le masque, la passerelle, les DNS, la durée de bail**.

Configuration finale client : 
IP : 172.20.1.176 /24
Masque : 255.255.255.0
Gateway : 172.20.1.254
DNS : 1.1.1.1 / 8.8.8.8
Durée de bail : 42191sec (~12h)

### 3. Questions de réflexion

**Question 1.** Pourquoi le client utilise-t-il **`0.0.0.0` comme IP
source** pour le Discover, alors que c'est une adresse non routable&nbsp;?
Que se passerait-il avec n'importe quelle autre adresse&nbsp;?

Le client envoie un Discover car il n'a pas encore d'adresse IP pour communiquer. Il emet donc une trame vide en 0.0.0.0 "Broadcast" (en UDP) pour contacter le serveur. 
Avec une autre addresse, la demande serait rejetée.

**Question 2.** Pourquoi le **Request** est-il **rediffusé en broadcast**
alors que le client connaît déjà l'IP du serveur après l'Offer&nbsp;?

Au moment du "Request", le client n'a pas encore appliquée l'adresse IP que le serveur lui a offerte. Il ne peut donc pas envoyé un paquet ciblé vers le serveur.
Le Broadcast reste donc son seul moyen de communiquer avec le serveur.

**Question 3.** À quoi sert le **transaction ID (xid)** présent dans les
4 paquets&nbsp;? Que se passerait-il s'il était omis dans un réseau avec
plusieurs serveurs DHCP&nbsp;?

Le xid corrèle les messages d'une même transaction DHCP, indispensable car le protocole est en broadcast/UDP sans connexion. 
Sans lui, dans un réseau multi-serveurs ou multi-clients, il serait impossible de savoir quelle réponse correspond à quelle requête, provoquant des attributions d'IP erronées.

**Question 4.** Que renvoie le serveur si vous demandez explicitement une
adresse hors du pool (essayez `dhclient -v -s 172.20.1.99 eth0`)&nbsp;?
Justifiez.

Un message "No DHCPOFFERS received" apparaît après quelques tentatives. Le serveur ignore la demande car il ne distribue que des IP qu'il gère.
Si ça n'était pas le cas, il y aurait des conflits d'adresses et un manque de cohérence dans le réseau. Après coup, un nouveau Discover est lancé et le REQUEST est rectifiée avec une IP valide (c'est le NAK).

**Question 5.** La directive `dhcp-authoritative` est active sur notre
serveur. Quel est son effet **comportemental** sur les NAK&nbsp;?

Le dhcp-authoritative fait passer le serveur d'un mode passif (ignore les demandes invalides) à un mode actif (répond par un NAK immédiat). Résultat : les clients mal configurés sont corrigés tout de suite et repartent sur un Discover propre, ce qui accélère et fiabilise l'attribution d'adresses sur un réseau dont ce serveur est l'autorité.

### 4. Renouvellement de bail (T1/T2)

Le bail est de 12&nbsp;h, T1 (renouvellement) à 6&nbsp;h, T2 (rebind) à 10&nbsp;h30.
En **2-3 phrases**, décrivez la différence entre un renouvellement T1 et
un rebind T2 (destinataire du paquet, comportement attendu).

Dans le cas d'un renouvellement T1 à 6h, le client recontacte directement le serveur DHCP d'origine en UDP vers 172.20.1.2 pour lui demander de renouveller son bail. 
Si le serveur est injoignable et n'a pas répondu à l'appelle T1, le rebind T2 consiste à faire la même demande en Broadcast pour solliciter n'importe quel serveur DHCP du réseau.
Enfin si les deux demandes T1 et T2 sont sans réponse, le client va repartir sur un Discover complet.
