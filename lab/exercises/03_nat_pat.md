# Exercice 3 — NAT / PAT en action : règles iptables et table conntrack

**Durée estimée :** 1 h
**Objectif :** observer en direct comment une **adresse IP privée** et un
**port source** sont traduits par le routeur, en lisant la table de suivi
de connexions (`conntrack`). Manipuler les règles `iptables` pour casser
et réparer le NAT.

## Mise en place

Si le client n'a pas de route par défaut, fixez-la&nbsp;:

```bash
docker exec lab_client bash -c "ip route del default 2>/dev/null; ip route add default via 172.20.1.254"
```

## Partie A — Observation passive

Dans un premier terminal, **tournez** `conntrack` en watch sur le routeur&nbsp;:

```bash
docker exec lab_nat_router watch -n 0.5 'conntrack -L 2>/dev/null'
```

Dans un second, lancez plusieurs requêtes en parallèle depuis le client&nbsp;:

```bash
for i in 1 2 3 4 5; do
  docker exec lab_client curl -s -o /dev/null http://172.20.0.10/whoami &
done; wait
```

Et regardez ce que voit le serveur&nbsp;:

```bash
docker exec lab_client curl -s http://172.20.0.10/whoami
docker logs --tail 20 lab_internet
```

> ℹ️ Dans l'image nginx officielle, `/var/log/nginx/access.log` est un
> **symlink vers `/dev/stdout`** — la lecture passe donc par `docker logs`,
> pas par `tail` sur le fichier.

### À rendre — répondez directement dans ce fichier

**Question A.1.** Recopiez **deux lignes** de `conntrack -L` représentatives.
Annotez chaque champ&nbsp;: `src`, `dst`, `sport`, `dport`, **puis le
second tuple** (reply), et expliquez ce que le tuple-reply signifie.

Tuple aller (client > serveur) : 
`src=172.20.1.176` / `dst=172.20.0.10` / `sport=39972` / `dport=80`

Tuple retour (serveur > routeur) :
`src=172.20.0.10` / `dst=172.20.0.254` / `sport=80` / `dport=39972 `

`Le tuple-reply décrit comment le routeur s'attend à voir revenir les paquets de réponse. On observe bien que la destination est le routeur (pas l'ip du client), le MASQUERADE est donc bien actif et c'est le port qui va permettre la communication avec le bon destinataire.` 

**Question A.2.** Quelle IP voit le serveur `internet` dans
`access.log`&nbsp;? Pourquoi pas `172.20.1.50`&nbsp;?

`le serveur voit 172.20.0.254 (l'IP WAN du routeur) et non 172.20.1.50/176, car le MASQUERADE réécrit l'adresse source de chaque paquet sortant.` 

**Question A.3.** Combien de **ports sources distincts** apparaissent
côté NAT pour les 5 requêtes parallèles&nbsp;? Que se passerait-il avec
65&nbsp;000 connexions simultanées&nbsp;? (donnez une borne théorique).

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse._

## Partie B — Casser le NAT et réparer

Affichez la règle MASQUERADE&nbsp;:

```bash
docker exec lab_nat_router iptables -t nat -L POSTROUTING -n -v --line-numbers
```

Supprimez-la&nbsp;:

```bash
docker exec lab_nat_router iptables -t nat -D POSTROUTING 1
```

Relancez `curl http://172.20.0.10/whoami` depuis le client. Que se
passe-t-il **côté client** (timeout, refus, autre)&nbsp;? **Côté serveur**
(log nginx)&nbsp;?

> 💬 **Votre réponse (observation client + serveur) :**
>
> _Remplacez ce texte par votre réponse._

Vérifiez avec un tcpdump sur le routeur, côté WAN&nbsp;:

```bash
docker exec lab_nat_router tcpdump -i eth0 -nn host 172.20.0.10
```

**Question B.** Quelle IP source apparaît dans les paquets sortants&nbsp;?
Pourquoi l'absence de MASQUERADE cause un problème pour la **réponse**
plutôt que pour l'**aller**&nbsp;?

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse._

Remettez la règle&nbsp;:

```bash
docker exec lab_nat_router iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

## Partie C — DNAT (redirection de port entrant)

Le NAT vu jusqu'ici est sortant (source NAT). Ajoutez maintenant une
règle de **DNAT** pour publier le port 80 du serveur `internet` sur
le routeur, accessible depuis le LAN à l'adresse du routeur&nbsp;:

```bash
docker exec lab_nat_router iptables -t nat -A PREROUTING -i eth1 \
    -p tcp --dport 8080 -j DNAT --to-destination 172.20.0.10:80
```

Testez&nbsp;:

```bash
docker exec lab_client curl -s http://172.20.1.254:8080/whoami
```

### À rendre — répondez directement dans ce fichier

**Question C.1.** Quelle IP voit nginx maintenant dans `access.log`&nbsp;?
Comparez avec l'IP vue en partie A et expliquez la différence.

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse._

**Question C.2.** Modifiez la règle pour que nginx voie l'**IP réelle**
du client. Indice&nbsp;: il manque encore une règle de SNAT pour le retour,
OU activez la fonction « hairpin » avec une règle dans `POSTROUTING`.

> 💬 **Votre réponse (règle iptables + observation nginx) :**
>
> _Remplacez ce texte par votre réponse._

**Question C.3.** Donnez **un cas d'usage réel** (datacenter ou
domestique) pour ce couple DNAT + SNAT.

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse._

## Questions de synthèse

**Question S.1 — NAT vs PAT.** Donnez la différence en une phrase, et
indiquez lequel des deux est implémenté par notre `MASQUERADE`. Justifiez.

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse._

**Question S.2 — NAT et sécurité.** Vrai ou faux : « le NAT protège un
réseau interne ». Argumentez en 3-4 phrases (pensez aux connexions
**sortantes**).

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse._

**Question S.3 — IPv6.** IPv6 a globalement supprimé le besoin de NAT.
Citez **deux raisons** pour lesquelles le NAT reste néanmoins utilisé
en IPv6 dans certains contextes.

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse._
