# Exercice 5 (Bonus) — Un faux serveur DHCP sur le réseau

**Niveau :** ⭐⭐⭐ · **Durée :** ~45 min
**📖 Fiche théorique à relire :** `https://<votre-site>.pythonanywhere.com/dhcp` (section *Points de vigilance*)

## 🎯 Objectifs

- Comprendre pourquoi DHCP est **vulnérable** : le client fait confiance au premier serveur qui répond.
- Observer un faux serveur (*rogue DHCP*) détourner un client.
- Utiliser un petit script de **détection**.

> ⚠️ Cette attaque se fait **uniquement dans le lab**. Lancer un faux serveur DHCP sur un vrai
> réseau (école, entreprise, Wi-Fi public) est **interdit** et perturbe tous les utilisateurs.

> 🖥️ Préparation : avoir terminé l'exercice 3. Ouvrez **deux terminaux client** 💻 et gardez un terminal Codespace 🖥️.

---

## Partie A — L'attaquant arrive

Un cinquième conteneur, `rogue-dhcp` (`172.20.1.99`), est déjà préparé dans le lab mais ne démarre
pas par défaut. Sa configuration ([`lab/rogue/dnsmasq.conf`](../rogue/dnsmasq.conf)) distribue des
adresses avec **lui-même comme passerelle et comme DNS**.

🖥️ **Terminal Codespace** (dans le dossier `lab`) :

```bash
docker compose --profile bonus up -d rogue-dhcp
docker compose --profile bonus ps
```

👀 Il y a maintenant **5** machines, dont `lab_rogue_dhcp`.

**Question A.1.** Ouvrez le fichier `lab/rogue/dnsmasq.conf` dans Codespace. Quelle passerelle et quel
DNS ce serveur donne-t-il aux clients ? Comparez avec `lab/dhcp-server/dnsmasq.conf`.

> 💬 **Votre réponse :**
>
> Le faux serveur donne **`172.20.1.99`** (lui-même) comme passerelle **et** comme DNS, avec la plage `172.20.1.150`–`160` et un bail d'1 h.
> Le vrai serveur donne la passerelle **`172.20.1.254`** (le routeur) et les DNS **`1.1.1.1`, `8.8.8.8`**, avec la plage `172.20.1.100`–`200` et un bail de 12 h.

---

## Partie B — La course aux offres

### Étape 1 — Écouter les offres

💻 **Terminal client n°1** — affichez chaque **Offer** reçue (serveur, IP proposée, passerelle proposée) :

```bash
tshark -i eth0 -l -Y "dhcp.option.dhcp == 2" -T fields \
  -e dhcp.option.dhcp_server_id -e dhcp.ip.your -e dhcp.option.router 2>/dev/null
```

### Étape 2 — Demander une adresse

💻 **Terminal client n°2** — demandez une nouvelle adresse, puis regardez votre passerelle :

```bash
dhclient -r eth0; rm -f /var/lib/dhcp/dhclient.leases
dhclient -v eth0
ip route
```

Répétez l'étape 2 **trois fois**.

**Question B.1.** Combien d'**Offers** voyez-vous dans le terminal n°1 pour chaque demande ?
De quels serveurs viennent-elles ?

> 💬 **Votre réponse :**
>
> **2 Offers** pour chaque demande :
> - `172.20.1.99` (faux serveur) propose `172.20.1.156` avec la passerelle `172.20.1.99` ;
> - `172.20.1.2` (vrai serveur) propose `172.20.1.196` avec la passerelle `172.20.1.254`.

**Question B.2.** Quelle offre le client a-t-il acceptée (ligne `DHCPOFFER of … from …` et `ip route`) ?
D'après vous, selon quel critère le client choisit-il ?

> 💬 **Votre réponse :**
>
> Les 3 fois, le client a accepté l'offre du **faux serveur** : `DHCPOFFER of 172.20.1.156 from 172.20.1.99` et `default via 172.20.1.99`.
> Le client prend **la première offre reçue**, sans vérifier qui l'envoie : DHCP n'authentifie pas les serveurs.
> Le vrai serveur a même répondu `DHCPNAK`, mais le client a quand même accepté l'ACK du faux serveur.

---

## Partie C — Les conséquences

💻 **Terminal client n°2** — essayez d'accéder au site web :

```bash
ip route
curl -m 5 http://172.20.0.10/whoami
```

**Question C.1.** Le site répond-il ? Expliquez pourquoi, en regardant la passerelle du client.

> 💬 **Votre réponse :**
>
> Non : `curl: (28) Connection timed out after 5001 milliseconds`.
> La passerelle du client est `172.20.1.99` (le faux serveur) au lieu de `172.20.1.254` (le vrai routeur). Le faux serveur ne fait pas suivre le trafic, donc la requête n'atteint jamais le site.

**Question C.2.** Ici, le faux serveur ne fait pas suivre le trafic : le client perd simplement son accès.
Un vrai attaquant ferait suivre le trafic **tout en le lisant**. Que pourrait-il voir ou modifier ?
(Pensez aussi au **DNS** qu'il a distribué.)

> 💬 **Votre réponse :**
>
> Tout le trafic du client passe par lui (attaque **Man-in-the-Middle**) : il peut lire les données non chiffrées (mots de passe en HTTP, contenus) et les modifier.
> Comme il est aussi le **DNS**, il peut répondre une fausse IP pour un nom de site et renvoyer le client vers un faux site (hameçonnage).

---

## Partie D — Détecter l'attaque

Un script de détection est fourni : [`detect_rogue_dhcp.sh`](detect_rogue_dhcp.sh). Ouvrez-le et lisez-le.

💻 Arrêtez tshark dans le **terminal client n°1** (<kbd>Ctrl</kbd>+<kbd>C</kbd>) et ressortez du client (`exit`).
Dans ce même terminal, depuis la racine du dépôt 🖥️, lancez le détecteur sur le poste client :

```bash
docker exec -i lab_client bash < lab/exercises/detect_rogue_dhcp.sh
```

💻 **Terminal client n°2** — redemandez une adresse :

```bash
dhclient -r eth0; rm -f /var/lib/dhcp/dhclient.leases
dhclient -v eth0
```

👀 Le détecteur affiche une ligne `OK` pour le vrai serveur et une ligne `ALERTE` pour l'intrus.

**Question D.1.** Recopiez la sortie du détecteur.

> 💬 **Votre réponse :**
>
> ```
> Surveillance des offres DHCP sur eth0 (Ctrl+C pour arrêter)...
> OK     : offre du serveur autorisé 172.20.1.2
> ALERTE : serveur DHCP NON autorisé détecté : 172.20.1.99
> ```

**Question D.2.** Expliquez avec vos mots ce que fait le script : quelle information lit-il dans
chaque Offer, et comment décide-t-il de lever une alerte ?

> 💬 **Votre réponse :**
>
> Le script écoute le réseau avec `tshark` et garde seulement les **Offers** (type DHCP = 2). Dans chacune, il lit l'**adresse du serveur** (option 54).
> Il la compare au seul serveur autorisé (`172.20.1.2`) : identique → `OK`, différente → `ALERTE`.

**Question D.3.** Ce script **détecte** l'attaque, mais l'**empêche**-t-il ? Sur les vrais réseaux, les
switchs proposent une protection appelée **DHCP Snooping** : seuls les ports déclarés « de confiance »
ont le droit d'envoyer des Offers. Expliquez pourquoi cette solution est plus efficace.

> 💬 **Votre réponse :**
>
> Non, il **détecte** seulement : le client a quand même accepté l'offre du faux serveur.
> Le **DHCP Snooping** bloque les Offers directement sur le switch : une Offer qui arrive par un port non déclaré de confiance est jetée. Elle n'atteint jamais les clients, donc l'attaque est **empêchée**, pas seulement signalée.

---

## Nettoyage

🖥️ Arrêtez le détecteur (<kbd>Ctrl</kbd>+<kbd>C</kbd>), supprimez le faux serveur, puis remettez le client à zéro :

```bash
docker compose --profile bonus stop rogue-dhcp
docker compose --profile bonus rm -f rogue-dhcp
docker compose restart client
```

---

## ✅ Ce qu'il faut retenir

- DHCP n'authentifie pas les serveurs : le client accepte en général **la première offre** reçue.
- Un faux serveur peut imposer sa **passerelle** et son **DNS**, et donc détourner tout le trafic.
- On peut **détecter** les offres illégitimes (surveillance) et les **bloquer** (DHCP Snooping sur les switchs).

## 🆘 En cas de problème

| Problème | Solution |
| -------- | -------- |
| `no such service: rogue-dhcp` | N'oubliez pas `--profile bonus` dans la commande |
| Le vrai serveur gagne à chaque fois | C'est possible ! Répétez la demande plusieurs fois et comparez |
| Le détecteur n'affiche rien | Vérifiez qu'il tourne bien, puis relancez `dhclient` dans l'autre terminal |
| `bash: lab/exercises/detect_rogue_dhcp.sh: No such file` | Placez-vous à la racine du dépôt : `cd /workspaces/*/` |