# Installation

Deux machines virtuelles : **Ubuntu Desktop 24.04** (victime, héberge Snort, syslog-ng et la pile ELK) et **Kali Linux** (attaquant).
Dans la suite, `<IP_UBUNTU>` et `<IP_KALI>` désignent leurs adresses sur le réseau privé (par défaut `192.168.56.101` et `192.168.56.102`), et `<INTERFACE>` l'interface host-only de la victime (par défaut `enp0s8`).

## 1. Machines virtuelles

1. Installer **VirtualBox**.
2. Créer la VM **Ubuntu Desktop 24.04** : 2 vCPU et **6 à 8 Go de RAM** (Elasticsearch est gourmand, 4 Go est un minimum juste). Disque 30 Go.
3. Créer la VM **Kali Linux** (image pré-construite pour VirtualBox recommandée) : 2 vCPU, 2 Go de RAM. Identifiants par défaut : `kali` / `kali`.
4. Configurer **deux cartes réseau sur chaque VM** :
   - **Carte 1 en NAT** : donne accès à internet (pour installer les paquets).
   - **Carte 2 en réseau privé hôte (host-only)** : permet aux deux VMs de communiquer entre elles, sur le réseau `192.168.56.0/24`.
5. Vérifier que les deux machines se voient. Depuis Kali :
   ```bash
   ping <IP_UBUNTU>
   ```
6. Sur Ubuntu, mettre à jour le système :
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

## 2. Docker, Elasticsearch et Kibana

1. Installer Docker et Compose :
   ```bash
   sudo apt install -y docker.io docker-compose-v2
   sudo systemctl enable --now docker
   sudo usermod -aG docker $USER
   ```
   Se déconnecter puis se reconnecter (ou redémarrer la VM) pour utiliser `docker` sans `sudo`.
2. Autoriser assez de mémoire virtuelle pour Elasticsearch :
   ```bash
   sudo sysctl -w vm.max_map_count=262144
   echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf   # persistant apres reboot
   ```
3. Démarrer la pile depuis le dépôt :
   ```bash
   git clone https://github.com/benadibakeren/Projet1_secu_info.git
   cd Projet1_secu_info/docker
   docker compose up -d
   ```
4. Vérifier (attendre 1 à 2 minutes le temps du démarrage) :
   ```bash
   docker ps
   curl http://localhost:9200
   ```
   Kibana : `http://<IP_UBUNTU>:5601`.

Paramètres du `docker-compose.yml` : images 8.14.0, Elasticsearch en nœud unique (`discovery.type=single-node`), 1 Go de mémoire Java, **sécurité désactivée** (`xpack.security.enabled=false`), ports 9200 et 5601, réseau `securite`. La sécurité est désactivée pour simplifier les tests en laboratoire.

Remarque : les conteneurs ne redémarrent pas automatiquement après un arrêt de la VM. Après chaque reboot, relancer `docker compose up -d` depuis le dossier `docker/`.

## 3. Services cibles sur la victime

Les scénarios d'attaque 3, 4 et 5 visent des services qui doivent tourner sur la victime :
```bash
sudo apt install -y openssh-server apache2
sudo systemctl enable --now ssh apache2
```
(SSH pour la force brute, Apache pour l'injection SQL et le parcours de répertoires)

## 4. Snort (IDS)

1. Installer Snort :
   ```bash
   sudo apt install -y snort
   ```
   Pendant l'installation, indiquer l'interface réseau à surveiller (`<INTERFACE>`, la carte host-only) et le réseau local `HOME_NET` : `192.168.56.0/24`.
2. Installer les règles du projet :
   ```bash
   sudo cp config/snort-local.rules /etc/snort/rules/local.rules
   ```
3. Dans `/etc/snort/snort.conf`, vérifier que `HOME_NET` correspond au réseau de la victime et que la ligne `include $RULE_PATH/local.rules` est active.
4. Tester la configuration :
   ```bash
   sudo snort -T -c /etc/snort/snort.conf -i <INTERFACE>
   ```
   Le message attendu est « Snort successfully validated the configuration! ».
5. Lancer Snort en écrivant les alertes dans `/var/log/snort/alert` :
   ```bash
   sudo snort -A fast -q -c /etc/snort/snort.conf -i <INTERFACE> -l /var/log/snort
   ```
   Cette commande occupe le terminal (Snort reste en écoute). Pour l'arrêter : `Ctrl + C`, ou `sudo pkill snort` depuis un autre terminal.

## 5. syslog-ng (collecte)

1. Installer syslog-ng et son module HTTP :
   ```bash
   sudo apt install -y syslog-ng syslog-ng-mod-http
   ```
2. Installer la configuration du projet :
   ```bash
   sudo cp config/syslog-ng.conf /etc/syslog-ng/conf.d/snort-elastic.conf
   sudo syslog-ng --syntax-only        # verifie la syntaxe
   sudo systemctl restart syslog-ng
   sudo systemctl status syslog-ng
   ```

Rôle de la configuration : la source `s_snort` lit en continu `/var/log/snort/alert` (sans parsing), la destination `d_elastic` envoie chaque ligne par HTTP POST vers `http://localhost:9200/securite-logs/_doc` avec deux champs, `message` (la ligne d'alerte) et `date` (horodatage ISO).

## 6. Kibana : data view

Pour visualiser les alertes dans Kibana, créer une data view :
1. Ouvrir `http://<IP_UBUNTU>:5601` → menu → Stack Management → Data Views → Create data view.
2. Name : `securite-logs`, Index pattern : `securite-logs*`, Timestamp field : `date`.
3. Les alertes sont ensuite visibles dans l'application Discover.

## 7. Notification

Le script `scripts/alerte.py` interroge Elasticsearch et envoie un courriel lors de la détection de nouvelles alertes. [À COMPLÉTER : prérequis Gmail, configuration et automatisation]

## 8. Vérification de bout en bout

Voir [utilisation.md](utilisation.md), section « Test sans attaque » : une ligne ajoutée à `/var/log/snort/alert` doit apparaître dans Kibana. Pour un test réel, lancer une attaque depuis Kali (par exemple `sudo nmap -sS <IP_UBUNTU>`) pendant que Snort tourne, puis vérifier dans Discover.