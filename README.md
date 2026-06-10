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


