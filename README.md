# Troubleshooting Kubernetes Applications

A talk at [Velocity NYC 2018](https://conferences.oreilly.com/velocity/vl-ny/public/schedule/detail/69892):

_When?_  &nbsp;&nbsp;&nbsp; Wednesday Oct 3, 1:30pm (40min) <br />
_Where?_ &nbsp;&nbsp; Beekman/Sutton North <br />

<a href="#preparation" target="_self">Preparation</a> | <a href="#intro" target="_self">Intro</a> | <a href="#poking-pods" target="_self">Poking Pods</a>
--- | --- | ---
<a href="#storage" target="_self">Storage</a> | <a href="#network" target="_self">Network</a> | <a href="#security" target="_self">Security</a>
<a href="#observability" target="_self">Observability</a> | <a href="#vaccination" target="_self">Vaccination</a> | <a href="#references" target="_self">References</a>


The slide deck is available [here](http://dev/null) and to demonstrate the different failures and how to fix them I'm using the following commands. Note that when you see a &#128196; icon it means a reference to the official Kubernetes [docs](https://kubernetes.io/docs/).

## Preparation

Before the talk, set up:

```
# create the namespace we'll be operating on:
kubectl create ns vnyc

# in different tmux pane keep an eye on the resources:
watch kubectl -n vnyc get all
```

## Intro

Using [00_intro.yaml](00_intro.yaml):

```
kubectl -n vnyc apply -f 00_intro.yaml

kubectl -n vnyc describe deploy/unhappy-camper

THEPOD=$(kubectl -n vnyc get po -l=app=whatever --output=jsonpath={.items[*].metadata.name})
kubectl -n vnyc describe po/$THEPOD
kubectl -n vnyc logs $THEPOD
kubectl -n vnyc exec -it $THEPOD -- sh

kubectl -n vnyc delete deploy/unhappy-camper
```

## Poking Pods

### Image issue

Using [01_pp_image.yaml](01_pp_image.yaml):

```
kubectl -n vnyc apply -f 01_pp_image.yaml
kubectl -n vnyc get pods
kubectl -n vnyc get events | grep confused | grep Error

kubectl -n vnyc patch deployment confused-imager  \
                --patch '{ "spec" : { "template" : { "spec" : { "containers" : [ { "name" : "something" , "image" : "mhausenblas/simpleservice:0.5.0" } ] } } } }' \

kubectl -n vnyc delete deploy/confused-imager
```

Relevant real-world examples on StackOverflow:

- [Kubernetes imagePullSecrets not working; getting “image not found”](https://stackoverflow.com/questions/32510310/kubernetes-imagepullsecrets-not-working-getting-image-not-found)
- [Trying to create a Kubernetes deployment but it shows 0 pods available](https://stackoverflow.com/questions/51139988/trying-to-create-a-kubernetes-deployment-but-it-shows-0-pods-available)

### Keeps crashing

Using [02_pp_oomer.yaml](02_pp_oomer.yaml) and [02_pp_oomer-fixed.yaml](02_pp_oomer-fixed.yaml):

```
kubectl -n vnyc apply -f 02_pp_oomer.yaml
kubectl -n vnyc exec -it $(kubectl -n vnyc get po -l=app=oomer --output=jsonpath={.items[*].metadata.name}) -c greedymuch -- cat /sys/fs/cgroup/memory/memory.limit_in_bytes /sys/fs/cgroup/memory/memory.usage_in_bytes

# wait > 5s
kubectl -n vnyc describe po $(kubectl -n vnyc get po -l=app=oomer --output=jsonpath={.items[*].metadata.name})

kubectl -n vnyc apply -f 02_pp_oomer-fixed.yaml
# wait > 20s
kubectl -n vnyc exec -it $(kubectl -n vnyc get po -l=app=oomer --output=jsonpath={.items[*].metadata.name}) -c greedymuch -- cat /sys/fs/cgroup/memory/memory.limit_in_bytes /sys/fs/cgroup/memory/memory.usage_in_bytes

kubectl -n vnyc delete deploy wegotan-oomer
```

Relevant real-world examples on StackOverflow: 

- [InfluxDB container dies over time, and can't restart](https://stackoverflow.com/questions/37877432/influxdb-container-dies-over-time-and-cant-restart)

### Something's wrong with the app

Using [03_pp_logs.yaml](03_pp_logs.yaml): 

```
kubectl -n vnyc apply -f 03_pp_logs.yaml

# nothing to see here:
kubectl -n vnyc describe deploy/hiccup

# but I see it in the logs:
kubectl -n vnyc logs --follow $(kubectl -n vnyc get po -l=app=hiccup --output=jsonpath={.items[*].metadata.name})

kubectl -n vnyc delete deploy hiccup
```

Relevant real-world examples on StackOverflow: 

- [My kubernetes pods keep crashing with “CrashLoopBackOff” but I can't find any log](https://stackoverflow.com/questions/41604499/my-kubernetes-pods-keep-crashing-with-crashloopbackoff-but-i-cant-find-any-lo)
- [Kubernetes Readiness probe failed error](https://stackoverflow.com/questions/48540929/kubernetes-readiness-probe-failed-error)

### Pod lifecycle

![pod lifecycle](img/pod-lifecycle-inline.png)
_Download [in original resolution](https://github.com/mhausenblas/troubleshooting-k8s-apps/raw/master/img/pod-lifecycle.png)._

References:

- [Debugging microservices - Squash vs. Telepresence](https://www.weave.works/blog/debugging-microservices-squash-vs-telepresence)
- [Debugging and Troubleshooting Microservices in Kubernetes with Ray Tsang (Google)](https://www.weave.works/blog/debugging-and-troubleshooting-microservices-in-kubernetes)
- [Troubleshooting Kubernetes Using Logs](https://blog.papertrailapp.com/troubleshoot-kubernetes-using-logs/)

## Storage

Using [04_storage-failedmount.yaml](04_storage-failedmount.yaml) and [04_storage-failedmount-fixed.yaml](04_storage-failedmount-fixed.yaml): 

```
kubectl -n vnyc apply -f 04_storage-failedmount.yaml

# has the data been written?
kubectl -n vnyc exec -it $(kubectl -n vnyc get po -l=app=wheresmyvolume --output=jsonpath={.items[*].metadata.name}) -c writer -- cat /tmp/out/data

# has the data been read in?
kubectl -n vnyc exec -it $(kubectl -n vnyc get po -l=app=wheresmyvolume --output=jsonpath={.items[*].metadata.name}) -c reader -- cat /tmp/in/data

kubectl -n vnyc describe po $(kubectl -n vnyc get po -l=app=wheresmyvolume --output=jsonpath={.items[*].metadata.name})

kubectl -n vnyc apply -f 04_storage-failedmount-fixed.yaml

kubectl -n vnyc delete deploy wheresmyvolume
```

Relevant real-world examples on StackOverflow: 

- [How to find out why mounting an emptyDir volume fails in Kubernetes?](https://stackoverflow.com/questions/51206154/how-to-find-out-why-mounting-an-emptydir-volume-fails-in-kubernetes)
- [Kubernetes NFS volume mount fail with exit status 32](https://stackoverflow.com/questions/34113569/kubernetes-nfs-volume-mount-fail-with-exit-status-32)

References:

- [Debugging Kubernetes PVCs](https://itnext.io/debugging-kubernetes-pvcs-a150f5efbe95) 

## Network

Using [05_network-wrongsel.yaml](05_network-wrongsel.yaml) and [05_network-wrongsel-fixed.yaml](05_network-wrongsel-fixed.yaml): 

```
kubectl -n vnyc run webserver --image nginx --port 80

kubectl -n vnyc apply -f 05_network-wrongsel.yaml 

kubectl -n vnyc run -it --rm debugpod --restart=Never --image=centos:7 -- curl webserver.vnyc

kubectl -n vnyc run -it --rm debugpod --restart=Never --image=centos:7 -- ping webserver.vnyc

kubectl -n vnyc run -it --rm debugpod --restart=Never --image=centos:7 -- ping $(kubectl -n vnyc get po -l=run=webserver --output=jsonpath={.items[*].status.podIP})

kubectl -n vnyc apply -f 05_network-wrongsel-fixed.yaml 

kubectl -n vnyc delete deploy webserver 
```

Other scenarios often found:

- See an error message that says something like `connection refused`? You could be hitting the `127.0.0.1` issue with the solution to make the app listen on `0.0.0.0` rather than on localhost. Further, see also some discussion [here](https://superuser.com/questions/949428/whats-the-difference-between-127-0-0-1-and-0-0-0-0).
- Missing firewall rules, from cluster-internal open ports to communication between clusters can cause all kinds of issues. It very much depends on the environment (AWS, Azure, GCP, on-premises, etc.) how exactly you go about it and most certainly is an infra admin task rather than an appops task.
- Taking a pod offline for debugging: on the pod, simply remove the relevant label(s) the service uses in its `selector` and that removes the pod from the pool of endpoints the service has to serve traffic to while leaving the pod running, ready for you to `kubectl exec -it` in.

Relevant real-world examples on StackOverflow: 

- [Connection Refused error when connecting to Kubernetes Redis Service](https://stackoverflow.com/questions/48597726/connection-refused-error-when-connecting-to-kubernetes-redis-service/) StackOverflow question—
- [“kubectl get pods” showing STATUS - ImagePullbackOff](https://stackoverflow.com/questions/51164795/kubectl-get-pods-showing-status-imagepullbackoff)
- [Service not exposing in kubernetes](https://stackoverflow.com/questions/51662015/service-not-exposing-in-kubernetes)
- [Kubernetes: Can not curl minikube pod](https://stackoverflow.com/questions/52289583/kubernetes-can-not-curl-minikube-pod/52289956)

References:

- [Debug Services](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/) &#128196;
- [Troubleshooting Kubernetes Networking Issues](https://gravitational.com/blog/troubleshooting-kubernetes-networking/)

## Security

```
kubectl -n vnyc create sa prober
kubectl -n vnyc run -it --rm probepod --serviceaccount=prober --restart=Never --image=centos:7 -- sh

# in the container; will result in an 403, b/c we don't have the permissions necessary:
export CURL_CA_BUNDLE=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
APISERVERTOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -H "Authorization: Bearer $APISERVERTOKEN"  https://kubernetes.default/api/v1/namespaces/vnyc/pods

# different tmux pane, verify if the SA actually is allowed to:
kubectl -n vnyc auth can-i list pods --as=system:serviceaccount:vnyc:prober

# … seems not to be the case, so give sufficient permissions:
kubectl create clusterrole podreader \
        --verb=get --verb=list \
        --resource=pods

kubectl -n vnyc create rolebinding allowpodprobes \
        --clusterrole=podreader \
        --serviceaccount=vnyc:prober \
        --namespace=vnyc
```

Relevant real-world examples on StackOverflow: 

- [Accessing Kubernetes API from pod fails although roles are configured is configured](https://stackoverflow.com/questions/52095161/accessing-kubernetes-api-from-pod-fails-although-roles-are-configured-is-configu)
- [How to deploy a deployment in another namespace in Kubernetes?](https://stackoverflow.com/questions/52297676/how-to-deploy-a-deployment-in-another-namespace-in-kubernetes/52298101#52298101)

References see [kubernetes-security.info](https://kubernetes-security.info/).

## Observability

From metrics to logs to tracing.

### Service ops in practice

Show [Linkerd 2.0 in action](https://medium.com/@mhausenblas/linkerd-2-0-service-ops-for-you-and-me-281cc5bd6424).

### Distributed tracing

Show [Jaeger 1.6 in action](https://www.jaegertracing.io/docs/1.6/getting-started/).

References:

- [Logs and Metrics](https://medium.com/@copyconstruct/logs-and-metrics-6d34d3026e38)
- [Evolution of Monitoring and Prometheus](https://www.slideshare.net/brianbrazil/evolution-of-monitoring-and-prometheus-dublin-2018)
- [The life of a span](https://medium.com/jaegertracing/the-life-of-a-span-ee508410200b)
- OpenShift Commons Briefing #82: [Distributed Tracing with Jaeger & Prometheus on Kubernetes](https://blog.openshift.com/openshift-commons-briefing-82-distributed-tracing-with-jaeger-prometheus-on-kubernetes/)

## Vaccination

Show chaoskube in action.

References:

- [Kubernetes: five steps to well-behaved apps](https://medium.com/@betz.mark/kubernetes-five-steps-to-well-behaved-apps-a7cbeb99471a)
- [Kubernetes Best Practices](https://medium.com/google-cloud/kubernetes-best-practices-8d5cd03446e2)
- [Developing on Kubernetes](https://kubernetes.io/blog/2018/05/01/developing-on-kubernetes/)
- [Kubernetes Application Operator Basics](https://blog.openshift.com/kubernetes-application-operator-basics/) 
- [chaoskube](https://github.com/linki/chaoskube), for example see [using chaoskube with OpenEBS](https://blog.openebs.io/chaos-engineering-on-openebs-7d4e0f995545)

## References

### General

- [Troubleshoot Applications](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-application/) &#128196;
- [Troubleshoot Clusters](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/) &#128196;
- A site dedicated to [Kubernetes Troubleshooting](https://kubernetes.feisky.xyz/en/troubleshooting/) 
- [Debug a Go Application in Kubernetes from IDE](https://itnext.io/debug-a-go-application-in-kubernetes-from-ide-c45ad26d8785)
- CrashLoopBackoff, Pending, FailedMount and Friends: Debugging Common Kubernetes Cluster (KubeCon NA 2017): [video](https://www.youtube.com/watch?v=7FOCG5kua1w) and [slide deck](https://afontofuseless.info/debugging-kubernetes-app-deploys-kc2017/)
- 10 Most Common Reasons Kubernetes Deployments Fail: [Part 1](https://kukulinski.com/10-most-common-reasons-kubernetes-deployments-fail-part-1/) and [Part 2](https://kukulinski.com/10-most-common-reasons-kubernetes-deployments-fail-part-1/)

### Language or platform specific

- [Debugging Microservices: How Google SREs Resolve Outages](https://www.infoq.com/presentations/google-debug-microservices)
- [Debugging Microservices: Lessons from Google, Facebook, Lyft](https://thenewstack.io/debugging-microservices-lessons-from-google-facebook-lyft/)
- [Troubleshooting Java applications on OpenShift](https://developers.redhat.com/blog/2017/08/16/troubleshooting-java-applications-on-openshift/)
- Google Kubernetes Engine [Troubleshooting](https://cloud.google.com/kubernetes-engine/docs/troubleshooting) docs
