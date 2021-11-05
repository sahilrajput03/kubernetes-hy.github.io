---
path: '/part-3/3-gke-features'
title: 'GKE features'
hidden: false
---

<text-box variant='learningObjectives' name='Learning Objectives'>

After this section you can

- Compare DBaaS to a self hosted solution

- Have the pods in the cluster autoscale

- Have the cluster itself autoscale

</text-box>


## Volumes again ##

Now we arrive at an intersection. We can either start using a Database as a Service (DBaaS) such as the Google Cloud SQL in our case or just use the PersistentVolumeClaims with our own Postgres images and let the Google Kubernetes Engine take care of storage via PersistentVolumes for us.

Both solutions are widely used.

<exercise name='Exercise 3.06: DBaaS vs DIY'>

  Do a pros/cons comparison of the solutions in terms of meaningful differences. This includes **at least** the required work and cost to initialize as well as maintain. Backup methods and their ease of usage should be considered as well.

  Set the list into the README of the project.

</exercise>

<exercise name='Exercise 3.07: Commitment'>

  Use Google Cloud SQL or Postgres with PersistentVolumeClaims in your project. Give a reasoning to which you chose in the README. There are no non-valid reasons, an excellent would be "because it sounded easier".

</exercise>

## Scaling ##

Scaling can be either horizontal scaling or vertical scaling. Vertical scaling is the act of increasing resources available to a pod or a node. Horizontal scaling is what we most often mean when talking about scaling, increasing the number of pods or nodes. We'll focus on horizontal scaling.

### Scaling pods ###

There are multiple reasons for wanting to scale an application. The most common reason is that the number of requests an application receives exceeds the number of requests that can be processed. Limitations are often either the amount of requests that a framework is intended to handle or the actual CPU or RAM.

I've prepared an application that uses up CPU resources here: `jakousa/dwk-app7:e11a700350aede132b62d3b5fd63c05d6b976394`. The application accepts a query parameter to increase the time until freeing CPU via "?fibos=25", you should use values between 15 and 30.

**deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpushredder-dep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cpushredder
  template:
    metadata:
      labels:
        app: cpushredder
    spec:
      containers:
        - name: cpushredder
          image: jakousa/dwk-app7:e11a700350aede132b62d3b5fd63c05d6b976394
          resources:
            limits:
              cpu: "150m"
              memory: "100Mi"
```

Note that finally we have set the resource limits for a Deployment as well. The suffix of the CPU limit "m" stands for "thousandth of a core". Thus `150m` equals 15% of a single CPU core (`150/1000=0,15`).

The service looks completely familiar by now.

**service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cpushredder-svc
spec:
  type: LoadBalancer
  selector:
    app: cpushredder
  ports:
    - port: 80
      protocol: TCP
      targetPort: 3001
```

Next we have _HorizontalPodAutoscaler_. This is an exciting new Resource for us to work with.

**horizontalpodautoscaler.yaml**

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: cpushredder-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpushredder-dep
  minReplicas: 1
  maxReplicas: 6
  targetCPUUtilizationPercentage: 50
```

HorizontalPodAutoscaler automatically scales pods horizontally. The yaml here defines what is the target Deployment, how many minimum replicas and what is the maximum replica count. The target CPU Utilization is defined as well. If the CPU utilization exceeds the target then an additional replica is created until the max number of replicas.

```console
$ kubectl top pod -l app=cpushredder
  NAME                               CPU(cores)   MEMORY(bytes)
  cpushredder-dep-85f5b578d7-nb5rs   1m           20Mi

$ kubectl get hpa
  NAME              REFERENCE                    TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
  cpushredder-hpa   Deployment/cpushredder-dep   0%/50%    1         6         1          62s

$ kubectl get svc
  NAME              TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
  cpushredder-svc   LoadBalancer   10.31.254.209   35.228.149.206   80:32577/TCP   94s
```

After a few requests to the external IP here the application will start using more CPU. Note that if you request above the limit the pod will be taken down.

```console
$ kubectl logs -f cpushredder-dep-85f5b578d7-nb5rs
  Started in port 3001
  Received a request
  started fibo with 25
  Received a request
  started fibo with 25
  Received a request
  started fibo with 25
  Fibonacci 25: 121393
  Closed
  Fibonacci 25: 121393
  Closed
  Fibonacci 25: 121393
  Closed
```

After a few requests we will see the *HorizontalPodAutoscaler* create a new replica as the CPU utilization rises. As the resources are fluctuating, sometimes very greatly due to increased resource usage on start or exit, the *HPA* will by default wait 5 minutes between downscaling attempts. If your application has multiple replicas even at 0%/50% just wait. If the wait time is set to a value that's too short for stable statistics of the resource usage the replica count may start "thrashing".

Figuring out autoscaling with HorizontalPodAutoscalers can be one of the more challening tasks. Choosing which resources to look at and when to scale

<exercise name='Exercise 3.08: Project v1.5'>

  Set sensible resource limits for the project. The exact values are not important. Test what works.

</exercise>

<exercise name='Exercise 3.09: Sensible processes'>

  Set sensible resource limits for the "Ping-pong" and "Log output" applications. The exact values are not important. Test what works.

</exercise>

### Scaling nodes ###

Scaling nodes is a supported feature in GKE. Via the cluster autoscaling feature we can use the right amount of nodes needed.

```console
$ gcloud container clusters update dwk-cluster --zone=europe-north1-b --enable-autoscaling --min-nodes=1 --max-nodes=5
  Updated [https://container.googleapis.com/v1/projects/dwk-gke/zones/europe-north1-b/clusters/dwk-cluster].
  To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/europe-north1-b/dwk-cluster?project=dwk-gke
```

For a more robust cluster see examples on creation here: <https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-autoscaler>

<img src="../img/gke_scaling.png">

Cluster autoscaling may disrupt pods by moving them around as the number of nodes increases or decreases. To solve possible issues with this the resource [PodDisruptionBudget](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#how-disruption-budgets-work) with which the requirements for a pod can be defined via two of the fields: _minAvailable_ and _maxUnavailable_.

**poddisruptionbudget.yaml**

```yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: example-app-pdb
spec:
  maxUnavailable: 50%
  selector:
    matchLabels:
      app: example-app
```

This would ensure that no more than half of the pods can be unavailable at. The Kubernetes documentation states "The budget can only protect against voluntary evictions, not all causes of unavailability."

_Side note:_ Kubernetes also offers the possibility to limit resources per namespace. This can prevent apps in the development namespace from consuming too many resources. Google has created a nice video that explains the possibilities of the `ResourceQuota` object.

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/xjpHggHKm78" frameborder="0" allow="accelerometer; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

<exercise name='Exercise 3.10: Project v1.6'>

  GKE includes monitoring systems already so we can just enable the monitoring.

  Read documentation for Kubernetes Engine Monitoring [here](https://cloud.google.com/monitoring/kubernetes-engine) and setup logging for the project in GKE.

  You can optionally include Prometheus as well.

  Submit a picture of the logs when a new todo is created.

</exercise>

<quiz id="cdd5ff16-6883-403a-8eb7-2cab7b8aa364"></quiz>