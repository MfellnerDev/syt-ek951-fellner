# syt-ek951-fellner

***
**Autor**: Manuel Fellner
**Version**: 24.01.2025

## 1. Theoretisches
### 1.1 Kafka

- Plattform für verteilte Daten-Streams
- Wird genutzt, um Daten in Echtzeit zu übertragen, zu speichern und zu verarbeiten
- Funktioniert mit sogenannten "topics", in denen producer schreiben und consumer lesen
- Ist gut skalierbar, zuverlässig und wird oft für Event-Streaming, Log-Management oder als Bindeglied in sonstigen modernen Datenarchitekturen genutzt

### 1.2 MinIO

- Object-Storage der sich wie z.B. Amazon S3 verhält (-> API-kompatibel)
- Wird verwendet, um große Mengen an unstrukturierten Daten wie Bilder, Videos, Backups oder Log-Files zu speichern
- Ist schnell und light-weight
- Eignet sich gut für Cloud-native Anwendungen oder Self-Hosted setups


### 1.3 Kind

- steht für **Kubernetes IN Docker**
- Tool, mit dem sich Kubernetes-Cluster einfach lokal erstellen lassen
- Nutzt Docker-Container, um die Knoten des Clusters zu simulieren

## 2. Umsetzung eines konkreten Workflows mittels Kafka und MinIO

### 2.1 Kubernetes cluster aufsetzen

Wir erstellen folgendes YAML-File namens `kind-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
```

Dieses Cluster starten wir nun mit dem folgenden Befehl:

```shell
$ kind create cluster --name minio-kafka --config kind-config.yaml

$ Set kubectl context to "kind-minio-kafka"
```

Nun können wir das Cluster mit dem folgenden Befehl accessen:

```shell
$ kubectl cluster-info --context kind-minio-kafka
```

Wir bekommen hier die folgende Ausgabe:


![](https://uploads.mfellner.com/xR45sg1CV31w.png)

Wenn wir dann mit dem folgenden Befehl die aktuell betriebene Nodes auflisten, sollte dies folgendermaßen aussehen:

```shell
$ kubectl get no
```

![](https://uploads.mfellner.com/LoHFHuPfdMTy.png)

### 2.2 Kafka installieren

Nun installieren wir Kafka. Um jedoch gescheit zu laufen, benötigen wir davor noch die folgenden Services:

- Certmanager
- Zookeper

Diese installieren wir folgendermaßen:

```shell
$ $ kubectl create -f https://github.com/jetstack/cert-manager/releases/download/v1.6.2/cert-manager.yaml

$ kubectl get ns

$ kubectl get po -n cert-manager
```

Certmanager wäre damit erstmal installiert.

Zookeper instalieren wir mittels [helm](https://helm.sh/docs/intro/install/?ref=blog.min.io):

```shell
$ helm repo add pravega https://charts.pravega.io

$ helm repo update
  
$ helm install zookeeper-operator --namespace=zookeeper --create-namespace pravega/zookeeper-operator

```

Nun deployen wir noch den zookeper service:

```shell
$ vim zookeper

apiVersion: zookeeper.pravega.io/v1beta1  
kind: ZookeeperCluster  
metadata:  
    name: zookeeper  
    namespace: zookeeper  
spec:  
    replicas: 1
```


Nun verifizieren wir, dass der Zookeper service läuft:

```shell
$ kubectl -n zookeeper get po
```

Wir bekommen hier die folgende Ausgabe:

![](https://uploads.mfellner.com/bc2gFT8aT8SP.png)

Nun können wir endlich die Kafka Komponenten installieren.

Kafka hat einen Operator namens **Koperator**, welchen wir verwenden, um unsere Kafka Installation zu verwalten.

Wir installieren diesen mit dem folgenden Befehl:

```shell
$ kubectl create --validate=false -f https://github.com/banzaicloud/koperator/releases/download/v0.21.2/kafka-operator.crds.yaml

$ helm repo add banzaicloud-stable https://kubernetes-charts.banzaicloud.com/
```

Hier bekommen wir den folgen Fehler:

