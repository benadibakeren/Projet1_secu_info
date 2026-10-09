# Analyse et conclusion

## Bilan

Le système répond au sujet : Snort détecte cinq types d'intrusion, syslog-ng transporte les alertes vers Elasticsearch, Kibana permet de les explorer, et un script de notification prévient l'administrateur. L'ensemble se reproduit à partir du dépôt (Docker Compose pour ELK, règles Snort et configuration syslog-ng versionnées).

## Limites

- **Détection par signatures** : seules les attaques décrites par nos règles sont détectées. Une attaque inconnue ou légèrement modifiée passe inaperçue.
- **Règles simples et faux positifs** : la règle `1=1` ou le seuil de scan peuvent déclencher sur du trafic légitime ; à l'inverse, un attaquant lent évite les seuils (faux négatifs). Une alerte doit être validée avant d'être déclarée « confirmée ».
- **Trafic chiffré** : un IDS réseau ne lit pas le contenu HTTPS ou d'un VPN ; les règles web (scénarios 4 et 5) ne voient que le HTTP en clair.
- **Données peu structurées** : syslog-ng envoie la ligne d'alerte brute dans un seul champ `message`. Les IP, le sid et la priorité ne sont pas des champs filtrables séparément, ce qui limite tableaux de bord et agrégations.
- **Robustesse de l'envoi** : le corps JSON est construit par simple substitution de `${MESSAGE}` ; un guillemet dans un message pourrait casser le document.
- **Sécurité de la pile** : Elasticsearch et Kibana tournent sans authentification ni TLS, acceptable en laboratoire seulement.
- **Échelle** : un nœud Elasticsearch, une victime, du trafic généré à la main ; pas de haute disponibilité ni de politique de rétention.
- **Détection sans réponse** : Snort fonctionne en IDS, aucune attaque n'est bloquée.

## Améliorations possibles

- Parser les alertes dans syslog-ng (extraction de `src_ip`, `dest_ip`, `sid`, `priority`) ou passer par un pipeline d'ingestion Elasticsearch, puis construire des tableaux de bord (top sources, alertes par type, courbe dans le temps).
- Basculer en **IPS** (Snort en ligne) ou déclencher des règles de pare-feu à partir des alertes.
- Ajouter une détection par **anomalies** (profil de comportement normal, seuils statistiques) en complément des signatures.
- Utiliser les alertes natives de Kibana pour la notification, avec plusieurs canaux (courriel, SMS, messagerie d'équipe).
- Activer l'authentification et TLS sur la pile ELK, et définir une durée de conservation des logs.
- Corréler d'autres sources (logs du serveur web, journaux système, pare-feu) et synchroniser les horloges (NTP).
- Automatiser le rejeu des cinq scénarios pour valider les règles après chaque modification.

## Perspectives — veille technologique

- **Suricata** : IDS/IPS multi-thread, compatible avec une grande partie des règles de type Snort, avec une sortie JSON native (EVE) qui s'ingère directement dans Elasticsearch.
- **Wazuh** : plateforme open source combinant détection d'intrusion sur l'hôte, surveillance de l'intégrité des fichiers et gestion des logs, déjà citée dans le sujet comme alternative.
- **Elastic Security** : règles de détection et corrélation intégrées à la pile Elastic.
- **UEBA** : analyse du comportement des utilisateurs et des entités pour repérer les écarts par rapport au profil habituel.
- **SOAR / XDR** : automatisation des réponses par playbooks et corrélation multi-sources pour réduire le temps de traitement des incidents.
- **Honeypots** : leurres surveillés fournissant une alerte précoce et des données sur les attaquants.
