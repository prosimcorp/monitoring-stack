# Monitoring Stack

## Important

Before diving deeper in the specific documentation of this repository, you must know that is part of an entire flow
composed by several repositories. In the following diagram you will have a better insight.

```text
                                                                +----------------------+
                                                         +----->|  3. Tooling Stack    |
                                                         |      +----------------------+
                                                         |
+-----------------------+       +--------------------+   |      +----------------------+
| 1. Automated K8S      +------>| 2. FluxCD Template +---+----->|  4. Monitoring Stack |
+-----------------------+       +--------------------+   |      +----------------------+
                                                         |
                                                         |      +----------------------+
                                                         +----->|  5. Applications     |
                                                                +----------------------+
```

**[1] Automated K8S:** This stage consists of a Kubernetes cluster which is created and automated using GitOps approach.  
We cover this stage with several repositories ready for different cloud providers:
  * [Automated EKS](https://github.com/prosimcorp/automated-eks)
  * [Automated GKE](https://github.com/prosimcorp/automated-gke)
  * [Automated DO](https://github.com/prosimcorp/automated-do)

> This stage can be covered for different cloud providers or with different technologies. Don't hesitate to code them 
> if needed

**[2] [FluxCD Template](https://github.com/prosimcorp/fluxcd-template)**

**[3] [Tooling Stack](https://github.com/prosimcorp/tooling-stack)**

**[4] [Monitoring Stack](https://github.com/prosimcorp/monitoring-stack)**

## Description

Monitoring stack to deploy things needed to watch metrics, logs and traces and see them in a graphical dashboard.
Reliable and automated.

## Motivation

As SREs, we need reliability on the operators we use, and some tools must be configured to work together.
When a new cluster is created, it needs a whole tooling stack deployed.

One of the problem we detected in te past is about SREs finding some bug on their clusters' tooling. The procedure this
case would be to fix it and upgrade the cluster, then apply the changes to the rest of the clusters, spending a lot of 
time of different people to do the task, which is basically replicating the changes. Because of this, we decided to
provide a tooling stack, well tested and integrated where everyone can collaborate once and rise the improvements
easier.

## What you should expect

This stack will be made on top of well tested tools, such as Prometheus, Loki, FluentBit, Grafana and so on. This is
done this way due to monitoring must be as reliable as possible. 

It is expected to test new tools if the paradigm changes, but they will be tested in `develop` clusters first. 
Deploying the `develop` stack you can test all the stack with the latest changes and collaborate to improve it. When changes
become solid, they will be promoted to `production` deployable stack

## Some requirements here

> Following requirements are already covered by other projects crafted by us to bootstrap Kubernetes:
>
> - [Tooling Stack](https://github.com/prosimcorp/tooling-stack)

To deploy this system inside Kubernetes, some requirements must be satisfied.

- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) `v1.23+`
- [Helm](https://helm.sh/docs/intro/install/) `v3.9.0+`
- [Replika Operator](https://github.com/prosimcorp/replika) `v0.2.5+`
- [Reforma Operator](https://github.com/prosimcorp/reforma) `v0.1.1+`

### A very special ConfigMap 

A ConfigMap called `cluster-info`, deployed inside the namespace `kube-system`. This resource is used to customize
some deployments according to the cloud provider, the environment, etc.

For example, this repository relies on the information stored there to customize the messages sent by Alertmanager 
to Slack, adding a header with useful information about the cluster.
This information will help on debugging process of the problems, giving a better insight about the source.

Learning by example is funnier, so the content of the configmap is like the following:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-info
  namespace: kube-system
data:
  # The provider. Available values are: AWS, GCP, DO
  provider: AWS 
  # Project ID on GCP, account number on AWS, or project name on DO
  account: "111111111111" 
  region: eu-west-1
  name: your-kubernetes-cluster-name
```

### Several secrets must be created

> We recommend not to craft secrets this way, but automate its provisioning with tools like External Secrets, 
> with credentials coming from a secrets vault. To follow this recommendation, just fork the repository and include the 
> External Secrets CRs with the same `Secret` names for the targets

#### A secret for Grafana

To generate credentials for Grafana, just execute the following command against your cluster before deploying the stack.
This can be done on cluster creation time:

```console
kubectl create secret generic admin-credentials -n grafana \
    --from-literal username="your-username" \
    --from-literal password="your password"
```

#### A secret for Alertmanager

Notifications are sent to Slack using a webhook, to a channel called `#monitoring-stack-global-notifications`. This
mechanism needs a `Secret` resource to give the webhook URL.

To generate the secret, just execute the following command against your cluster before deploying the stack.
This can be done on cluster creation time:

```console
kubectl create secret generic alertmanager-global-notifications-slack -n kube-prometheus-stack \
    --from-literal apiURL="https://your-webhook-URL"
```

## How to access Grafana

The dashboard can be accessed using a tunnel as follows:

```console
kubectl port-forward service/grafana -n grafana 8080:80
```

Then, in your browser, you can access the dashboard just accessing [http://localhost:8080](http://localhost:8080)

#### The credentials for Grafana

Just look for a secret called `admin-credentials` inside the `grafana` namespace using kubectl as follows. You created
it on a previous step.

```console
kubectl get secret admin-credentials -n grafana --template={{.data.username}} | base64 -d

kubectl get secret admin-credentials -n grafana --template={{.data.password}} | base64 -d
```

## Some nodes must be tainted
Some tools must be as resilient as possible and monitoring tools are an example for that, which are deployed only on 
Kubernetes nodes specially tainted for them. This is done using tolerations and node affinity on deployment time as follows.

```yaml
# Manifest shorted for better focusing
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
          - key: node-type
            operator: In
            values:
              - monitoring
tolerations:
- key: monitoring
  value: dedicated
  operator: Equal
  effect: NoSchedule
```

Be sure that you have a node group with the taints already configured on your Kubernetes cluster resources to host these pods. 
If those nodes don't exist, Kubernetes will not be able to schedule the resources on any node, because of the conditions, 
and the deployment will never happen.

## How to deploy using Flux

SREs understand how difficult is to deploy everything, so we have done that task for you to make it easier.
On the root of this repository you can find a directory called `flux` with environments inside, i.e. `flux/production` or
`flux/develop`. All you need from now is pointing your Flux deployment to the stack you want to deploy, using a GitRepository
and Kustomization as follows:

```yaml
---
# Point git repository for Monitoring Stack
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: monitoring-stack
  namespace: flux-system
spec:
  interval: 20s
  ref:
    branch: master
    tag: v0.1.0
  secretRef:
    name: flux-system
  url: https://github.com/prosimcorp/monitoring-stack.git
---
# Deploy monitoring stack
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: monitoring-stack
  namespace: flux-system
spec:
  interval: 10m
  retryInterval: 1m
  path: ./flux/develop
  prune: true
  sourceRef:
    kind: GitRepository
    name: monitoring-stack
    namespace: flux-system
```

Pay special attention to the `spec.path` because there is where the desired stack will be defined, setting
the parameter to `./flux/develop` or to `./flux/production`

Anyway, as described previously, we did it for you in the templates, i.e. the template inside
[FluxCD Template](https://github.com/prosimcorp/fluxcd-template)

## Troubleshooting

#### _Why Kube Scheduler and Kube Controller Manager components are not monitored_

On managed Kubernetes, some components are launched on master nodes. Due to it, they are not reachable to be scrapped
by Prometheus.

For these reasons, we decided to disable them

---

#### _Kube Proxy targets seems to be down and always firing on EKS_

> Pay attention, this is happening only on EKS, not on GKE

Don't worry, little Timmy. This is because a parameter into Kube Proxy's configuration is set to be too secure on AWS.
It is needed only to execute the following commands:

```console
kubectl edit cm/kube-proxy-config -n kube-system
```

Then, adjust the `metricsBindAddress` parameter value from `127.0.0.1:10249` to `0.0.0.0:10249` as follows:

```yaml
# Change from
metricsBindAddress: 127.0.0.1:10249 ### <--- Too secure

# Change to
metricsBindAddress: 0.0.0.0:10249
```

Restart involved pods executing the following:

```console
kubectl rollout restart -n kube-system daemonset kube-proxy
```

When some minutes passed, your changes will be overwritten again by the EKS control plane. This is an expected
behaviour. We did not craft a Reforma's `Patch` due to you should not need to restart pods that much.

---

#### _Fluent Bit alarm firing about not processing logs in the last 15 minutes_

This happens when some pods of the DeamonSet is not running.

Usually happens when deploying this stack on EKS with busy nodes. The reason for this is related to the ENIs attached 
to the nodes are scheduling IPs and not releasing them when the load decreases, so new Pods can not get an available one. 
This ends with pods not being able to run in the nodes. 

This bad behaviour of EKS clusters can be mitigated launching more `NodeGroups` with the same capabilities, labels, etc; 
to increase the number of available IPs. Once some IPs are available, just rollout the `DaemonSet` for Fluent Bit

```console
kubectl rollout restart -n fluent-bit daemonset.apps/fluent-bit
```

---

## How to collaborate

1. Open an issue and discuss the problem to find together the best way to solve it
2. Fork the repository, create a branch and change inside everything you need
3. Launch a cluster pointing monitoring-stack to your repository with the changes
4. Open a Pull Request to merge your code. Don't be dirty with the commits (squash them), be clear with the commit message, etc
5. Cheers 🎉