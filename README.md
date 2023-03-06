# Autoscaler Demos

## General Setup

For simplicity we use [Minikube](https://minikube.sigs.k8s.io/docs/start/) as
Kubernetes distribution, but also provide hints how to use/install all tools on
full-blown k8s clusters.

Basic CPU and Memory metrics are provided by metrics-server. For external
metrics we will use prometheus as an example. See respective sections for
details and troubleshooting.

### Load generator image

We use [Resource Consumer](https://github.com/kubernetes/kubernetes/tree/d52ecd5f70cdf5f13f919bab56cf08cd556a2e26/test/images/resource-consumer)
to quickly and precisely set CPU and Memory usage of our test pods.

Examples:

```bash
curl --data "millicores=100&durationSec=60" $(minikube service autoscaler-demo --url)/ConsumeCPU
curl --data "megabytes=128&durationSec=60" $(minikube service autoscaler-demo --url)/ConsumeMem
```

Note: There is a minimum CPU load with this image. For my laptop, it's ~20mCPU,
so be sure to check and choose sufficiently high values for testing to prevent
misleading scaling results!

### Minikube Setup

Install [Minikube](https://minikube.sigs.k8s.io/docs/start/) on your system,
start it and enable the
[Metrics Server](https://kubernetes.io/docs/tutorials/hello-minikube/#enable-addons).

```bash
minikube start
minikube addons enable metrics-server
minikube addons enable dashboard
```

Change the default `metric-resolution` from `60s` to something like `10s`:

```bash
kubectl edit deployments.apps -n kube-system metrics-server
```

Check if metrics server is running properly:

```bash
kubectl get apiservices v1beta1.metrics.k8s.io -o yaml
kubectl get apiservices v1beta1.metrics.k8s.io
kubectl top node
kubectl -n kube-system top pod
```

### Minikube Workarounds

Unfortunately, docker-based k8s setups often lead to issues with monitoring
tools like metrics-server or prometheus/grafana.

Some issues I came across and found a workaround:

1.  Some versions of metrics server just won't work with Minikube even when
    enabling it as addon. This is a
    [known issue](https://github.com/kubernetes/minikube/issues/13969) and can
    be worked-around with

    ```bash
      minikube start --extra-config=kubelet.housekeeping-interval=10s
    ```

2.  When using Grafana K8s dashboards, change runtime to `CRI-O` because cAdvisor
    with docker runtime is not exporting `image` and `container` tags, thus
    breaking basically all relevant graphs (see
    [L1](https://github.com/k3s-io/k3s/issues/473),
    [L2](https://github.com/rancher/rancher/issues/38934),
    [L3](https://github.com/prometheus-community/helm-charts/issues/1938),
    [L4](https://github.com/prometheus-community/helm-charts/issues/2690)).

    ```bash
    minikube start --container-runtime=cri-o
    ```

3.  When using [keda](https://keda.sh/), downgrade k8s versionbecause of issue [
    [L1](https://github.com/kedacore/keda/issues/4008),
    [L2](https://github.com/kubernetes-sigs/custom-metrics-apiserver/issues/146)]

    ```bash
    minikube start --kubernetes-version=v1.24.10
    ```

    and use a matching `kubectl` version. Although errors seem to be harmless
    and can also simply be ignored.

### Metrics Server (general)

- Minikube

  ```bash
  minikube addons enable metrics-server
  ```

- Manual deployment

  ```bash
  kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
  ```

- [Official Helm Chart](https://artifacthub.io/packages/helm/metrics-server/metrics-server)
- [Bitnami Helm Chart](https://artifacthub.io/packages/helm/bitnami/metrics-server)

### Prometheus Stack

We use the shiny new operator-based cloud-native deployment option for the
entire "Prometheus Stack", i.e. Prometheus + Operator, Grafana, default
Dashboards and Rules, etc.

Helm chart:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```

```bash
kubectl expose service -n monitoring kube-prometheus-stack-grafana --type=NodePort --target-port=3000 --name=grafana-np
kubectl expose service -n monitoring kube-prometheus-stack-prometheus --type=NodePort --target-port=9090 --name=prometheus-np
```

```bash
minikube service -n monitoring grafana-np
minikube service -n monitoring prometheus-np
```

```bash
kubectl get secret -n monitoring kube-prometheus-stack-grafana -o jsonpath="{.data.admin-user}" | base64 --decode ; echo
```

Watch for example CPU load with the following PromQL queries:

```
rate(container_cpu_usage_seconds_total{pod=~"autoscaler-.*"}[5m])
rate(container_cpu_cfs_throttled_periods_total{pod=~"autoscaler-.*"}[5m])
```

## Horizontal Pod Autoscaler (HPA)

The HPA api is built-in, so there's nothing to install.

NOTE: Currently HPAs are not visible in Kubernetes Dashboard
(see [here](https://github.com/kubernetes/dashboard/issues/7607))

### Example Calculation

HPA scales pod replicas $N$ based on some metric $m$ according to

```math
N_\text{new} = \left\lceil N_\text{current}
\, \frac{\bar m_\text{current}}{m_\text{desired}} \right\rceil \,.
```

For a given desired `averageUtilization` of 50 for the CPU metric, a request of
200m and a synthetic consumption of 150m this means:

- current utilization: 150 / 200 = 75%
- desired utilization: 50%
- scaling factor: ceil(75/50) = 2

Similarly, a CPU load of 201m would lead to a scaling factor of 3 (because of
the ceil function).

### Basic Metrics

Let's first try a CPU-based autoscaling relying purely on metrics server data.

1. Deploy or enable Metrics Server and make sure it's ready
2. Deploy the [autoscaler-demo deployment](deployment.yaml) together with its
   [horizontal pod autoscaler](hpa.yaml).

   ```bash
   kubectl apply -f deployment.yml
   kubectl apply -f hpa.yml
   ```

3. Monitor resource usage with

   ```bash
   kubectl get events
   kubectl get hpa -w
   watch -n1 kubectl get pods
   watch -n1 kubectl top pod
   ```

4. Increase CPU load in container

   ```bash
   curl --data "millicores=120&durationSec=60" $(minikube service autoscaler-demo --url)/ConsumeCPU
   ```

5. Watch how HPA increases replica count, waits the cool-down period and scales
   down again

### External Metrics (testing)

For this to work we need to install an additional aggregation layer which
provids the `custom.metrics.k8s.io` and/or `external.metrics.k8s.io` APIs.
An external metrics provider could be e.g.
[Datadog](https://www.datadoghq.com/blog/autoscale-kubernetes-datadog/) or
Prometheus via [Prometheus Adapter](https://github.com/kubernetes-sigs/prometheus-adapter).
However, setting this up is kinda PITA most of the time. So, lazy people as we
are, let's go for the easy option and simply use [KEDA](https://keda.sh).

With KEDA we can directly put any metric query which we want to base the
autoscaling on into the manifest of a new custom resource called `ScaledObject`.
KEDA supports all relevant monitoring tools as "scalers", in particular
Prometheus and Datadog. Find a short intro
[here](https://sysdig.com/blog/kubernetes-hpa-prometheus/).

Note, that Metrics Server is not required at all in this case!

1. Deploy Prometheus stack as explained above
2. Deploy KEDA via Helm chart

   ```bash
   helm install keda kedacore/keda --namespace keda --create-namespace
   ```

3. Deploy our prepared [ScaledObject](hpa_keda.yaml), which in turn will take
   care of the HPA part

   ```
   kubectl apply -f deployment.yml
   kubectl apply -f hpa_keda.yml
   ```

4. Increase the CPU load

   ```bash
   curl --data "millicores=600&durationSec=60" $(minikube service autoscaler-demo --url)/ConsumeCPU
   ```

5. Watch KEDA doing its job

   ```bash
   kubectl get scaledobjects.keda.sh -w
   kubectl get hpa -w
   kubectl get pods -w
   kubectl get events -w
   ```

KEDA does not use utilization but AverageValue by default. Read up on the
differences [here](https://keda.sh/docs/2.9/concepts/scaling-deployments/#triggers).

---

Note: For simplicity, the scaling in this demo was based on an already
available (default) CPU throttling metric. A far better external metric would
be something like "http requests per second" which in turn requires metrics
from an ingress controller like Nginx or Traefik or from a service mesh like
Istio.

### External Metrics (production)

In this example we will deploy a [simple webserver](https://github.com/traefik/whoami)
with `/api` and `/metrics` endpoints. [Traefik](https://traefik.io/) will act
as ingress controller and send metrics to our Prometheus stack.

Disclaimer: I've shamelessly stolen parts of this section from
[this nice tutorial](https://www.linkedin.com/pulse/k8saws-eks-hpa-based-traefik-metrics-prometheus-adapter-ihar-vauchok/).

#### Prometheus Setup

In contrast to older Prometheus deployments, the operator version does not rely
on the `prometheus.io/scrape` annotation, but uses the much more powerful
service and pod monitor concepts.

So first we need to patch the Prometheus (operator) setup such that it accepts
`serviceMonitor` objects without a specific release-tag
(see [here](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack#prometheusioscrape)).

```bash
# optional: check what will change
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring \
--set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false \
--set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false

# required
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring \
--set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false \
--set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false
```

#### Install Traefik

As usual, we go for the Helm Chart deployment, but this time we need to adapt
some values as well.

Fortunately, the upstream devs have already
[added support](https://github.com/traefik/traefik-helm-chart/issues/626) for
the Prometheus operator / ServiceMonitor deployment type. So all we need to do
is add a few more config options via [values file](traefik_values.yaml).

```bash
helm repo add traefik https://traefik.github.io/charts
helm install traefik traefik/traefik -n traefik --create-namespace -f traefik_values.yaml
```

Dashboard and Prometheus metrics are already enabled by default with the
current Helm Chart version. So just for fun, let's have a look at the Traefik
dashboard.

```bash
kubectl expose deployment -n traefik traefik --type=NodePort --target-port=9000 --name=traefik-np
```

```bash
firefox $(minikube service -n traefik traefik-np --url | head -n1)/dashboard/
```

---

Fun fact: the Traefik Helm Chart also has a [built-in HPA template](https://github.com/traefik/traefik-helm-chart/blob/v21.1.0/traefik/templates/hpa.yaml)
that only needs to be [activated](https://github.com/traefik/traefik-helm-chart/blob/v21.1.0/traefik/values.yaml#L721) ;-).

#### Deploy Webserver

Deployment, service and Traefik ingress is all covered in
[whoami.yaml](whoami.yaml).

Let's check the `/api` endpoint after deploying the file.

```bash
kubectl apply -f whoami.yaml
```

```bash
curl $(minikube service -n traefik traefik --url | head -n1)/api | jq
```

The metrics endpoint is only available internally. Let's also check it's output
(we don't need it anymore later, so only do a quick `port-forward` here...)

```bash
kubectl port-forward -n traefik deployments/traefik 9100:9100
curl localhost:9100/metrics | grep whoami
```

This should show Traefik metrics specific to the `whoami` ingress.

Since the `serviceMonitor` takes care of the Prometheus linkage, we should also
be able to see some Traefik http request logs there already.

```bash
minikube service -n monitoring prometheus-np
```

```
traefik_entrypoint_requests_total
rate(traefik_service_requests_total{service=~"default-whoami-.*"}[2m])
```

While monitoring, generate some request load on the webserver

```bash
while true; do curl -s $(minikube service -n traefik traefik --url | head -n1)/api > /dev/null; sleep 0.1; done
```

#### Autoscaler

This basically works the same as in the previous chapter. Just deploy the
respective `ScaledObject` from [traefik_keda.yaml](traefik_keda.yaml) and have
a look at the following outputs:

```bash
kubectl get scaledobjects.keda.sh -w
kubectl get hpa -w
kubectl get pods -w
kubectl get events -w
```

## Vertical Pod Autoscaler (VPA)

### Installation

[Official VPA resources](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)

- based on shell scripts :/
- [Upstream Helm Chart discussion](https://github.com/kubernetes/autoscaler/issues/3068) ->
  Summary: they won't provide an official one

Inofficial VPA Helm Charts:

- [cowboysysop](https://github.com/cowboysysop/charts/tree/master/charts/vertical-pod-autoscaler)
- [Fairwinds](https://github.com/FairwindsOps/charts/tree/master/stable/vpa)

Another great tool related to VPA is
[Goldilocks](https://github.com/FairwindsOps/goldilocks) by Fairwinds which
provides a nice dashboard for VPA recommendations.

It seems the Helm Chart by cowboysysop is the most used one, although using any
should work.

```bash
helm repo add cowboysysop https://cowboysysop.github.io/charts/
helm install cowboy-vpa cowboysysop/vertical-pod-autoscaler -n vpa --create-namespace
```

For quick resource inspection, the
[kube-capacity](https://github.com/robscott/kube-capacity) kubectl-plugin is
great. It can be installed via `krew`.

```bash
yay krew
export PATH=$PATH:$HOME/.krew/bin
kubectl krew install resource-capacity
```

---

VPA can also utilize Prometheus as historical data query backend. Read up on
how to configure [here](https://github.com/kubernetes/autoscaler/issues/1551).
In this tutorial we'll stick with Metrics Server data.

### Manual Deployment

1. Deploy or enable Metrics Server and make sure it's ready
2. Deploy the [autoscaler-demo deployment](deployment.yaml) together with its
   [vertical pod autoscaler](vpa.yaml) with `updateMode: "Off"`

   ```
   kubectl apply -f deployment.yml
   kubectl apply -f vpa.yml
   ```

3. Monitor resource usage with

   ```bash
   kubectl get events
   kubectl get vpa -w
   watch -n1 kubectl get pods
   watch -n1 kubectl top pod
   ```

   and check verbose recommendations with

   ```bash
   kubectl get vpa vpa-demo -o jsonpath='{.status..containerRecommendations}' | jq
   ```

4. Increase CPU load in container

   ```bash
   curl --data "millicores=120&durationSec=60" $(minikube service autoscaler-demo --url)/ConsumeCPU
   ```

5. Watch how VPA changes the CPU recommendation (which is equal to `target` in
   the verbose output)

   and if available

   ```bash
   watch -n1 kubectl resource-capacity --pods -n default
   ```

6. Switch to `updateMode: "Auto"` and increase replica count to 2 to allow pod
   rotation.

7. Increase memory usage this time

   ```bash
   curl --data "megabytes=250&durationSec=240" $(minikube service autoscaler-demo --url)/ConsumeMem
   ```

8. Wait (quite some time) until VPA adapts the recommendation and eventually
   rotates pods.

Note: By default, VPA will not scale down to 0 pods unless (in our case) the
Helm Chart value `admissionController.pdb.minAvailable` is set to 0.

### Goldilocks

Make sure to remove VPAs from the last section before continuing.

Install via [Helm Chart](https://artifacthub.io/packages/helm/fairwinds-stable/goldilocks)

```bash
helm repo add fairwinds-stable https://charts.fairwinds.com/stable
helm install goldilocks fairwinds-stable/goldilocks --namespace goldilocks --create-namespace
```

Open dashboard

```bash
kubectl expose service -n goldilocks goldilocks-dashboard --type=NodePort --target-port=8080 --name=goldilocks-np
minikube service -n goldilocks goldilocks-np
```

Label `default` namespace to enable Goldilocks on its workloads

```bash
kubectl label ns default goldilocks.fairwinds.com/enabled=true
kubectl get ns --show-labels
```

Increase CPU/Memory load and monitor the recommendations in the dashboard.

Remove label after testing

```bash
kubectl label ns default goldilocks.fairwinds.com/enabled-
```
