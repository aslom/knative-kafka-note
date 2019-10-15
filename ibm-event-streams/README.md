

# Configure Event Streams

Follow [instructions](https://cloud.ibm.com/docs/services/EventStreams?topic=eventstreams-connecting#connect_enterprise_external_console
) to get JSON configutation such as:




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



# Create topic names "my-topic" in Event Streams.


Create topic in Event Stream:

Using kafka_2.12-2.3.0 (download form Apache webiste)

```
./bin/kafka-topics.sh --create --bootstrap-server broker-0-33rc6rhn7fw8k0q4.kafka.svc01.us-south.eventstreams.cloud.ibm.com:9093 --command-config ../ibmes1.properties --topic my-topic --replication-factor 3 --partitions 1
```

and check that topic is created

```
./bin/kafka-topics.sh --list --bootstrap-server broker-0-33rc6rhn7fw8k0q4.kafka.svc01.us-south.eventstreams.cloud.ibm.com:9093 --command-config ../ibmes1.properties 
```

output

```
my-topic
```


# Verify access to Event Streams works from you laptop

```
./bin/kafka-console-producer.sh --broker-list broker-0-33rc6rhn7fw8k0q4.kafka.svc01.us-south.eventstreams.cloud.ibm.com:9093 --producer.config ../ibmes1.properties --topic my-topic
```

```
./bin/kafka-console-consumer.sh --bootstrap-server broker-0-33rc6rhn7fw8k0q4.kafka.svc01.us-south.eventstreams.cloud.ibm.com:9093 --consumer.config ../ibmes1.properties --topic my-topic --from-beginning
```


# Verify access to Event Streams from your cluster

TBW

# Run test event sink

Using event-dsiplay and adapting getting started instrucitons to setup test environment: https://knative.dev/docs/eventing/getting-started/



```bash
kubectl apply --filename - << END
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-display
spec:
  replicas: 1
  selector:
    matchLabels: &labels
      app: hello-display
  template:
    metadata:
      labels: *labels
    spec:
      containers:
        - name: event-display
          # Source code: https://github.com/knative/eventing-contrib/blob/release-0.6/cmd/event_display/main.go
          image: gcr.io/knative-releases/github.com/knative/eventing-sources/cmd/event_display@sha256:37ace92b63fc516ad4c8331b6b3b2d84e4ab2d8ba898e387c0b6f68f0e3081c4

---

# Service pointing at the previous Deployment. This will be the target for event
# consumption.
  kind: Service
  apiVersion: v1
  metadata:
    name: hello-display
  spec:
    selector:
      app: hello-display
    ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
END
```


```
kubectl get deployments hello-display
```

expected output:

```
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
hello-display   1/1     1            1           5s
```


# Send test event directly to event sink


```bash
kubectl apply --filename - << END
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: curl
  name: curl
spec:
  containers:
    # This could be any image that we can SSH into and has curl.
  - image: radial/busyboxplus:curl
    imagePullPolicy: IfNotPresent
    name: curl
    resources: {}
    stdin: true
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    tty: true
END
```

Send events

```
kubectl attach curl -it
```

Inside

```
curl -v "hello-display.default.svc.cluster.local" \
  -X POST \
  -H "Ce-Id: say-hello" \
  -H "Ce-Specversion: 0.2" \
  -H "Ce-Type: greeting" \
  -H "Ce-Source: not-sendoff" \
  -H "Content-Type: application/json" \
  -d '{"msg":"Hello Knative!"}'

```

expected output:

```
> POST / HTTP/1.1
> User-Agent: curl/7.35.0
> Host: hello-display.default.svc.cluster.local
> Accept: */*
> Ce-Id: say-hello
> Ce-Specversion: 0.2
> Ce-Type: greeting
> Ce-Source: not-sendoff
> Content-Type: application/json
> Content-Length: 24
>
< HTTP/1.1 202 Accepted
< Content-Length: 0
< Date: Tue, 15 Oct 2019 03:00:16 GMT
<
```

And check that it was received:


```
kubectl logs -l app=hello-display --tail=100
```

Expected output

```
☁️  cloudevents.Event
Validation: valid
Context Attributes,
  specversion: 0.2
  type: greeting
  source: not-sendoff
  id: say-hello
  contenttype: application/json
Data,
  {
    "msg": "Hello Knative!"
  }
```



# Configure Event Stream Kubernetes secret

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


# Install Knative addon

Follow [instructions](https://cloud.ibm.com/docs/containers?topic=containers-serverless-apps-knative#knative-setup)

TODO VERIFY: Install Knative Kafka source if not installed:

```
k apply -f https://github.com/knative/eventing-contrib/releases/download/v0.9.0/kafka-source.yaml
```


# Create knative source for Event Streams topic

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


# Send test message to Event Streams using Kafka client

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



