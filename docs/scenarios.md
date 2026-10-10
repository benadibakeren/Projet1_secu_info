# Scénarios d'attaque et détection

Ce document décrit les 5 scénarios d'attaque réalisés contre la machine victime,
et la façon dont le système de détection (Snort) les repère.

## Environnement

| Élément            | Valeur                    |
|--------------------|---------------------------|
| Machine victime    | Ubuntu, 192.168.56.101    |
| Machine attaquante | Kali Linux, 192.168.56.102|
| IDS                | Snort 2.9.20              |
| Interface écoutée  | enp0s8 (réseau host-only) |
| Fichier de règles  | /etc/snort/rules/local.rules |
| Chaîne de traitement | Attaque -> Snort -> syslog-ng -> Elasticsearch -> Kibana |

## Schéma du flux

```
Machine Kali (attaque)
        |
        v
   Snort (détecte et écrit une alerte)
        |
        v
   syslog-ng (lit le fichier d'alertes et l'envoie)
        |
        v
   Elasticsearch (stocke les alertes)
        |
        v
   Kibana (affiche les alertes)
```

## Tableau récapitulatif

| # | Attaque              | Outil (Kali) | Règle SID | Ce qui est détecté            | MITRE ATT&CK |
|---|----------------------|--------------|-----------|-------------------------------|--------------|
| 1 | Balayage ICMP (ping) | ping         | 1000001   | Paquets ICMP vers la victime  | T1595 Active Scanning |
| 2 | Scan de ports        | nmap         | 1000002   | Rafale de demandes de connexion | T1595.001 Scanning IP Blocks |
| 3 | Force brute SSH      | hydra        | 1000003   | Tentatives répétées sur le port 22 | T1110 Brute Force |
| 4 | Parcours de répertoires | curl      | 1000004   | Motif "../" dans une requête web | T1083 File and Directory Discovery |
| 5 | Injection SQL        | curl         | 1000005   | Motif "1=1" dans une requête web | T1190 Exploit Public-Facing Application |

---

## Scénario 1 : Balayage ICMP (ping)

**Description.** L'attaquant envoie des paquets ICMP (ping) pour vérifier si la
machine victime est en ligne. C'est souvent la toute première étape d'une attaque :
savoir quelles machines répondent sur le réseau.

**Pourquoi ce scénario.** Il illustre la phase de reconnaissance, la plus simple,
et sert de test de base pour valider toute la chaîne de détection.

**Commande d'attaque (Kali) :**
```bash
ping -c 4 192.168.56.101
```

**Règle Snort :**
```
alert icmp any any -> $HOME_NET any (msg:"PING detecte"; sid:1000001; rev:1;)
```

Explication :
- `alert icmp` : lève une alerte sur le trafic ICMP (le protocole du ping)
- `any any -> $HOME_NET any` : de n'importe quelle source vers le réseau surveillé
- `msg` : le texte affiché dans l'alerte
- `sid:1000001` : numéro d'identifiant unique de la règle (les règles personnelles commencent à 1000000)

**Alerte générée :**
```
[**] [1:1000001:1] PING detecte [**] [Priority: 0] {ICMP} 192.168.56.102 -> 192.168.56.101
```

**Capture Kibana :** voir `captures/ping.png`

**Contre-mesure.** Configurer le pare-feu pour limiter ou bloquer les requêtes ICMP
entrantes, ou appliquer une limitation de débit pour ne pas répondre à un balayage massif.

---

## Scénario 2 : Scan de ports

**Description.** L'attaquant teste de nombreux ports de la victime pour découvrir
quels services sont ouverts (SSH, web, etc.). C'est la suite logique de la reconnaissance.

**Pourquoi ce scénario.** Un scan de ports précède presque toujours une attaque
ciblée : il révèle la surface d'attaque disponible.

**Commande d'attaque (Kali) :**
```bash
sudo nmap -sS 192.168.56.101
```

**Règle Snort :**
```
alert tcp any any -> $HOME_NET any (msg:"Scan de ports detecte"; flags:S; detection_filter:track by_src, count 20, seconds 3; sid:1000002; rev:1;)
```

Explication :
- `flags:S` : repère les paquets SYN, c'est-à-dire les demandes de connexion
- `detection_filter:track by_src, count 20, seconds 3` : ne lève l'alerte que si plus de 20 demandes arrivent de la même source en 3 secondes. C'est la signature d'un scan, pas d'une connexion normale.

**Alerte générée :**
```
[**] [1:1000002:1] Scan de ports detecte [**] [Priority: 0] {TCP} 192.168.56.102:63876 -> 192.168.56.101:8087
```
(une ligne par port testé)

**Capture Kibana :** voir `captures/scan.png`

**Contre-mesure.** Détecter et bloquer automatiquement les sources qui scannent
(par exemple avec un pare-feu dynamique), et n'exposer que les ports strictement nécessaires.

---

## Scénario 3 : Force brute SSH

**Description.** L'attaquant essaie de se connecter en SSH en testant une liste de
mots de passe à la suite, dans l'espoir d'en trouver un valide.

**Pourquoi ce scénario.** La force brute est une des techniques les plus courantes
pour obtenir un accès. Elle vise directement l'authentification.

**Prérequis côté victime :** un serveur SSH installé et actif (openssh-server).

**Commandes d'attaque (Kali) :**
```bash
echo -e "123456\npassword\nadmin\nkali\ntest\nroot\nazerty\nqwerty\nbonjour\n1234" > passwords.txt
hydra -l benadibk -P passwords.txt ssh://192.168.56.101
```

**Règle Snort :**
```
alert tcp any any -> $HOME_NET 22 (msg:"Force brute SSH detectee"; flags:S; detection_filter:track by_src, count 5, seconds 10; sid:1000003; rev:1;)
```

Explication :
- `$HOME_NET 22` : cible le port 22, celui du service SSH
- `detection_filter:... count 5, seconds 10` : plus de 5 tentatives de connexion en 10 secondes depuis la même source déclenche l'alerte

**Alerte générée :**
```
[**] [1:1000003:1] Force brute SSH detectee [**] [Priority: 0] {TCP} 192.168.56.102:33540 -> 192.168.56.101:22
```

**Capture Kibana :** voir `captures/ssh.png`

**Contre-mesure.** Installer fail2ban pour bannir une adresse après plusieurs échecs,
désactiver l'authentification par mot de passe au profit des clés SSH, et interdire
la connexion directe du compte root.

---

## Scénario 4 : Parcours de répertoires (directory traversal)

**Description.** L'attaquant injecte des `../` dans une URL pour tenter de sortir du
dossier web et lire des fichiers système sensibles, comme `/etc/passwd`.

**Pourquoi ce scénario.** C'est une attaque web classique qui vise la confidentialité
des fichiers du serveur.

**Prérequis côté victime :** un serveur web actif (Apache) sur le port 80.

**Commande d'attaque (Kali) :**
```bash
curl --path-as-is "http://192.168.56.101/../../../../etc/passwd"
```

**Règle Snort :**
```
alert tcp any any -> $HOME_NET 80 (msg:"Directory traversal detecte"; content:"../"; sid:1000004; rev:1;)
```

Explication :
- `$HOME_NET 80` : surveille le trafic web (port 80)
- `content:"../"` : cherche la chaîne `../` dans la requête, caractéristique d'une tentative de remontée de dossiers

**Alerte générée :**
```
[**] [1:1000004:1] Directory traversal detecte [**] [Priority: 0] {TCP} 192.168.56.102:33314 -> 192.168.56.101:80
```

**Capture Kibana :** voir `captures/traversal.png`

**Contre-mesure.** Valider et nettoyer les chemins côté serveur, interdire les `../`
dans les entrées utilisateur, et restreindre les droits d'accès du serveur web.

---

## Scénario 5 : Injection SQL

**Description.** L'attaquant insère du code SQL (par exemple `1=1`) dans un paramètre
d'URL pour fausser une requête en base de données et contourner un contrôle, par
exemple une authentification.

**Pourquoi ce scénario.** L'injection SQL est l'une des attaques web les plus
répandues et les plus dangereuses (elle peut exposer toute une base de données).

**Prérequis côté victime :** un serveur web actif (Apache) sur le port 80.

**Commande d'attaque (Kali) :**
```bash
curl "http://192.168.56.101/?id=1%20OR%201=1"
```
(`%20` représente l'espace dans l'URL)

**Règle Snort :**
```
alert tcp any any -> $HOME_NET 80 (msg:"Injection SQL detectee"; content:"1=1"; sid:1000005; rev:1;)
```

Explication :
- `content:"1=1"` : cherche le motif `1=1`, très souvent utilisé dans les injections
  pour rendre une condition toujours vraie

**Alerte générée :**
```
[**] [1:1000005:1] Injection SQL detectee [**] [Priority: 0] {TCP} 192.168.56.102:52370 -> 192.168.56.101:80
```

**Capture Kibana :** voir `captures/SQL.png`

**Contre-mesure.** Utiliser des requêtes préparées (requêtes paramétrées), valider
les entrées utilisateur, et placer un pare-feu applicatif web devant le serveur.

---

## Limites de la détection

Notre détection repose sur des signatures simples, ce qui a des limites qu'il faut assumer :

- La règle d'injection SQL ne repère que le motif `1=1`. Un attaquant pourrait
  utiliser une autre formulation ou encoder sa charge pour passer inaperçu.
- La règle de parcours de répertoires repère `../`, mais un encodage de ce motif
  (par exemple `%2e%2e%2f`) ne serait pas détecté par cette seule règle.
- La détection de force brute et de scan repose sur des seuils. Une attaque très lente
  (une tentative toutes les minutes) passerait sous le seuil.

Pour aller plus loin, on pourrait ajouter des règles couvrant ces variantes, utiliser
des préprocesseurs de Snort, ou combiner un second outil comme Suricata.