***
**Autor**: Manuel Fellner
**Version**: 24.01.2025

## 1. Theoretisches
### 1.1 Kafka
- siehe *[1]*
- Plattform für verteilte Daten-Streams
- Wird genutzt, um Daten in Echtzeit zu übertragen, zu speichern und zu verarbeiten
- Funktioniert mit sogenannten "topics", in denen producer schreiben und consumer lesen
- Ist gut skalierbar, zuverlässig und wird oft für Event-Streaming, Log-Management oder als Bindeglied in sonstigen modernen Datenarchitekturen genutzt

### 1.2 MinIO

- siehe *[2]*
- Object-Storage der sich wie z.B. Amazon S3 verhält (-> API-kompatibel)
- Wird verwendet, um große Mengen an unstrukturierten Daten wie Bilder, Videos, Backups oder Log-Files zu speichern
- Ist schnell und light-weight
- Eignet sich gut für Cloud-native Anwendungen oder Self-Hosted setups


### 1.3 Kind

- siehe *[3]*
- steht für **Kubernetes IN Docker**
- Tool, mit dem sich Kubernetes-Cluster einfach lokal erstellen lassen
- Nutzt Docker-Container, um die Knoten des Clusters zu simulieren

## 2. Umsetzung eines konkreten Workflows mittels Kafka und MinIO

Folgende Tutorial wird befolgt: https://blog.min.io/complex-workflows-apache-kafka-minio/

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

- Zookeper *[4]*: Kümmert sich um die Verwaltung und Synchronisation von Kafka-Cluster-Komponenten. Stellt sicher, dass alle Kafka-Broker ("Server") wissen, wann neue Broker hinzukommen oder ausfallen. Enthält ebenso eine Topologie des Clusters
- Certmanager *[5]*: Stellt sicher, dass alle Kafka-Broker TLS-Zertifikate ausgestellt bekommen, um eine verschlüsselte Kommunikation sicherzustellen.

Diese installieren wir folgendermaßen:

```shell
$ $ kubectl create -f https://github.com/jetstack/cert-manager/releases/download/v1.6.2/cert-manager.yaml

$ kubectl get ns

$ kubectl get po -n cert-manager
```

Certmanager wäre damit erstmal installiert.

Zookeper instalieren wir mittels helm *[6]*:

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

Kafka hat einen Operator namens **Koperator** *[7]*, welchen wir verwenden, um unsere Kafka Installation zu verwalten. Dieser ermöglicht im gesamten die Möglichkeit, Kafka deployments und Automatisierungen zu vereinfachen. Er erweitert Kubernetes, um Apache Kafka-Cluster zu verwalten und bietet zahlreiche Funktionen für die Verwaltung, Skalierung und Überwachung von Kafka.

Wir installieren diesen mit dem folgenden Befehl:

```shell
$ kubectl create --validate=false -f https://github.com/banzaicloud/koperator/releases/download/v0.21.2/kafka-operator.crds.yaml

$ helm repo add banzaicloud-stable https://kubernetes-charts.banzaicloud.com/
```

Hier bekommen wir den folgen Fehler:

![](https://uploads.mfellner.com/e5tQoBoF3u6E.png)

Als Text:

```
Error: looks like "https://kubernetes-charts.banzaicloud.com/" is not a valid chart repository or cannot be reached: Get "https://kubernetes-charts.banzaicloud.com/index.yaml": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
```

Nach einer kleinen Recherche fiel auf, dass der Anbieter des Images, in diesem Fall `banzaicloud`, komplett vom Internet verschwunden ist. Zwar bestehen noch die gesamten Repositories, jedoch gibt es keine Website mehr, welche online unter `https://banzaicloud.com` erreichbar ist. 

Deshalb weichen wir in diesem Fall vom Tutorial-Image ab und verwenden das folgende:

```shell
$ helm install strimzi-kafka-operator strimzi/strimzi-kafka-operator --namespace kafka --create-namespace
```

Dies sollte (theoretisch) dasselbe Image sein, wie es Banzaicloud angeboten hat. Damit funktioniert die Installation auch.

Nun führen wir den `kubectl -n kafka get po` Befehl aus um sicherzustellen, dass sich Kafka auch gestartet hat:

![](https://uploads.mfellner.com/XrEvHtTWxcAr.png)

Dies scheint auch der Fall zu sein.

### 2.3 Ein Kafka topic konfigurieren

Ein sogenanntes `topic` ist einfach gesagt die grundlegende Einheit zur Organisation und Speicherung von Daten in Kafka. Es ist basicially ein Stream, in den die producer Nachrichten schreiben und aus dem die consumer Nachrichten lesen können. *[8]*


Aus Tutorial zwecken erstellen wir nun ein Topic, welches `my-topic` heißt. Hierbei müssen wir ein paar Daten anpassen:

```yml

apiVersion: kafka.strimzi.io/v1beta2 # Der Name musste angepasst werden
kind: KafkaTopic
metadata:
  name: my-topic
  namespace: kafka  # Wichtig, dass dieser Name zu unserem erstellen Namespace passt!
spec:
  partitions: 1
  replicas: 1
  config:
    retention.ms: "604800000"
    cleanup.policy: "delete

```

Diese Konfiguration wenden wir jetzt mit dem `kubectl apply -n kafka -f [filename]` Befehl an, was ebenso funktioniert hat:

![](https://uploads.mfellner.com/dzrGVUwiFdda.png)

Für den nächsten Schritt benötigen wir die IP-Adresse und den Port eines Kafka Pods:

```shell
$ kubectl -n kafka describe po strimzi-cluster-operator-76b947897f-2ln9x | grep -i IP:
# Nicht vergessen, dass unser Cluster einen anderen Namen hat als das im Tutorial!
```

Die Ausgabe ist die folgende:

![](https://uploads.mfellner.com/QpnA81d5iP1H.png)

Also haben wir jetzt die folgende IP-Adresse:
- `10.244.3.4`

Als nächstes holen wir uns einmal die Ports:

```shell
$ kubectl -n kafka get po strimzi-cluster-operator-76b947897f-2ln9x -o yaml | grep -iA1 containerport
```

Die Ausgabe ist die folgende:

![](https://uploads.mfellner.com/px46wHZTIiwP.png)

Also bekommen wir hier als Port den Port `8080`, welcher als `http` Port gekennzeichnet ist. Dies ist merkwürdig, da er 1. nicht mit den Ports im Tutorial übereinstimmt und 2. dieser Port nicht erreichbar ist.

Die Ports, welche laut Tutorial vorzufinden wären, sind die folgenden:

```
    - containerPort: 29092  
      name: tcp-internal  
--  
    - containerPort: 29093  
      name: tcp-controller
```

Mit der folgenden Definition:

>- `Tcp-internal 29092`: This is the port used when you act as a consumer wanting to process the incoming messages to the Kafka cluster.  
  >  
>- `Tcp-controller 29093`: This is the port used when a producer, such as MinIO, wants to send messages to the Kafka cluster.

Dies ist merkwürdig, da hier etwas nicht stimmen kann.

### 2.4 MinIO installieren

Wir werden MinIO in seinem eigenen namespace im Kubernetes Cluster installieren.

Als erstes fetchen wir MinIO vom Github repo *[9]*:

```shell
$ git clone https://github.com/minio/operator.git
```
Dann wenden wir als nächstes die Ressourcen, welche für die Installation von MinIO benötigt werden, mittels `kubectl` an:

```shell
$ kubectl apply -k operator/resources
$ kubectl apply -k operator/examples/kustomization/tenant-lite
```

Beide Befehle haben hier funktioniert und geben uns die folgende Ausgabe:

![](https://uploads.mfellner.com/iqPD9BX67pZ2.png)

Als nächstes verifizieren wir, dass MinIO wirklich gestartet wurde. In diesem Fall können wir uns den Port der MinIO Konsole direkt holen:

```shell
$ kubectl -n tenant-lite get svc | grep -i console
```

![](https://uploads.mfellner.com/6uC2ov2OOPII.png)

In diesem Fall gibt es ebenso eine leichte Veränderung im Vergleich zum Tutorial: im Tutorial trägt die Konsole den Namen `storage-lite-console`, wobei diese bei uns `myminio-console` heißt.

Der nächste Schritt wäre, dass wir eine Weiterleitung des MinIO Konsolen-Ports über Kubernetes zu unserem Host-System:

```shell
$ kubectl -n tenant-lite port-forward svc/myminio-console 39443:9443 #Achtung! Andere Name als im Tutorial
```

Dies funktioniert auch wunderbar:

![](https://uploads.mfellner.com/Lxyx6SKpagH6.png)

Wenn wir nun im Browser zu `https://localhost:39443` gehen, sehen wir die folgende Weboberfläche:

![](https://uploads.mfellner.com/GNGGOA1ypRIw.png)

Hier können wir uns mit den folgenden Zugangsdaten einloggen:
- Benutzername: `minio`
- Passwort: `minio123` 

Hier können wir die MinIO Instanz dann ausgiebig per Interface verwalten.

### 2.5 MinIO producer konfigurieren

Nun möchten wir diejenige Komponente konfigurieren, welche Daten in unser topic sendet - den Producer.

Wir erstellen uns hierfür einfach ein Ubuntu-Pod, um eine saubere Trennung der Arbeitsumgebungen zu schaffen. Ebenso erspart uns dies die Port-Weiterleitung, welche dann ständig von Nöten wäre.

Dafür legen wir das folgene Config-File an:

```yml
apiVersion: v1  
kind: Pod  
metadata:  
  name: ubuntu  
  labels:  
    app: ubuntu  
spec:  
  containers:  
  - image: ubuntu  
    command:  
      - "sleep"  
      - "604800"  
    imagePullPolicy: IfNotPresent  
    name: ubuntu  
  restartPolicy: Always
```

Diesen Pod starten wir dann mittels `kubectl apply -f [filename].

Um nun Konsolenzugriff in diesen Pod zu erlangen, führen wir den folgenden Befehl aus:

```shell
$ kubectl exec -it ubuntu -- /bin/bash
```

Nun machen wir ein reguläres update und installieren uns den minIO Client:

```shell
$ apt-get update
$ apt-get -y install wget
$ wget https://dl.min.io/client/mc/release/linux-amd64/mc
$ chmod +x mc
$ mv mc /usr/local/bin/
```

Ob die Installation erfolgreich war, kann durch den folgenden Befehl überprüft werden:

```shell
$ mc --version
```

![](https://uploads.mfellner.com/8kURzwCoe4Zw.png)

Als nächstes müssen wir den `mc admin` so konfigurieren, dass er auch unser erstelles MinIO Cluster verwendet:

- `mc alias set <alias_name> <minio_tenant_url> <minio_username> <minio_password>`

In unserem Fall wäre das:

```shell
$ mc alias set myminio https://minio.tenant-lite.svc.cluster.local minio minio123
```

Dies bringt uns die folgende Ausgabe:

![](https://uploads.mfellner.com/Cu5kylDyKEdu.png)


Um zu überprüfen, ob die Konfiguration so funktioniert, können wir den folgenden Befehl eingeben:

```shell
$ mc admin info myminio
```

![](https://uploads.mfellner.com/Dxk8Yqn2yiOO.png)

Dieser Output deckt sich mit den Angaben, welche im Tutorial herrschen.

Als nächstes müssen wir noch die Kafka Konfiguration in MinIO durch den `mc admin` Befehl vornehmen, dafür müssen wir den folgenden Befehl anpassen:

```shell

$ mc admin config set myminio \  
notify_kafka:1 \  
brokers="10.244.1.5:29093" \  
topic="my-topic" \  
tls_skip_verify="off" \  
queue_dir="" \  
queue_limit="0" \  
sasl="off" \  
sasl_password="" \  
sasl_username="" \  
tls_client_auth="0" \  
tls="off" \  
client_tls_cert="" \  
client_tls_key="" \  
version="" --insecure
```

Diese Konfiguration sieht gut aus, JEDOCH haben wir das folgende Problem (welches vorher schon angesprochen wurde): hier wird die IP-Adresse, sowie ein Port erfragt. Dieser Port muss entweder `29092` ODER `209093` sein. In diesem Fall möchte das Tutorial den `29093` (TCP-Controller) verwenden.

Wir haben jedoch nur den Port `8080` bekommen. Dies ist sehr merkwürdig, damit bekommen wir nämlich nur den folgenden Fehler;

```shell
$ mc admin config set myminio \  
notify_kafka:1 \  
brokers="10.244.3.4:8080" \  
topic="my-topic" \  
tls_skip_verify="off" \  
queue_dir="" \  
queue_limit="0" \  
sasl="off" \  
sasl_password="" \  
sasl_username="" \  
tls_client_auth="0" \  
tls="off" \  
client_tls_cert="" \  
client_tls_key="" \  
version="" --insecure
```

Fehler:

![](https://uploads.mfellner.com/UgeazlH2xN7e.png)

Als Text:

```
mc: <ERROR> Unable to set 'notify_kafka:1 brokers=10.244.3.4:8080 topic=my-topic tls_skip_verify=off queue_dir= queue_limit=0 sasl=off sasl_password= sasl_username= tls_client_auth=0 tls=off client_tls_cert= client_tls_key= version=' to server: error (1:kafka): kafka: client has run out of available brokers to talk to: read tcp 10.244.1.8:37758->10.244.3.4:8080: i/o timeout.
```

Also kann dieser Port wohl nicht für die Konfiguration verwendet werden.

Leider habe ich keine direkte Lösung für dieses Problem gefunden. Da ich jedoch durchaus ein Interesse an der funktionierenden Übung habe, werde ich mich demnächst genauer damit beschäftigen.

## Quellen

*[1]*: https://kafka.apache.org/; 24.01.2025
*[2]*: https://min.io/; 24.01.2025
*[3]*: https://kind.sigs.k8s.io/; 24.01.2025
*[4]*: https://www.openlogic.com/blog/using-kafka-zookeeper; 24.01.2025
*[5]*: https://docs.otterize.com/features/kafka/tutorials/k8s-kafka-mtls-cert-manager; 24.01.2025
*[6]*: https://helm.sh/docs/intro/install/?ref=blog.min.io; 24.01.2025
*[7]*: https://helm.sh/docs/intro/install/?ref=blog.min.io; 24.01.2025
*[8]*: https://kafka.apache.org/intro; 24.01.2025
*[9]*: https://github.com/minio/operator.git; 24.01.2025
