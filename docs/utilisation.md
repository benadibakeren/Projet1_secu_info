# Guide d'utilisation

## 1. Lancer l'environnement (VM Ubuntu)

```bash
cd Projet1_secu_info/docker && docker compose up -d      # Elasticsearch + Kibana
sudo systemctl restart syslog-ng                         # collecte
sudo snort -A fast -q -c /etc/snort/snort.conf -i <INTERFACE>   # détection (terminal dédié)
```

Ouvrir Kibana : `http://<IP_UBUNTU>:5601`.

## 2. Créer la vue de données Kibana

1. **Stack Management → Data Views → Create data view**.
2. Nom et motif d'index : `securite-logs`.
3. Champ d'horodatage : `date`.
4. Ouvrir **Discover** et choisir la plage de temps.

## 3. Test sans attaque (valide syslog-ng → Elasticsearch → Kibana)

Utile aussi pour développer la notification sans lancer Snort :

```bash
echo '10/09-12:00:00.000000  [**] [1:1000001:1] PING detecte [**] [Priority: 0] {ICMP} 192.168.1.100 -> 192.168.1.200' \
  | sudo tee -a /var/log/snort/alert
curl "http://localhost:9200/securite-logs/_search?pretty"
```

La ligne doit apparaître dans Elasticsearch, puis dans Discover.

## 4. Reproduire un scénario

1. Sur Ubuntu : Snort et syslog-ng en marche.
2. Depuis Kali : lancer l'attaque du scénario (voir [scenarios.md](scenarios.md)).
3. Dans Kibana (Discover), filtrer par exemple : `message : "Scan de ports detecte"`.
4. Vérifier l'alerte et, si elle est configurée, la notification.

Les scénarios 4 et 5 ciblent le port 80 : un serveur web doit tourner sur la victime, par exemple :
```bash
sudo apt install apache2 -y        # ou : sudo python3 -m http.server 80
```

## 5. Arrêt

```bash
docker compose down        # depuis le dossier docker/
sudo systemctl stop syslog-ng
# Snort : Ctrl+C dans son terminal
```

## 6. Dépannage

| Problème | Piste |
|---|---|
| Elasticsearch redémarre en boucle | `vm.max_map_count` trop bas ou RAM insuffisante |
| Aucun log dans Kibana | `systemctl status syslog-ng`, vérifier que `/var/log/snort/alert` est alimenté, tester `curl localhost:9200` |
| Snort n'écrit rien | Mauvaise interface (`-i`), `HOME_NET` incorrect, `local.rules` non inclus |
| Pas de résultat dans Discover | Mauvaise plage de temps ou champ d'horodatage |
