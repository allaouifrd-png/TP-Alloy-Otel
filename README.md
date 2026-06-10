# TP-Alloy-Otel

**Exercice 1  ·**  **Mettre Alloy en route**                                                                     

**Objectif :** lancer un conteneur Alloy avec un pipeline minimal — un receiver OTLP relié à un exporteur debug — puis ouvrir l'UI pour inspecter le graphe de composants.

Pour commencer cette exercie, je vais créer un dossier nommée **alloy-lab** et me placer dans ce dossier :

```bash
mkdir alloy-lab
cd alloy-lab
```
Ensuite, je vais crée un fichier nommé **config.alloy** et y insérer la configuration suivante : 

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
La partie ci-dessous, permet d'écouter les données OpenTelemetry 

```bash
otelcol.receiver.otlp "default"
```
Toute les communications OTLP/gRPC sont acceptées 

```bash
grpc {
 endpoint = "0.0.0.0:4317"
}
```
et les communications OTLP/HTTP sont également acceptées

```bash
http {
 endpoint = "0.0.0.0:4318"
}
```
Cette partie, indique où envoyer les données reçues. Dans l'exemple ci-dessous, les metrics, logs et traces sont envoyés vers l'exporteur debug. 

```bash
output {
 metrics = [...]
 logs = [...]
 traces = [...]
}
```
La partie ci-dessous, permet d'afficher les données reçues dans les logs du conteneur :

```bash
otelcol.exporter.debug "default"
```
et cette partie permet juste d'afficher le contenu complet des signaux :

```bash
verbosity = "detailed"
```
Après avoir créer le fichier config.alloy, je vais lancer le conteneur alloy et monter le volume contenant le fichier de configuration alloy situé sur l'hôte dans le conteneur. 

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
```
Grâce a cette commande je peux m'assurer que Alloy fonctionne correctement 

```bash
curl -s http://localhost:12345/-/ready
```
La commande ci-dessous me permet de voir les logs du conteneur Alloy :
```bash
docker logs alloy
```

Sur l'interface graphique, je peux visualiser le graph Alloy :

