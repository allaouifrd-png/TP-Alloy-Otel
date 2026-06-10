# TP-Alloy-Otel

**Exercice 1  ·**  **Mettre Alloy en route**                                                                     
**Objectif :** lancer un conteneur Alloy avec un pipeline minimal — un receiver OTLP relié à un exporteur debug — puis ouvrir l'UI pour inspecter le graphe de composants.

Pour commencer cet exercice, je vais créer un dossier nommé **alloy-lab** et me placer dans ce dossier :

```bash
mkdir alloy-lab
cd alloy-lab
```
Ensuite, je vais créer un fichier nommé **config.alloy** et y insérer la configuration suivante :

```bash
otelcol.receiver.otlp "default" {
  grpc {
    endpoint = "0.0.0.0:4317"
  }

  http {
    endpoint = "0.0.0.0:4318"
  }

  output {
    metrics = [otelcol.exporter.debug.default.input]
    logs    = [otelcol.exporter.debug.default.input]
    traces  = [otelcol.exporter.debug.default.input]
  }
}

otelcol.exporter.debug "default" {
  verbosity = "detailed"
}
```
La partie ci-dessous permet d'écouter les données OpenTelemetry :

```bash
otelcol.receiver.otlp "default"
```
Toutes les communications OTLP/gRPC sont acceptées :

```bash
grpc {
 endpoint = "0.0.0.0:4317"
}
```
Et les communications OTLP/HTTP sont également acceptées :

```bash
http {
 endpoint = "0.0.0.0:4318"
}
```
Cette partie indique où envoyer les données reçues. Dans l'exemple ci-dessous, les métriques, logs et traces sont envoyés vers l'exporteur debug : 

```bash
output {
 metrics = [...]
 logs = [...]
 traces = [...]
}
```
La partie ci-dessous permet d'afficher les données reçues dans les logs du conteneur :

```bash
otelcol.exporter.debug "default"
```
Et cette partie permet simplement d'afficher le contenu complet des signaux :

```bash
verbosity = "detailed"
```
Après avoir créé le fichier **config.alloy**, je vais lancer le conteneur Alloy et monter le volume contenant le fichier de configuration Alloy situé sur l'hôte dans le conteneur. 

Commande pour lancer le conteneur : 

```bash
docker run -d \
  --name alloy \
  -p 4317:4317 \
  -p 4318:4318 \
  -p 12345:12345 \
  -v /home/ubuntu/alloy-lab/config.alloy:/etc/alloy/config.alloy \
  grafana/alloy:v1.5.1 \
  run /etc/alloy/config.alloy \
  --server.http.listen-addr=0.0.0.0:12345
  --stability.level=experimental
```
Important : sans ceci **--stability.level=experimental**, le conteneur ne se lance pas, car comme indiqué dans la documentation technique d’Alloy, certains composants, dont otelcol.exporter.debug, sont encore classés au niveau de stabilité experimental. Par défaut, Alloy n’autorise que les composants generally-available, sauf si ce niveau de stabilité est explicitement activé au démarrage.

Grâce à cette commande, je peux m'assurer qu'Alloy fonctionne correctement : 

```bash
ubuntu@ubuntu-telemetry:~/alloy-lab$ curl -s http://192.168.1.78:12345/-/ready
Alloy is ready.
ubuntu@ubuntu-telemetry:~/alloy-lab$ 
```
La commande ci-dessous me permet de voir les logs du conteneur Alloy :

```bash
docker logs alloy
```
Grâce aux logs, on peut voir que les récepteurs OTLP/gRPC et OTLP/HTTP ont bien démarré et écoutent sur leur port respectif.

```bash
ts=2026-06-10T09:13:42.537790103Z level=info msg="Starting GRPC server" component_path=/ component_id=otelcol.receiver.otlp.default endpoint=0.0.0.0:4317
```

```bash
ts=2026-06-10T09:13:42.537921124Z level=info msg="Starting HTTP server" component_path=/ component_id=otelcol.receiver.otlp.default endpoint=0.0.0.0:4318
```

Sur l'interface graphique, je peux visualiser le graphe Alloy, ce qui indique également que les deux composants sont correctement liés :

<img width="722" height="671" alt="image" src="https://github.com/user-attachments/assets/732112c5-04ef-4643-a703-000064957696" />

**Exercice 2  ·**  **Envoyer des données OTLP avec telemetrygenEnvoyer des données OTLP avec telemetrygen**   

**Objectif :** utiliser l'outil de référence de la communauté OpenTelemetry, telemetrygen, pour pousser de faux traces, métriques et logs dans Alloy. Confirmer leur arrivée en lisant les logs Alloy.

Pour faire cette exercice, comme indiqué dans l'indice le conteneur telemetrygen doit être sur le même réseau Docker qu'Alloy pour résoudre alloy:4317. Donc je vais vérifier le réseau dans lequel tourne le conteneur alloy afin de m'assurer que telemetrygen s'aura bien contacter alloy et lui envoyés les données nécessaires.

Dans mon cas, j’ai lancé le conteneur sans toucher à la configuration réseau, donc tout fonctionne sur le réseau bridge :

```bash
ubuntu@ubuntu-telemetry:~/alloy-lab$ sudo docker inspect alloy | grep NetworkMode
            "NetworkMode": "bridge",
ubuntu@ubuntu-telemetry:~/alloy-lab$ 
```
Ensuite, grâce à la commande ci-dessous, je vais lancer temporairement un conteneur Docker contenant l'image telemetrygen, qui génère et envoie 5 traces OpenTelemetry vers un collecteur OTLP :

```bash
docker run --rm \
  --network bridge \
  ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest \
  traces \
  --otlp-endpoint 172.17.0.1:4317 \
  --otlp-insecure \
  --traces 5
```

Ensuite, grâce à la commande ci-dessous, je vais lancer temporairement un conteneur Docker contenant l'image telemetrygen, qui génère et envoie des métriques OpenTelemetry pendant 10 secondes vers un collecteur OTLP :

```bash
docker run --rm \
  --network bridge \
  ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest \
  metrics \
  --otlp-endpoint 172.17.0.1:4317 \
  --otlp-insecure \
  --duration 10s
```

Ensuite, grâce à la commande ci-dessous, je vais lancer temporairement un conteneur Docker contenant l'image telemetrygen, qui génère et envoie des logs OpenTelemetry pendant une durée de 5 secondes vers un collecteur OTLP :

```bash
docker run --rm \
  --network bridge \
  ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest \
  logs \
  --otlp-endpoint 172.17.0.1:4317 \
  --otlp-insecure \
  --duration 5s
```

Grâce à la commande ci-dessous, je vais vérifier dans les logs du conteneur alloy que les traces, métriques et logs OpenTelemetry ont bien été reçus par le collecteur :

```bash
sudo docker logs alloy 2>&1 | grep -E "msg=Traces|msg=Metrics|msg=Logs"
```
```bash
ubuntu@ubuntu-telemetry:~/alloy-lab$ sudo docker logs alloy 2>&1 | grep -E "msg=Traces|msg=Metrics|msg=Logs"
ts=2026-06-10T09:52:31.689577258Z level=info msg=Traces component_path=/ component_id=otelcol.exporter.debug.default "resource spans"=1 spans=2
ts=2026-06-10T09:52:33.690146916Z level=info msg=Traces component_path=/ component_id=otelcol.exporter.debug.default "resource spans"=1 spans=2
ts=2026-06-10T09:52:35.691684874Z level=info msg=Traces component_path=/ component_id=otelcol.exporter.debug.default "resource spans"=1 spans=2
ts=2026-06-10T09:52:37.693111746Z level=info msg=Traces component_path=/ component_id=otelcol.exporter.debug.default "resource spans"=1 spans=2
ts=2026-06-10T09:52:39.68843139Z level=info msg=Traces component_path=/ component_id=otelcol.exporter.debug.default "resource spans"=1 spans=2
ts=2026-06-10T09:52:55.035977834Z level=info msg=Metrics component_path=/ component_id=otelcol.exporter.debug.default "resource metrics"=1 metrics=12 "data points"=12
ts=2026-06-10T09:53:03.692704903Z level=info msg=Logs component_path=/ component_id=otelcol.exporter.debug.default "resource logs"=1 "log records"=6
ubuntu@ubuntu-telemetry:~/alloy-lab$
```
Les résultats montrent qu'Alloy a correctement reçu et traité les traces, métriques et logs OpenTelemetry envoyés par telemetrygen, ce qui permet de valider le bon fonctionnement de la configuration mise en place.

**Exercice 2  ·**  **Instrumenter une vraie application avec le SDK OpenTelemetry**   

**Objectif :** faire tourner une petite application Flask auto-instrumentée OpenTelemetry. L'application émet traces, métriques et logs en OTLP vers Alloy au fil du trafic généré avec curl.

Dans l'exercice précédent, nous avons envoyé de fausses données à l'aide de telemetrygen. Dans cet exercice, je vais déployer une application Flask et récolter les traces, les métriques et les logs pour les envoyer vers Alloy.

Pour cela, je vais créer un dossier **app** dans lequel je vais créer un fichier **app.py** :

```bash
from flask import Flask
import random
import time

app = Flask(__name__)

@app.route("/")
def home():
    time.sleep(random.uniform(0.1, 0.5))

    if random.random() < 0.1:
        return "simulated error", 500

    return "hello", 200

app.run(host="0.0.0.0", port=5000)
```
Ensuite, j'ai créé un fichier **requirements.txt** qui sert à lister toutes les dépendances Python nécessaires au bon fonctionnement de l'application **app.py** :

```bash
nano requirements.txt
```

```bash
flask
opentelemetry-distro
opentelemetry-exporter-otlp
```

Ensuite, je vais créer un fichier **Dockerfile** dans lequel je vais construire l'application **app.py**, et cette commande :

```bash
RUN pip install -r requirements.txt
```
demande à pip d'installer automatiquement tous les paquets présents dans le fichier.

```bash
FROM python:3.11-slim

WORKDIR /

COPY requirements.txt .
RUN pip install -r requirements.txt

RUN opentelemetry-bootstrap -a install

COPY app.py .

CMD ["opentelemetry-instrument","python","/app.py"]
```
Pour contruire l'image, je vais utiliser cette commande : 

```bash
docker build -t demo-app .
```
Dans cette étape, je vais lancer l'application Docker précédemment construite :

```bash
sudo docker run -d \
  --name app \
  --network bridge \
  -p 5000:5000 \
  -e OTEL_SERVICE_NAME=demo \
  -e OTEL_EXPORTER_OTLP_ENDPOINT=http://172.17.0.1:4318 \
  -e OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf \
  -e OTEL_TRACES_EXPORTER=otlp \
  -e OTEL_METRICS_EXPORTER=otlp \
  -e OTEL_LOGS_EXPORTER=otlp \
  demo-app
```
Pour générer du trafic je vais utiliser cette commande : 

```bash
for i in $(seq 1 30)
do
  curl -s http://192.168.1.78:5000/ >/dev/null
done
```
Les logs montrent bien que le service demo envoie des traces :
```bash
service.name: Str(demo)
ResourceSpans
InstrumentationScope opentelemetry.instrumentation.flask
Name: GET /
```
On observe également des erreurs HTTP 500 générées aléatoirement par l’application :

```bash
Status code    : Error
http.status_code: Int(500)
```
Enfin, les métriques HTTP sont également présentes :

```bash
ResourceMetrics
Name: http.server.duration
http.status_code: Int(200)
http.status_code: Int(500)
```

**Exercice 4  ·**  **Maîtriser la syntaxe Alloy : pipeline, UI, hot reload	Fondamentaux**   

**Objectif :** étendre le pipeline Alloy avec une chaîne de processors entre le receiver OTLP et l'exporteur debug, observer le graphe en direct dans l'UI, et recharger la configuration sans redémarrer Alloy.

Dans cet exercice, je vais ajouter deux processors entre le receiver OTLP et l'exporteur debug. Ensuite, je vais recharger Alloy sans redémarrer le conteneur. 

Dans le fichier **config.alloy** 

Je vais intégrer ce contenu en remplacement de l'ancienne configuration :

```bash
otelcol.receiver.otlp "default" {
  grpc {
    endpoint = "0.0.0.0:4317"
  }

  http {
    endpoint = "0.0.0.0:4318"
  }

  output {
    metrics = [otelcol.processor.attributes.lab.input]
    logs    = [otelcol.processor.attributes.lab.input]
    traces  = [otelcol.processor.attributes.lab.input]
  }
}

otelcol.processor.attributes "lab" {
  action {
    key    = "deployment.environment"
    value  = "lab"
    action = "insert"
  }

  output {
    metrics = [otelcol.processor.batch.default.input]
    logs    = [otelcol.processor.batch.default.input]
    traces  = [otelcol.processor.batch.default.input]
  }
}

otelcol.processor.batch "default" {
  output {
    metrics = [otelcol.exporter.debug.default.input]
    logs    = [otelcol.exporter.debug.default.input]
    traces  = [otelcol.exporter.debug.default.input]
  }
}

otelcol.exporter.debug "default" {
  verbosity = "detailed"
}
```
Ensuite, je vais recharger Alloy à chaud grâce à cette commande : 

```bash
curl -X POST http://192.168.1.78:12345/-/reload
```

Pour vérifier que Alloy fonctionne, je vais utiliser cette commande : 

```bash
curl -s http://192.168.1.78:12345/-/ready
```

Et pour générer du trafic, je vais lancer cette commande : 

```bash
for i in $(seq 1 20); do curl -s http://192.168.1.78:5000/ >/dev/null; done
```

Au niveau des logs, je vois que les processors attributes et batch sont bien chargés par Alloy. L’attribut deployment.environment: Str(lab) apparaît dans les informations reçus.

```bash
node_id=otelcol.processor.batch.default
node_id=otelcol.processor.attributes.lab
deployment.environment: Str(lab)
```

Au niveau du graphe, l’UI Alloy affiche bien les 4 composants du pipeline. Le flux passe du receiver OTLP vers le processor attributes, puis vers le processor batch, puis vers l’exporteur debug.

<img width="1137" height="832" alt="image" src="https://github.com/user-attachments/assets/2a80668b-b865-487c-8de9-9f3e28e72b4e" />

Cet exercice m'a permis de comprendre qu'Alloy fonctionne comme une pipeline de traitement capable d'enrichir (ajouter des labels) et d'optimiser (regrouper les données pour les envoyer par lots) la télémétrie. Les données ne vont plus directement du Receiver à l'Exporter mais passent par le Receiver OTLP --> Processor Attributes --> Processor Batch --> Exporter Debug. De plus, le hot reload permet d'appliquer les changements appliqués dans la configuration d'Alloy, ce qui permet d'éviter l'interruption du service en cas de changement de configuration.

**Exercice 5  ·**  **Scraper des cibles Prometheus et expédier vers Mimir**                                                                     
**Objectif :** déployer Mimir comme backend de métriques, déployer node_exporter comme cible à scraper, configurer Alloy pour le scraper et faire remote_write vers Mimir, et visualiser dans Grafana.

Pour faire cette exercice, je vais créer un répertoire dédiée à Mimir et m'y positionner : 

```bash
mkdir -p mimir
cd mimir
```
Dans le fichier mimir.yaml je vais renseigner le contenu suivant : 

```bash
multitenancy_enabled: false

server:
  http_listen_port: 9009

common:
  storage:
    backend: filesystem

blocks_storage:
  backend: filesystem
  filesystem:
    dir: /data/blocks

compactor:
  data_dir: /data/compactor

ruler:
  rule_path: /data/rules

target: all
```

Ensuite, je vais lancer Mimir grâce a cette commande : 

```bash
docker run -d \
  --name mimir \
  -p 9009:9009 \
  -v /home/ubuntu/mimir/mimir.yaml:/etc/mimir.yaml \
  grafana/mimir:2.14.0 \
  -config.file=/etc/mimir.yaml
```
La commande suivante, me permet de voir les logs du conteneur Mimir : 

```bash
docker logs mimir
```
Grâce à ces logs, je m'aperçoit que Mimir à bien démarrer et écoute sur les ports HTTP 9009 et gRPC 9095

```bash
server listening on addresses http=[::]:9009 grpc=[::]:9095
```
Ensuite, je vais lancer Node Exporter grâce à la commande ci-dessous : 

```bash
docker run -d \
  --name node-exporter \
  -p 9100:9100 \
  prom/node-exporter:v1.8.2
```

La commande ci-dessous permet de vérifier que Node Exporter fonctionne correctement en affichant les métriques système qu'il expose au format Prometheus :

```bash
curl http://192.168.1.78:9100/metrics
```

Ci-dessous l'une des valeurs retournées grâce à la commande précédente : 

```bash
process_virtual_memory_bytes 1.269792768e+09
```
Cette valeur indique que le processus utilise environ 1,27 Go de mémoire virtuelle

Ensuite, je vais déployer Grafana grâce à la commande ci-dessous : 

```bash
docker run -d \
  --name grafana \
  -p 3000:3000 \
  grafana/grafana:11.4.0
```

L'interface d'administration de Grafana est accessible via cette ip : 

```bash
http://192.168.1.78:3000
```

Dans cette étape, je vais modifier Alloy 

```bash
nano /home/ubuntu/alloy-lab/config.alloy
```
Pour ajouter le contenu suivant à la fin :

```bash
prometheus.scrape "node" {
  targets = [{
    __address__ = "192.168.1.78:9100",
  }]

  forward_to = [prometheus.remote_write.mimir.receiver]
}

prometheus.remote_write "mimir" {
  endpoint {
    url = "http://192.168.1.78:9009/api/v1/push"
  }
}
```
Ensuite, je vais recharger Alloy gâce a cette commande :

```bash
curl -X POST http://192.168.1.78:12345/-/reload
```

Et vérifier si Alloy est bien prêt :

```bash
curl http://192.168.1.78:12345/-/ready
```

Après avoir lancé la commande suivante :

```bash
curl -s "http://192.168.1.78:9009/prometheus/api/v1/query?query=up"
```
J'ai eu ce message d'erreur de Mimir "expanding series: too many unhealthy instances in the ring"

Les logs Mimir indique aussi ceci "at least 2 live replicas required, could only find 1" cela indique que Mimir attend plusieurs replicas alors que dans ce TP je n'ai qu'une seul instance Mimir. 

Donc j'ai du modifier le fichier le fichier Mimir.yaml en ajoutant ce bloc a la fin du fichier : 

```bash
ingester:
  ring:
    replication_factor: 1

store_gateway:
  sharding_ring:
    replication_factor: 1
```

J'ai enregistré la configuration et relancé le conteneur avec la nouvelle configuration Mimir : 

```bash
sudo docker restart mimir
```
Grâce a la commande suivante, je peux voir si Mimir reçoit bien les métriques : 

Grâce à la commande suivante, j'ai pu vérifier si Mimir recevait correctement les métriques envoyées par Alloy :

curl -s "http://192.168.1.78:9009/prometheus/api/v1/query?query=up"

Lors de la première exécution, la requête a bien été traitée par Mimir puisque le statut retourné était success. Mias, le champ result était vide, ce qui signifie qu'aucune métrique up n'était encore disponible dans Mimir à ce moment-là.
```bash
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": []
  }
}
```
Quelques instants plus tard, après que Alloy ait effectué le scrape de Node Exporter et transmis les métriques à Mimir, la même requête a retourné le résultat suivant :
```bash
{
  "status":"success",
  "data":{
    "resultType":"vector",
    "result":[
      {
        "metric":{
          "__name__":"up",
          "instance":"192.168.1.78:9100",
          "job":"prometheus.scrape.node"
        },
        "value":[1781099079.415,"1"]
      }
    ]
  }
}
```
La valeur 1 de la métrique up indique que la cible 192.168.1.78:9100, correspondant à Node Exporter, est joignable et que son scrape est effectué correctement par Alloy.

J'ai ensuite ajouté Mimir comme source de données dans Grafana en utilisant le type de datasource Prometheus avec l'URL de Mimir. Dans l'outil Explore, l'exécution de la requête up retourne également la valeur 1, ce qui confirme que les métriques sont bien collectées par Alloy, stockées dans Mimir puis consultables depuis Grafana.

<img width="1898" height="885" alt="image" src="https://github.com/user-attachments/assets/1c1866f6-3498-4444-80a7-59331434a3c6" />

Cette vérification valide le bon fonctionnement de la chaîne complète : Node Exporter → Alloy → Mimir → Grafana.



