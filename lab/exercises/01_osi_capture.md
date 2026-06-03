# Exercice 1 — OSI à travers une vraie capture de paquets

**Durée estimée :** 45 min
**Objectif :** relier chaque couche du modèle OSI à un champ concret observable
dans une capture réseau effectuée sur le lab.

## Préparation

Configurez le client (s'il ne l'a pas déjà été par DHCP — voir exercice 2)&nbsp;:

```bash
docker exec lab_client bash -c "ip route del default 2>/dev/null; ip route add default via 172.20.1.254"
```

Vérifiez l'accès au site « public »&nbsp;:

```bash
docker exec lab_client curl -s http://172.20.0.10/whoami
```

## Manipulation

**Étape 1 — supprimez une éventuelle capture précédente** (sinon échec en
*Permission denied* car tcpdump abandonne ses privilèges vers l'utilisateur
`tcpdump` après ouverture du fichier) :

```bash
docker exec lab_client rm -f /tmp/http.pcap
```

**Étape 2 — lancez la capture côté client.** Les flags importants&nbsp;:
`-U` (écriture non bufferisée, indispensable si la capture est interrompue),
`-c 30` (s'arrête automatiquement après 30 paquets, ≈ 2 à 3 requêtes HTTP) :

```bash
docker exec lab_client tcpdump -i eth0 -U -w /tmp/http.pcap -nn -c 30 host 172.20.0.10
```

> ⚠️ Cette commande **bloque** le terminal tant que les 30 paquets ne sont
> pas capturés. Ouvrez un **second terminal** pour l'étape 3.

**Étape 3 — dans un second terminal, déclenchez du trafic** *pendant* que
tcpdump tourne&nbsp;:

```bash
docker exec lab_client curl -v http://172.20.0.10/
docker exec lab_client curl -v http://172.20.0.10/whoami
docker exec lab_client curl -s http://172.20.0.10/        # complément pour atteindre 30 paquets
```

tcpdump s'arrête seul dès le 30e paquet et affiche un résumé du type
`30 packets captured / 0 packets dropped by kernel`.

**Étape 4 — vérifiez que la capture est exploitable** :

```bash
docker exec lab_client capinfos /tmp/http.pcap
```

`Number of packets` doit être > 0. Si la capture est vide, voir la section
*Pièges fréquents* en bas de cet énoncé.

**Étape 5 — analysez la capture**. Trois vues utiles&nbsp;:

```bash
# Vue compacte : une ligne par paquet (utile pour repérer les n° de frames)
docker exec lab_client tshark -r /tmp/http.pcap

# Vue détaillée d'un paquet précis (ex. la requête GET = frame n°4)
docker exec lab_client tshark -r /tmp/http.pcap -V -Y 'frame.number == 4'

# Vue détaillée complète (pager : utilisez 'q' pour quitter)
docker exec -it lab_client sh -c "tshark -r /tmp/http.pcap -V | less"
```

> `-V` produit la décomposition complète couche par couche
> (Frame → Ethernet → IP → TCP → HTTP). Le filtre `-Y` cible un paquet
> par son numéro pour éviter d'avoir à scroller dans toute la trace.

## Visualisation assistée : `osi_inspect.py`

La sortie brute de `tshark -V` est dense (plusieurs centaines de lignes par
paquet). Un script Python est fourni dans ce dossier pour vous présenter,
pour chaque trame, un **tableau structuré par couche OSI** avec en plus
une **colonne d'explication pédagogique** pour chaque champ.

### Lister les trames

Depuis la racine du dépôt (l'hôte, pas l'intérieur du conteneur) :

```bash
./lab/exercises/osi_inspect.py
```

Vous obtenez la même vue compacte que `tshark` mais sans avoir à taper la
commande complète. Repérez la trame qui vous intéresse — typiquement la
trame portant `GET /` (ligne marquée `HTTP 141 GET / HTTP/1.1`).

### Détailler une trame

```bash
./lab/exercises/osi_inspect.py 4         # trame n°4 — la requête HTTP
./lab/exercises/osi_inspect.py 1         # trame n°1 — le SYN du handshake
./lab/exercises/osi_inspect.py 8         # trame n°8 — la réponse 200 OK
```

Le script affiche, pour chaque couche OSI **présente** dans la trame, les
champs clés avec **3 informations** :

| Colonne       | Contenu                                       |
| ------------- | --------------------------------------------- |
| `Champ`       | Nom du champ tel qu'extrait par tshark        |
| `Valeur`      | Valeur réelle observée dans **votre** capture |
| `Explication` | À quoi sert ce champ, comment l'interpréter   |

C'est exactement la matière dont vous avez besoin pour remplir le tableau
*« Couche / Élément observé / Valeur exemple »* demandé dans la section
suivante.

### Réutilisation pour les exercices suivants

Le script est générique. Pour disséquer une capture DHCP (exercice 2) ou
NAT (exercice 3), pointez-le vers le bon conteneur et le bon fichier :

```bash
./lab/exercises/osi_inspect.py 1 --pcap /tmp/dhcp.pcap --container lab_dhcp_server
./lab/exercises/osi_inspect.py 3 --pcap /tmp/nat.pcap  --container lab_nat_router
```

### Travail demandé avec ce script

1. Lancez `./lab/exercises/osi_inspect.py` pour obtenir la liste des trames.
2. Identifiez **une trame contenant du HTTP** (typiquement la requête `GET /`)
   et **une trame de contrôle TCP** (SYN, ACK seul, ou FIN).
3. Lancez le script avec le n° de chaque trame et **copiez la sortie**
   dans le README de votre fork (bloc de code).
4. Pour chacune des deux trames, **comptez et nommez** les couches OSI
   visibles (utilisez la ligne `Pile présente : …` en en-tête). Expliquez
   en 1 phrase pourquoi la couche 7 est absente sur la trame de contrôle TCP.

I. Trame HTTP avec une liste des couches visibles dans la pile : 4 0.000260 172.20.1.50 → 172.20.0.10 HTTP 141 GET / HTTP/1.1 Couches : L2 - Liaison Ethernet / L3 - Réseau IPv4 / L4 - Transport TCP / L7 Application HTTP

II. Trame de contrôle TCP (ACK) : 3 0.000074 172.20.1.50 → 172.20.0.10 TCP 66 58282 → 80 [ACK] Seq=1 Ack=1 Win=64256 Len=0 TSval=4080508213 TSecr=1517447008 Couches : L2 - Liaison Ethernet / L3 - Réseau IPv4 / L4 - Transport TCP

Pas de couche applicative sur la trame n°3 car elle ne sert qu'à établir, maintenir ou fermer la connexion établie au niveau "Transport".



## À rendre — répondez directement dans ce fichier

Pour **chaque couche OSI**, donnez **un exemple concret extrait de votre
capture** (champ, valeur observée). Justifiez en 1-2 phrases.

| Couche OSI         | Élément observé dans la capture | Valeur exemple |
| ------------------ | ------------------------------- | -------------- |
| 7 — Application    | _ex. méthode HTTP_              | `GET /whoami HTTP/1.1` |
| 6 — Présentation   | _ex. encodage / Content-Type_   | `text/html - text/plain` |
| 5 — Session        | _ex. Keep-Alive, cookies_       | `Connection: close`|
| 4 — Transport      | _ex. port TCP, flags_           | `Port dest = 80 / Flags = ..AP.. / Port source : 58286`|
| 3 — Réseau         | _ex. IP source / destination_   | `IP dest = 172.20.1.50`|
| 2 — Liaison        | _ex. adresses MAC_              | `MAC dest = ba:5b:85:8b:ca:d8`|
| 1 — Physique       | _non visible — pourquoi&nbsp;?_ | `Pas de couche Physique car les trames sont capturées à partir de la couche L2. Elles sont donc déjà décodées en bits par la carte réseau.`|

## Questions de réflexion

**Question 1.** Pourquoi l'**adresse MAC source** observée n'est-elle
**pas** celle du serveur `internet` mais celle du `nat-router`&nbsp;? Que
vous apprend cette observation sur la portée de chaque couche&nbsp;?

Lorsque le client va envoyer un paquet vers la destination : 172.20.0.10 ; il va consulter sa table de routage et constater que la destination est sur un autre réseau. Ainsi, la trame va passer par la passerelle par défaut (172.20.0.254), qui correspond au routeur NAT et la trame sera encapsulée avec comme "MAC destination", celle du routeur.

**Question 2.** Vous capturez sur `eth0` du client (côté LAN). Dans votre
trace, l'**IP source** sortante est `172.20.1.50`. Pourtant, `curl /whoami`
rapporte que le serveur perçoit `172.20.0.254`. Expliquez cette différence
et indiquez **où** il faudrait capturer pour voir l'IP réécrite.
*Astuce&nbsp;:* `docker exec lab_nat_router tcpdump -i any -nn -c 10 host 172.20.0.10`.

La capture de trame se fait sur l'interface "eth0" du client. Le paquet n'est donc pas encore passé par le routeur, ce qui explique pourquoi l'adresse sortante est le "172.20.1.50" mais le serveur web perçoit "172.20.0.254". L'un est capturée avant le passage par le WAN du routeur l'autre après. Pour constater ce changement, la capture devrait être faite sur l'interface WAN du routeur.

**Question 3.** Lancez `curl -v https://...` vers un site HTTPS public
(depuis l'hôte, pas le lab). Quelle couche change visiblement par
rapport au HTTP du lab&nbsp;? Quelles couches **disparaissent** de votre
visibilité&nbsp;?

La couche 7 "Application" était clairement visible sur HTTP. Elle n'est plus du tout visible lorsque l'on fait un -curl vers un site "https://" puisqu'elle est chiffrée par TLS. La couche 6 "Présentation" quant à elle apparaît puisque le "handshake" du TLS apparaît dans les trames.

**Question 4.** La couche 5 (Session) est très peu visible dans une
capture HTTP/1.1. Donnez **deux mécanismes applicatifs** qui jouent le
rôle de la couche session, et expliquez pourquoi ils sont implémentés
« plus haut »&nbsp;dans la pile.

La couche 5 n'apparît pas car en HTTP/1.1, les requête sont indépendantes les une des autres. Donc à chaque requête, le serveur ferme la session précedente. Deux éléments qui permettent de garder une session active sont les cookies et les token, ils permettent d'authentifier un utilisateur à chaque requête sans fermer la session. Or, TCP/IP n'a que 4 couches par rapport au modèle OSI, donc ces éléments sont présents dans la couche "Apllication", couche 7 du modèle OSI. C'est la couche applicative qui définit si une session doit rester active ou non.

## Pièges fréquents

* **Capture vide (`Number of packets: 0`)** — vous avez lancé `curl` *avant*
  tcpdump, ou tcpdump a été tué avant d'écrire son buffer. Solution : utilisez
  bien `-U -c 30` (étape 2) et déclenchez le trafic *après* le message
  `listening on eth0…`.
* **`tcpdump: /tmp/http.pcap: Permission denied`** — un fichier appartenant à
  l'utilisateur `tcpdump` (créé par une capture précédente) bloque
  l'écriture. Solution : `docker exec lab_client rm -f /tmp/http.pcap`.
* **`tshark … | less` n'affiche rien** — vous êtes dans un environnement
  sans TTY (script, pipeline). Retirez `| less` ou utilisez
  `docker exec -it lab_client sh -c "… | less"`.
