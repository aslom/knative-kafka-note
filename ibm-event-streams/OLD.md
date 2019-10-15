
Create or use existing k8s cluster (make sure cluster is single zone - otherwise problems with read only mounted dirs innside pods: https://cloud.ibm.com/docs/containers?topic=containers-cs_troubleshoot_storage
https://cloud.ibm.com/docs/containers?topic=containers-file_storage 




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



# Configure backbone

...

===











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