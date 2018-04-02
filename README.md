# Bus Demo

Git: https://github.com/benh/bus-demo

## Setup

### Third Party Services

```
dcos package install jenkins --yes
dcos package install cassandra --yes --options prod/cassandra-options.json
dcos package install kafka --yes --options prod/kafka-options.json --package-version 2.0.3-0.11.0
dcos package install spark --yes --options prod/spark-options.json
```

Or install from the GUI. Remember to also install the service CLIs if you install from the GUI.

### EdgeLB

```
dcos package repo add --index=0 edgelb-aws \
  https://edge-lb-infinity-artifacts.s3.amazonaws.com/autodelete7d/master/edgelb/stub-universe-edgelb.json
dcos package repo add --index=0 edgelb-pool-aws \
  https://edge-lb-infinity-artifacts.s3.amazonaws.com/autodelete7d/master/edgelb-pool/stub-universe-edgelb-pool.json

dcos package install dcos-enterprise-cli --yes --cli
dcos security org service-accounts keypair edgelb-private-key.pem edgelb-public-key.pem
dcos security org service-accounts create -p edgelb-public-key.pem -d "edgelb service account" edgelb-principal
dcos security org groups add_user superusers edgelb-principal
dcos security secrets create-sa-secret --strict edgelb-private-key.pem edgelb-principal edgelb-secret
rm -f edgelb-private-key.pem edgelb-public-key.pem

dcos package install --yes --options edgelb-options.json edgelb
dcos edgelb create prod/edgelb-pool-prod.yml
```

Find the public IP of EdgeLB:

```
dcos node ssh --master-proxy --private-ip "$(dcos edgelb endpoints bus-demo-prod | tail -1 | awk '{print $3}')" \
  /opt/mesosphere/bin/detect_ip_public
```

Optional: add an entry to /etc/hosts.

### App Components

```
dcos job add prod/cassandra-schema.json
dcos job run initialize-prod-cassandra-schema-job

dcos spark --name=/prod/spark run --submit-args='--driver-cores 0.1 --driver-memory 1024M --total-executor-cores 4 --class de.nierbeck.floating.data.stream.spark.KafkaToCassandraSparkApp https://oss.sonatype.org/content/repositories/snapshots/de/nierbeck/floating/data/spark-digest_2.11/0.2.1-SNAPSHOT/spark-digest_2.11-0.2.1-SNAPSHOT-assembly.jar METRO-Vehicles node-0-server.prodcassandra.autoip.dcos.thisdcos.directory:9042 broker.prodkafka.l4lb.thisdcos.directory:9092'

dcos marathon app add prod/ingest/marathon.json
dcos marathon app add prod/ui/marathon.json
```

## Live Demo

### Kubernetes

1. Install Kubernetes from UI.
1. While Kubernetes is installing, install /dev/cassandra, /dev/kafka, and /dev/spark.
1. Check Kubernetes in UI, then go and set up the CLI.

```
dcos package install kubernetes --yes

dcos kubernetes kubeconfig

kubectl apply -f schema-pod.yaml
```

### Cassandra

```
kubectl exec -it $(kubectl get pods --selector=app=schema-pod --output=jsonpath={.items..metadata.name}) /bin/bash

cqlsh node-0-server.devcassandra.autoip.dcos.thisdcos.directory
describe keyspace streaming;

/opt/bus-demo/import_data.sh node-0-server.devcassandra.autoip.dcos.thisdcos.directory

cqlsh node-0-server.devcassandra.autoip.dcos.thisdcos.directory
describe keyspace streaming;
```

### Spark

Check everything else is installed and then start the Spark job.

```
dcos spark --name=/dev/spark run --submit-args='--driver-cores 0.1 --driver-memory 1024M --total-executor-cores 4 --class de.nierbeck.floating.data.stream.spark.KafkaToCassandraSparkApp https://oss.sonatype.org/content/repositories/snapshots/de/nierbeck/floating/data/spark-digest_2.11/0.2.1-SNAPSHOT/spark-digest_2.11-0.2.1-SNAPSHOT-assembly.jar METRO-Vehicles node-0-server.devcassandra.autoip.dcos.thisdcos.directory:9042 broker.devkafka.l4lb.thisdcos.directory:9092'
```

### Jenkins

Create pipeline:

* Freestyle
* GitHub project > Project url https://github.com/guenter/bus-demo
* Post-build Actions > Marathon Deployment > Marathon URL http://marathon.mesos:8080

Kick of first build.

git merge dev
git push

# Go back to UI, wait for build.
# Go to UI, wait for /dev/bus-demo/ui to start.


# Go to browser, look at buses.

### Update Kafka

1. Show the Kafka version in the UI.
1. Start the update

```
dcos kafka --name=/prod/kafka update package-versions
dcos kafka --name=/prod/kafka update start --package-version=2.0.4-1.0.0
```

----- IF YOU WANT TO SHOW KAFKA TOPIC STREAM -----

dcos kafka --name=/dev/kafka topic list

dcos task exec kafka-toolbox /bin/bash -c "export JAVA_HOME=/opt/jdk1.8.0_144/jre/; /opt/kafka_2.12-0.11.0.0/bin/kafka-console-consumer.sh --zookeeper master.mesos:2181/dcos-service-prod__kafka --topic METRO-Vehicles --property print.timestamp=true --property print.value=false"


## Cleanup

```
dcos edgelb delete bus-demo-prod
dcos package uninstall edgelb --yes
dcos security secrets delete edgelb-secret
dcos security org service-accounts delete edgelb-principal
dcos marathon group remove /dcos-edgelb
dcos package repo remove edgelb-aws edgelb-pool-aws

dcos package uninstall cassandra --yes --app-id /prod/cassandra
dcos package uninstall kafka --yes --app-id /prod/kafka
dcos package uninstall spark --yes --app-id /prod/spark

dcos marathon app remove /prod/bus-demo/ui
dcos marathon app remove /prod/bus-demo/ingest
dcos marathon group remove /prod

dcos job remove initialize-prod-cassandra-schema-job
```
