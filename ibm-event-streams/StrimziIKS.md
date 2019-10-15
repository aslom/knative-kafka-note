# Install Strimzi

Follow
https://strimzi.io/quickstarts/minikube/


But for cluster creation use ephemeral strimzi:

```
kubectl apply -f https://raw.githubusercontent.com/strimzi/strimzi-kafka-operator/0.13.0/examples/kafka/kafka-ephemeral.yaml -n kafka
```

(persistent fails currently in strange ways [1])

End with
`kafka.kafka.strimzi.io/my-cluster created`



Verify

```
$ k -n kafka get pods
NAME                                          READY   STATUS    RESTARTS   AGE
my-cluster-entity-operator-67c5c49f76-vg4d4   3/3     Running   0          14m
my-cluster-kafka-0                            2/2     Running   0          15m
my-cluster-kafka-1                            2/2     Running   0          15m
my-cluster-kafka-2                            2/2     Running   0          15m
my-cluster-zookeeper-0                        2/2     Running   0          16m
my-cluster-zookeeper-1                        2/2     Running   0          16m
my-cluster-zookeeper-2                        2/2     Running   0          16m
strimzi-cluster-operator-646cdc566b-px9c2     1/1     Running   0          18m
```

[1] persistent Strimiz not working ...

in logs


```
k -n kafka logs my-cluster-zookeeper-0 zookeeper
Detected Zookeeper ID 1
mkdir: cannot create directory '/var/lib/zookeeper/data': Permission denied
/opt/kafka/zookeeper_run.sh: line 25: /var/lib/zookeeper/data/myid: No such file or directory
...
2019-05-14 19:55:38,032 ERROR Unexpected exception, exiting abnormally (org.apache.zookeeper.server.ZooKeeperServerMain) [main]
java.io.IOException: Unable to create data directory /var/lib/zookeeper/data/version-2
	at org.apache.zookeeper.server.persistence.FileTxnSnapLog.<init>(FileTxnSnapLog.java:87)
	at org.apache.zookeeper.server.ZooKeeperServerMain.runFromConfig(ZooKeeperServerMain.java:112)
	at org.apache.zookeeper.server.ZooKeeperServerMain.initializeAndRun(ZooKeeperServerMain.java:89)
	at org.apache.zookeeper.server.ZooKeeperServerMain.main(ZooKeeperServerMain.java:55)
	at org.apache.zookeeper.server.quorum.QuorumPeerMain.initializeAndRun(QuorumPeerMain.java:119)
	at org.apache.zookeeper.server.quorum.QuorumPeerMain.main(QuorumPeerMain.java:81)
```

Persistent Strimiz uses PVC:

```
$ k -n kafka describe pod my-cluster-zookeeper-0
...
  data:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  data-my-cluster-zookeeper-0
    ReadOnly:   false
...
Events:
  Type     Reason            Age                     From                     Message
  ----     ------            ----                    ----                     -------
  Warning  FailedScheduling  3m56s (x13 over 4m43s)  default-scheduler        pod has unbound immediate PersistentVolumeClaims (repeated 2 times)
```

PVC shows RWO but it is somehow not writable by Strimzi ...


```
(base) aslom@m ~ $ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                               STORAGECLASS       REASON   AGE
pvc-aae67e54-7681-11e9-8509-1ed58aace119   100Gi      RWO            Delete           Bound    kafka/data-my-cluster-zookeeper-0   ibmc-file-bronze            3m13s
```



# Create Strimzi based event source



Creating topic by running kafka producer:
(see https://strimzi.io/quickstarts/minikube/ for details)

```
kubectl -n kafka run kafka-producer -ti --image=strimzi/kafka:0.13.0-kafka-2.3.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --broker-list my-cluster-kafka-bootstrap:9092 --topic my-topic
```

And to receive them in a different terminal you can run:

```
kubectl -n kafka run kafka-consumer -ti --image=strimzi/kafka:0.13.0-kafka-2.3.
```


Make sure event-display is deployed:
```
$ k apply -f https://github.com/knative/eventing-contrib/releases/download/v0.9.0/event-display.yaml
service.serving.knative.dev/event-display created
```



Copy and edit event-source.yaml following instructions -  for Strimiz setup described above that should work:


```
$ cat kafka-source.yaml
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: KafkaSource
metadata:
  name: kafka-source
spec:
  consumerGroup: knative-group
  bootstrapServers: my-cluster-kafka-bootstrap.kafka:9092 #note the .kafka in URL for namespace
  topics: my-topic
  sink:
    apiVersion: serving.knative.dev/v1alpha1
    kind: Service
    name: event-display
```

and create event source:

```
$ k apply -f kafka-source.yaml
kafkasource.sources.eventing.knative.dev/kafka-source created
```

check logs kafka source: created:

```
$ k get pods
NAME                                  READY   STATUS    RESTARTS   AGE
kafka-source-fzttw-6ff7d4896b-zrf6p   1/1     Running   0          68s
$ k logs kafka-source-fzttw-6ff7d4896b-zrf6p
{"level":"info","ts":"2019-09-23T23:51:19.959Z","caller":"receive_adapter/main.go:110","msg":"Starting Apache Kafka Receive Adapter...","BootstrapServers":"my-cluster-kafka-bootstrap.kafka:9092","Topics":"my-topic","ConsumerGroup":"knative-group","SinkURI":"http://event-display.default.svc.cluster.local","SASL":false,"TLS":false}
{"level":"info","ts":"2019-09-23T23:51:19.959Z","caller":"adapter/adapter.go:122","msg":"Starting with config: ","BootstrapServers":"my-cluster-kafka-bootstrap.kafka:9092","Topics":"my-topic","ConsumerGroup":"knative-group","SinkURI":"http://event-display.default.svc.cluster.local","Name":"kafka-source","Namespace":"default","SASL":false,"TLS":false}
{"level":"info","ts":"2019-09-23T23:51:23.088Z","caller":"kafka/consumer_handler.go:65","msg":"Starting Consumer Group Handler, topic: my-topic"}
````


If kafka-source pod is not created in default namespace Status/Conditions in:

```
$ k describe kafkasource.sources.eventing.knative.dev/kafka-source
```

And check kafka controller logs , for example:

```
$ k logs kafka-controller-manager-0 -n knative-sources
...
{"level":"info","ts":"2019-07-24T00:24:43.394Z","caller":"reconciler/kafkasource.go:98","msg":"Reconciling new source: ","knative.dev/controller":"kafka-controller","request":"default/kafka-source","source":{"kind":"KafkaSource","apiVersion":"sources.eventing.knative.dev/v1alpha1","metadata":{"name":"kafka-source","namespace":"default","selfLink":"/apis/sources.eventing.knative.dev/v1alpha1/namespaces/default/kafkasources/kafka-source","uid":"694a2826-ada9-11e9-9bff-ce023c9439ca","resourceVersion":"169235","generation":2,"creationTimestamp":"2019-07-24T00:24:33Z","annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"sources.eventing.knative.dev/v1alpha1\",\"kind\":\"KafkaSource\",\"metadata\":{\"annotations\":{},\"name\":\"kafka-source\",\"namespace\":\"default\"},\"spec\":{\"bootstrapServers\":\"my-cluster-kafka-bootstrap.kafka:9092\",\"consumerGroup\":\"knative-group\",\"sink\":{\"apiVersion\":\"serving.knative.dev/v1alpha1\",\"kind\":\"Service\",\"name\":\"event-display\"},\"topics\":\"knative-demo-topic\"}}\n"}},"spec":{"bootstrapServers":"my-cluster-kafka-bootstrap.kafka:9092","topics":"knative-demo-topic","consumerGroup":"knative-group","net":{"sasl":{"user":{},"password":{}},"tls":{"cert":{},"key":{},"caCert":{}}},"sink":{"kind":"Service","name":"event-display","apiVersion":"serving.knative.dev/v1alpha1"},"resources":{"requests":{},"limits":{}}},"status":{"conditions":[{"type":"Deployed","status":"Unknown","lastTransitionTime":"2019-07-24T00:24:33Z"},{"type":"Ready","status":"False","lastTransitionTime":"2019-07-24T00:24:33Z","reason":"NotFound"},{"type":"SinkProvided","status":"False","lastTransitionTime":"2019-07-24T00:24:33Z","reason":"NotFound"}]}}}
{"level":"warn","ts":"2019-07-24T00:24:43.406Z","caller":"sdk/reconciler.go:76","msg":"Failed to reconcile &TypeMeta{Kind:,APIVersion:,}: sink \"default/event-display\" (serving.knative.dev/v1alpha1, Kind=Service) does not contain address","knative.dev/controller":"kafka-controller","request":"default/kafka-source"}
```

To test that event source works use two windows - one to send event and another to see it.


In one window:

```
kubectl -n kafka run kafka-producer -ti --image=strimzi/kafka:0.11.3-kafka-2.1.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --broker-list my-cluster-kafka-bootstrap.kafka:9092 --topic knative-demo-topic
If you don't see a command prompt, try pressing enter.
>{"msg": "This is a test2!"}
```

in other window verify event was received:

```
(base) aslom@m knative $ kubectl get pods
NAME                                              READY   STATUS     RESTARTS   AGE
event-display-jv2hj-deployment-7b85d7bbbf-vwtc6   0/3     Init:0/1   0          4s
kafka-source-5lsl8-d4c5f6885-lgvgh                2/2     Running    1          5m20s
kafka-source-strimzi-92rpc-694ff8fd-m5zzk         2/2     Running    1          30h
(base) aslom@m knative $ kubectl get pods
NAME                                              READY   STATUS    RESTARTS   AGE
event-display-jv2hj-deployment-7b85d7bbbf-vwtc6   2/3     Running   0          9s
kafka-source-5lsl8-d4c5f6885-lgvgh                2/2     Running   1          5m25s
kafka-source-strimzi-92rpc-694ff8fd-m5zzk         2/2     Running   1          30h

(base) aslom@m knative $ k logs event-display-jv2hj-deployment-7b85d7bbbf-vwtc6 user-container
☁️  cloudevents.Event
Validation: valid
Context Attributes,
  specversion: 0.2
  type: dev.knative.kafka.event
  source: /apis/v1/namespaces/default/kafkasources/kafka-source#knative-demo-topic
  id: partition:2/offset:0
  time: 2019-05-16T03:23:28.94Z
  contenttype: application/json
Extensions,
  key:
Data,
  {
    "msg": "This is a test2!"
  }
  ```
