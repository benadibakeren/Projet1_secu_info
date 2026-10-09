# Installation

Deux machines virtuelles : **Ubuntu Server** (victime, héberge Snort, syslog-ng et la pile ELK) et **Kali Linux** (attaquant).
Dans la suite, `<IP_UBUNTU>` et `<IP_KALI>` désignent leurs adresses.

## 1. Machines virtuelles

1. Installer **VirtualBox**.
2. Créer la VM **Ubuntu Server** : 2 vCPU et 4 Go de RAM minimum. Si l'installateur se fige avec des messages « soft lockup », ajouter des CPU et désactiver les mises à jour automatiques pendant l'installation.
3. Créer la VM **Kali Linux**.
4. Placer les deux VM sur un réseau commun et vérifier :
   ```bash
   ping <IP_UBUNTU>
   ```
5. Sur Ubuntu, mettre à jour le système :
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

## 2. Docker, Elasticsearch et Kibana

1. Installer Docker et le plugin Compose [À COMPLÉTER : méthode utilisée].
2. Autoriser assez de mémoire virtuelle pour Elasticsearch :
   ```bash
   sudo sysctl -w vm.max_map_count=262144
   ```
3. Démarrer la pile depuis le dépôt :
   ```bash
   git clone https://github.com/benadibakeren/Projet1_secu_info.git
   cd Projet1_secu_info/docker
   docker compose up -d
   ```
4. Vérifier :
   ```bash
   docker ps
   curl http://localhost:9200
   ```
   Kibana : `http://<IP_UBUNTU>:5601`.

Paramètres du `docker-compose.yml` : images 8.14.0, Elasticsearch en nœud unique (`discovery.type=single-node`), 1 Go de mémoire Java, **sécurité désactivée** (`xpack.security.enabled=false`), ports 9200 et 5601, réseau `securite`. La sécurité est désactivée pour simplifier les tests.

## 3. Snort (IDS)

1. Installer Snort :
   ```bash
   sudo apt install snort -y
   ```
   Pendant l'installation, indiquer l'interface réseau à surveiller et le réseau local (`HOME_NET`).
2. Installer les règles du projet :
   ```bash
   sudo cp config/snort-local.rules /etc/snort/rules/local.rules
   ```
3. Dans `/etc/snort/snort.conf`, vérifier que `HOME_NET` correspond au réseau de la victime et que la ligne `include $RULE_PATH/local.rules` est active.
4. Tester la configuration :
   ```bash
   sudo snort -T -c /etc/snort/snort.conf
   ```
5. Lancer Snort en écrivant les alertes dans `/var/log/snort/alert` :
   ```bash
   sudo snort -A fast -q -c /etc/snort/snort.conf -i <INTERFACE>
   ```

## 4. syslog-ng (collecte)

1. Installer syslog-ng et son module HTTP :
   ```bash
   sudo apt install syslog-ng syslog-ng-mod-http -y
   ```
   [À vérifier : le nom du paquet du module HTTP selon la version]
2. Installer la configuration du projet :
   ```bash
   sudo cp config/syslog-ng.conf /etc/syslog-ng/conf.d/snort-elastic.conf
   sudo syslog-ng --syntax-only        # vérifie la syntaxe
   sudo systemctl restart syslog-ng
   sudo systemctl status syslog-ng
   ```

Rôle de la configuration : la source `s_snort` lit en continu `/var/log/snort/alert` (sans parsing), la destination `d_elastic` envoie chaque ligne par HTTP POST vers `http://localhost:9200/securite-logs/_doc` avec deux champs, `message` (la ligne d'alerte) et `date` (horodatage ISO).

## 5. Notification

Le script `scripts/alerte.sh` envoie un courriel lors d'une intrusion confirmée. [A COMPLETER]

## 6. Vérification de bout en bout

Voir [utilisation.md](utilisation.md), section « Test sans attaque » : une ligne ajoutée à `/var/log/snort/alert` doit apparaître dans Kibana.
