# Exercice 3 — DHCP : obtenir une adresse IP automatiquement

**Niveau :** ⭐⭐ · **Durée :** ~1 h
**📖 Fiche théorique à lire avant :** `https://<votre-site>.pythonanywhere.com/dhcp`

## 🎯 Objectifs

- Constater qu'une machine **sans adresse IP** ne peut pas communiquer.
- Capturer les **4 messages DHCP** (Discover, Offer, Request, ACK = « DORA »).
- Retrouver dans ces messages la configuration reçue : IP, masque, passerelle, DNS, durée du bail.

> 🖥️ Préparation : lab démarré. Ouvrez **deux terminaux client** 💻 et gardez un terminal Codespace 🖥️
> (à la racine du dépôt).

---

## Partie A — Un client sans adresse

### Étape 1 — État de départ

💻 **Terminal client n°1** — notez l'adresse actuelle du client :

```bash
ip -br addr show eth0
```

👀 Vous voyez `172.20.1.50/24`. Cette adresse a été fixée **à la main** (par la configuration
du lab) : c'est une adresse **statique**.

### Étape 2 — Supprimer l'adresse

💻 **Terminal client n°1** — retirez l'adresse IP et faites oublier au client tout bail DHCP précédent :

```bash
ip addr flush dev eth0
rm -f /var/lib/dhcp/dhclient.leases
ip -br addr show eth0
```

💻 Essayez de joindre le routeur :

```bash
ping -c 2 172.20.1.254
```

**Question A.1.** Quel message obtenez-vous ? Pourquoi le client ne peut-il plus rien envoyer ?

> 💬 **Votre réponse :**
>
> `ping: connect: Network is unreachable` : sans adresse IP, le client n'a ni réseau ni route, il ne peut rien envoyer.

---

## Partie B — Capturer l'échange DORA

### Étape 3 — Lancer la capture

💻 **Terminal client n°1** — enregistrez les paquets DHCP (ports UDP 67 et 68) :

```bash
rm -f /tmp/dhcp.pcap
tcpdump -i eth0 -U -nn -c 4 -w /tmp/dhcp.pcap port 67 or port 68
```

👀 `listening on eth0…` puis attente.

### Étape 4 — Demander une adresse

💻 **Terminal client n°2** — lancez le client DHCP en mode bavard (`-v`) :

```bash
dhclient -v eth0
```

👀 Vous devez voir les 4 étapes, dans cet ordre :

```
DHCPDISCOVER on eth0 to 255.255.255.255 port 67 ...
DHCPOFFER of 172.20.1.1xx from 172.20.1.2
DHCPREQUEST for 172.20.1.1xx on eth0 to 255.255.255.255 port 67
DHCPACK of 172.20.1.1xx from 172.20.1.2
bound to 172.20.1.1xx -- renewal in ... seconds.
```

Et dans le terminal n°1, tcpdump s'arrête : `4 packets captured`.

**Question B.1.** Recopiez les 5 lignes affichées par `dhclient`. Quelle adresse IP avez-vous obtenue ?
Qui vous l'a donnée ?

> 💬 **Votre réponse :**
>
> ```
> DHCPDISCOVER on eth0 to 255.255.255.255 port 67 interval 3
> DHCPOFFER of 172.20.1.196 from 172.20.1.2
> DHCPREQUEST for 172.20.1.196 on eth0 to 255.255.255.255 port 67
> DHCPACK of 172.20.1.196 from 172.20.1.2
> bound to 172.20.1.196 -- renewal in 19253 seconds.
> ```
> J'ai obtenu **172.20.1.196**, donnée par le serveur DHCP **172.20.1.2**.

### Étape 5 — Analyser les 4 messages

🖥️ **Terminal Codespace** — listez les paquets capturés :

```bash
./lab/exercises/osi_inspect.py --pcap /tmp/dhcp.pcap
```

🖥️ Puis détaillez chacun d'eux (paquets 1 à 4) :

```bash
./lab/exercises/osi_inspect.py 1 --pcap /tmp/dhcp.pcap
./lab/exercises/osi_inspect.py 2 --pcap /tmp/dhcp.pcap
./lab/exercises/osi_inspect.py 3 --pcap /tmp/dhcp.pcap
./lab/exercises/osi_inspect.py 4 --pcap /tmp/dhcp.pcap
```

**Question B.2.** Complétez le tableau DORA avec **votre** capture :

| Étape       | IP source | IP destination    | MAC destination   | Type DHCP (opt 53) | Transaction ID |
| ----------- | --------- | ----------------- | ----------------- | ------------------ | -------------- |
| 1. Discover | `0.0.0.0` | `255.255.255.255` | `ff:ff:ff:ff:ff:ff` | 1 (DISCOVER) | `0x6a979421` |
| 2. Offer    | `172.20.1.2` | `172.20.1.196` | `2a:5d:5b:71:24:92` | 2 (OFFER) | `0x6a979421` |
| 3. Request  | `0.0.0.0` | `255.255.255.255` | `ff:ff:ff:ff:ff:ff` | 3 (REQUEST) | `0x6a979421` |
| 4. ACK      | `172.20.1.2` | `172.20.1.196` | `2a:5d:5b:71:24:92` | 5 (ACK) | `0x6a979421` |

**Question B.3.** Dans le paquet **Offer** (n°2), relevez la configuration proposée par le serveur :

| Information              | Option DHCP | Valeur |
| ------------------------ | ----------- | ------ |
| Adresse IP proposée      | (yiaddr)    | `172.20.1.196` |
| Masque de sous-réseau    | 1           | `255.255.255.0` |
| Passerelle (routeur)     | 3           | `172.20.1.254` |
| Serveurs DNS             | 6           | `1.1.1.1`, `8.8.8.8` |
| Durée du bail (secondes) | 51          | `43200` |

Convertissez la durée du bail en heures.

> 💬 **Votre réponse (conversion) :**
>
> 43200 ÷ 3600 = **12 heures**.

---

## Partie C — Vérifier la configuration reçue

### Étape 6 — Le client est configuré

💻 **Terminal client n°2** :

```bash
ip -br addr show eth0
ip route
cat /etc/resolv.conf
ping -c 2 172.20.1.254
```

**Question C.1.** Retrouvez-vous dans `ip addr`, `ip route` et `resolv.conf` les valeurs de la
question B.3 ? Le ping vers le routeur fonctionne-t-il à nouveau ?

> 💬 **Votre réponse :**
>
> Oui : `172.20.1.196/24` dans `ip addr`, `default via 172.20.1.254` dans `ip route`, `1.1.1.1` et `8.8.8.8` dans `resolv.conf`. Le ping vers le routeur fonctionne à nouveau.

### Étape 7 — Le point de vue du serveur

🖥️ **Terminal Codespace** — lisez le journal du serveur DHCP, en ne gardant que les lignes des 4 étapes :

```bash
docker logs lab_dhcp_server 2>&1 | grep -E "DHCP(DISCOVER|OFFER|REQUEST|ACK)"
```

👀 Chaque ligne contient le *transaction ID* (en décimal), l'étape (`DHCPDISCOVER`, `DHCPOFFER`…),
l'adresse IP et l'adresse MAC concernées.

> 💡 Pour voir **tout** le journal (options envoyées, etc.) : `docker logs --tail 40 lab_dhcp_server`.

**Question C.2.** Retrouvez les 4 étapes DORA dans le journal. Quelle **adresse MAC** le serveur
a-t-il associée à l'IP attribuée ?

> 💬 **Votre réponse :**
>
> Les 4 étapes DORA apparaissent. Le serveur a associé l'IP **172.20.1.196** à la MAC **2a:5d:5b:71:24:92**.

---

## Partie D — Comprendre

**Question D.1.** Pourquoi le client utilise-t-il `0.0.0.0` comme **IP source** dans le Discover ?

> 💬 **Votre réponse :**
>
> Le client n'a pas encore d'adresse IP, il utilise donc `0.0.0.0` (« aucune adresse »).

**Question D.2.** Pourquoi le Discover est-il envoyé à `255.255.255.255` et à la MAC `ff:ff:ff:ff:ff:ff` ?
Que signifient ces adresses ?

> 💬 **Votre réponse :**
>
> Le client ne connaît pas encore le serveur DHCP, il envoie donc à tout le monde : `255.255.255.255` = broadcast IP, `ff:ff:ff:ff:ff:ff` = broadcast MAC.

**Question D.3.** À quoi sert le **Transaction ID**, identique dans les 4 messages ?
(Imaginez 30 PC qui démarrent en même temps sur le même réseau.)

> 💬 **Votre réponse :**
>
> Il relie les 4 messages d'un même échange : avec 30 PC en même temps, chaque client reconnaît les réponses qui lui sont destinées.

**Question D.4.** Que se passe-t-il quand le **bail expire** ? Pourquoi les baux ont-ils une durée
limitée plutôt qu'infinie ?

> 💬 **Votre réponse :**
>
> Si le bail n'est pas renouvelé, le client perd son adresse IP. Une durée limitée permet de récupérer les adresses des machines qui ne sont plus sur le réseau.

---

## 🚀 Pour aller plus loin (facultatif)

**P.1 — Rendre son adresse.** 💻 Libérez le bail puis redemandez-en un :

```bash
dhclient -r -v eth0
ip -br addr show eth0
dhclient -v eth0
```

Quel message est envoyé par `dhclient -r` ? Obtenez-vous la même adresse ensuite ? Pourquoi, d'après vous ?

> 💬 **Votre réponse :**
>
> `dhclient -r` envoie un **DHCPRELEASE**. On récupère la même adresse, car le serveur se souvient de l'IP attribuée à cette MAC.

**P.2 — Panne du serveur DHCP.** 🖥️ Arrêtez le serveur, puis 💻 redemandez une adresse :

```bash
# 🖥️ Terminal Codespace (dans le dossier lab)
docker compose stop dhcp-server
```

```bash
# 💻 Terminal client
dhclient -r eth0
rm -f /var/lib/dhcp/dhclient.leases
timeout 30 dhclient -v -1 eth0
```

Que se passe-t-il ? Quelles conséquences dans une entreprise si le serveur DHCP tombe en panne ?
**Pensez à le redémarrer** : 🖥️ `docker compose start dhcp-server`.

> 💬 **Votre réponse :**
>
> Aucune réponse (`No DHCPOFFERS received`), le client reste sans adresse IP. En entreprise, les postes qui démarrent ne peuvent plus se connecter au réseau.

---

## ✅ Ce qu'il faut retenir

- Sans adresse IP, une machine ne peut pas communiquer : DHCP la configure **automatiquement**.
- L'échange se fait en 4 messages : **D**iscover, **O**ffer, **R**equest, **A**CK.
- Le client commence en **broadcast** (il ne connaît encore personne) depuis `0.0.0.0`.
- Le serveur fournit IP, masque, passerelle, DNS, et une **durée de bail**.

## 🆘 En cas de problème

| Problème | Solution |
| -------- | -------- |
| Seulement 2 paquets (REQUEST/ACK), pas de Discover | Le client se souvenait d'un ancien bail. 💻 `dhclient -r eth0; rm -f /var/lib/dhcp/dhclient.leases` puis recommencez l'étape 3 |
| `hostname: you must be root to change the host name` | Message sans importance, ignorez-le |
| `dhclient` attend sans fin | Le serveur DHCP est-il démarré ? 🖥️ `docker compose ps` |
| Remettre le client dans son état initial (`172.20.1.50`) | 🖥️ `docker compose restart client` |