Create or use existing k8s cluster (make sure cluster is single zone - otherwise problems with read only mounted dirs innside pods: https://cloud.ibm.com/docs/containers?topic=containers-cs_troubleshoot_storage
https://cloud.ibm.com/docs/containers?topic=containers-file_storage 


# Configure event streams

Create topic names "my-topic"

Get credentials

```

{
  "api_key": "2h7Fn2MJC-QMDI5...",
  "apikey": "2h7Fn2MJC-QMDI5...",
  "iam_apikey_description": "Auto-generated for key 8ea435c8-d73e-46e7-8187-f40c57748a51",
  "iam_apikey_name": "Service credentials-1",
  "iam_role_crn": "crn:v1:bluemix:public:iam::::serviceRole:Manager",
  "iam_serviceid_crn": "crn:v1:bluemix:public:iam-identity::a/e3a470d882f9e5b9a59f4c98b6cb2b40::serviceid:ServiceId-c642d368-7ddb-4880-a659-24da5a3249cc",
  "instance_id": "e28d1e5f-9ee1-426a-95b8-6e26cd9b6346",
  "kafka_admin_url": "https://33rc6rhn7fw8k0q4.svc01.us-south.eventstreams.cloud.ibm.com",
  "kafka_brokers_sasl": [
    "broker-0-33rc6rhn7fw8k0q4.kafka.svc01.us-south.eventstreams.cloud.ibm.com:9093",
    "broker-1-33rc6rhn7fw8k0q4.kafka.svc01.us-south.eventstreams.cloud.ibm.com:9093",
    "broker-5-33rc6rhn7fw8k0q4.kafka.svc01.us-south.eventstreams.cloud.ibm.com:9093",
    "broker-4-33rc6rhn7fw8k0q4.kafka.svc01.us-south.eventstreams.cloud.ibm.com:9093",
    "broker-2-33rc6rhn7fw8k0q4.kafka.svc01.us-south.eventstreams.cloud.ibm.com:9093",
    "broker-3-33rc6rhn7fw8k0q4.kafka.svc01.us-south.eventstreams.cloud.ibm.com:9093"
  ],
  "kafka_http_url": "https://33rc6rhn7fw8k0q4.svc01.us-south.eventstreams.cloud.ibm.com",
  "password": "2h7Fn2MJC-QMDI5...",
  "user": "token"
}
```

#  Install Strimzi

Follow
https://strimzi.io/quickstarts/minikube/


But for cluster creation  use ephemeral strimzi 

```
kubectl apply -f https://raw.githubusercontent.com/strimzi/strimzi-kafka-operator/0.13.0/examples/kafka/kafka-ephemeral.yaml -n kafka
```

(persistent fails currently in starange ways [1])

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


# Install Knative v0.9


Install Kafka if not installed:

```
k apply -f https://github.com/knative/eventing-contrib/releases/download/v0.9.0/kafka-channel.yaml
k apply -f https://github.com/knative/eventing-contrib/releases/download/v0.9.0/kafka-source.yaml
```


OLD:
Use IBM Knative Addon  `0.7.1`     
https://cloud.ibm.com/docs/containers?topic=containers-serverless-apps-knative

NOTE requirement of `Kubernetes 1.13 AND 1.13 workers` for cluster:

After installation check kafka controller is working:

```
$ k get pods -n knative-sources
NAME                         READY   STATUS    RESTARTS   AGE
kafka-controller-manager-0   1/1     Running   0          23s
$ k logs kafka-controller-manager-0 -n knative-sources
2019/09/23 23:27:29 Registering Components.
2019/09/23 23:27:29 Setting up Controller.
2019/09/23 23:27:29 Adding the Apache Kafka Source controller.
2019/09/23 23:27:29 Starting Apache Kafka controller.
```


VERY OLD: (deploying latest version from github should work too but did not have time to test it)
Follow "Manually installing Knative on IKS"
https://knative.dev/docs/install/knative-with-iks/

If you see errors like"

```
unable to recognize "https://github.com/knative/eventing/releases/download/v0.5.0/release.yaml": no matches for kind "ClusterChannelProvisioner" in version "eventing.knative.dev/v1alpha1"
unable to recognize "https://github.com/knative/eventing/releases/download/v0.5.0/release.yaml": no matches for kind "ClusterChannelProvisioner" in version "eventing.knative.dev/v1alpha1"
```

then run command again and should be fine.


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


# Configure Knative event source that uses Event Streams


Create topic in Event Stream:

```
./bin/kafka-topics.sh --create --bootstrap-server broker-0-33rc6rhn7fw8k0q4.kafka.svc01.us-south.eventstreams.cloud.ibm.com:9093 --command-config ../ibmes1.properties --topic my-topic --replication-factor 3 --partitions 1
```

```
$ ./bin/kafka-topics.sh --list --bootstrap-server broker-0-33rc6rhn7fw8k0q4.kafka.svc01.us-south.eventstreams.cloud.ibm.com:9093 --command-config ../ibmes1.properties
my-topic
```

Verify topic is working:

```
./bin/kafka-console-producer.sh --broker-list broker-0-33rc6rhn7fw8k0q4.kafka.svc01.us-south.eventstreams.cloud.ibm.com:9093 --producer.config ../ibmes1.properties --topic my-topic
```

```
./bin/kafka-console-consumer.sh --bootstrap-server broker-0-33rc6rhn7fw8k0q4.kafka.svc01.us-south.eventstreams.cloud.ibm.com:9093 --consumer.config ../ibmes1.properties --topic my-topic --from-beginning
```

Create secret with user and password from JSON


```
$ cat secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: es1secret
type: Opaque
stringData:
  user: token
  password: 2h7Fn2MJC-QMDI5x...
`
```

```
k apply -f secret.yaml
k get secret es1secret -o yaml
```

create ibmes1.yaml file that references secret created above:

```
$ cat ibmes1.yaml
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: KafkaSource
metadata:
  name: event-source-es1
spec:
  consumerGroup: "groupA"
  bootstrapServers:  "broker-0-33rc6rhn7fw8k0q4.kafka.svc01.us-south.eventstreams.cloud.ibm.com:9093"
  topics: "my-topic"
  net:
    sasl:
      enable: true
      user:
        secretKeyRef:
          name: es1secret
          key: user
      password:
        secretKeyRef:
          name: es1secret
          key: password
    tls:
      enable: true
  sink:
    apiVersion: serving.knative.dev/v1alpha1
    kind: Service
    name: event-display
```
 

You may need to disable istio mesh injection - it is not needed for event source and in past blocked access to event streams (as IBM cloud is external)

```
kubectl label namespace default istio-injection=disabled --overwrite=true
```



```
$ k apply -f ibmes1.yaml
kafkasource.sources.eventing.knative.dev/event-source-es1 created
```



TODO logs and event-display debugging (persistent instead of on0deman version?)



```
(base) aslom@m knative.dev $ k logs event-source-es1-lc4hf-77445c5446-7n47f
{"level":"info","ts":"2019-09-24T00:36:50.209Z","caller":"receive_adapter/main.go:110","msg":"Starting Apache Kafka Receive Adapter...","BootstrapServers":"broker-0-33rc6rhn7fw8k0q4.kafka.svc01.us-south.eventstreams.cloud.ibm.com:9093","Topics":"my-topic","ConsumerGroup":"groupA","SinkURI":"http://event-display.default.svc.cluster.local","SASL":true,"TLS":true}
{"level":"info","ts":"2019-09-24T00:36:50.209Z","caller":"adapter/adapter.go:122","msg":"Starting with config: ","BootstrapServers":"broker-0-33rc6rhn7fw8k0q4.kafka.svc01.us-south.eventstreams.cloud.ibm.com:9093","Topics":"my-topic","ConsumerGroup":"groupA","SinkURI":"http://event-display.default.svc.cluster.local","Name":"event-source-es1","Namespace":"default","SASL":true,"TLS":true}
panic: kafka: client has run out of available brokers to talk to (Is your cluster reachable?)

goroutine 1 [running]:
knative.dev/eventing-contrib/kafka/source/pkg/adapter.(*Adapter).Start(0xc0005be000, 0xf2b9e0, 0xc0005b6030, 0xc0005c0000, 0x0, 0x0)
	/home/prow/go/src/knative.dev/eventing-contrib/kafka/source/pkg/adapter/adapter.go:163 +0x1009
main.main()
	/home/prow/go/src/knative.dev/eventing-contrib/kafka/source/cmd/receive_adapter/main.go:117 +0xd35
$
```


Check Status/Conditions:

```
k get kafkasource.sources.eventing.knative.dev/event-source-es1 -o yaml
```


```
$ k get pods
NAME                                        READY   STATUS    RESTARTS   AGE
event-source-es1-gm7hq-7846bc564b-hkkfc     2/2     Running   1          3m55s
kafka-source-5lsl8-d4c5f6885-lgvgh          2/2     Running   1          21m
kafka-source-strimzi-92rpc-694ff8fd-m5zzk   2/2     Running   1          30h

(base) aslom@m knative $ k logs event-source-es1-gm7hq-7846bc564b-hkkfc receive-adapter
{"level":"info","ts":"2019-05-16T03:35:47.703Z","caller":"receive_adapter/main.go:110","msg":"Starting Apache Kafka Receive Adapter...","BootstrapServers":"kafka01-prod02.messagehub.services.us-south.bluemix.net:9093","Topics":"knative-demo-topic","ConsumerGroup":"groupA","SinkURI":"http://event-display.default.svc.cluster.local/","SASL":true,"TLS":true}
{"level":"info","ts":"2019-05-16T03:35:47.703Z","caller":"adapter/adapter.go:105","msg":"Starting with config: ","BootstrapServers":"kafka01-prod02.messagehub.services.us-south.bluemix.net:9093","Topics":"knative-demo-topic","ConsumerGroup":"groupA","SinkURI":"http://event-display.default.svc.cluster.local/","Name":"event-source-es1","Namespace":"default","SASL":true,"TLS":true}
```


# Send test message to event streams

I have use Apache Kafka Quickstart for kafka_2.11-2.0.0 with properties file:

```
$ cat ibmes1.properties
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="T7f3xuwnKG0Q5V71" password="Xwup9VRp9di...";
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
ssl.protocol=TLSv1.2
ssl.enabled.protocols=TLSv1.2
ssl.endpoint.identification.algorithm=HTTPS
```

In one window:

```
$ ./bin/kafka-console-producer.sh --broker-list kafka01-prod02.messagehub.services.us-south.bluemix.net:9093 --producer.config ../ibmes1.properties --topic knative-demo-topic
>{"msg": "This is a test3!"}
```

in another window verify event was received:


```
(base) aslom@m eventing (master) $ k get pods
NAME                                              READY   STATUS    RESTARTS   AGE
event-display-jv2hj-deployment-7b85d7bbbf-c6qm4   2/2     Running   0          8s
event-source-es1-p4fnj-786746df99-sj75l           1/1     Running   0          19m
kafka-source-5lsl8-d4c5f6885-lgvgh                2/2     Running   1          51m
kafka-source-strimzi-92rpc-694ff8fd-m5zzk         2/2     Running   1          31h

(base) aslom@m eventing (master) $ k logs event-display-jv2hj-deployment-7b85d7bbbf-c6qm4 user-container
☁️  cloudevents.Event
Validation: valid
Context Attributes,
  specversion: 0.2
  type: dev.knative.kafka.event
  source: /apis/v1/namespaces/default/kafkasources/event-source-es1#knative-demo-topic
  id: partition:0/offset:0
  time: 2019-05-16T04:09:16.083Z
  contenttype: application/json
Extensions,
  key:
Data,
  {
    "msg": "This is a test3!"
  }
```



