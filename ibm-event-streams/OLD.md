

# Configure backbone

...

===


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











====

DELETE

# Upgrade Knative Eventing

https://github.com/knative/eventing/blob/master/DEVELOPMENT.md

I have used export KO_DOCKER_REPO=docker.io/aslom - any docker registry should work


```
ko apply -f config/
```



# Configure Kafka event source (controller)

Instructions
https://github.com/knative/eventing-sources/tree/master/contrib/kafka/samples



```
ko apply -f contrib/kafka/config
```


```
$ k logs kafka-controller-manager-0 -n knative-sources
2019/05/14 20:52:10 Registering Components.
2019/05/14 20:52:10 Setting up Controller.
2019/05/14 20:52:10 Adding the Apache Kafka Source controller.
2019/05/14 20:52:10 Starting Apache Kafka controller.
```