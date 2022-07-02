---
# YAML header
ignore_macros: true
---

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
| 1. Terraform EKS/GKE  +------>| 2. Flux clusters   +---+----->|  4. Monitoring Stack |
+-----------------------+       +--------------------+   |      +----------------------+
                                                         |
                                                         |      +----------------------+
                                                         +----->|  5. Applications     |
                                                                +----------------------+
```

## Description
Monitoring stack to deploy things needed to watch metrics, logs and traces and see them in a graphical dashboard.
Reliable and automated.

## Motivation
We need reliability on monitoring, and some tools must be configured to work together. When a new cluster is created,
it needs a whole monitoring stack deployed. 

One of the problem we detected is about SREs finding some bug on its cluster's monitoring. The procedure this case would 
be to upgrade the template of the cluster and then apply the changes to all the clusters, spending a lot of time of 
different people to do the task, which is basically replicating the changes. Because of this, the SRE chapter decided to
provide a monitoring stack, well tested and integrated where all the members can collaborate once and rise the improvements
easier.

## What you should expect
This stack will be made on top of well tested tools, such as Prometheus, Loki, FluentBit, Grafana and so on. This is
done this way due to monitoring must be as reliable as possible. 

It is expected to test new tools if the paradigm changes, but they will be tested in `develop` clusters first. 
Deploying the `develop` stack you can test all the stack with the latest changes and collaborate to improve it. When changes
become solid, they will be promoted to `production` deployable stack

## Some requirements here

> All the following requirements are created inside Kubernetes either on cluster creation or on deployment the Tooling Stack

This project relies on several dependencies to work properly and be automatically configured.

#### External Secrets

External Secrets is needed to get some credentials from Vault to generate Kubernetes Secrets for several tools 
deployed on this stack, such as Grafana or Alertmanager.

> For more documentation, please go to the documentation of the
> [Tooling Stack](https://gitlab.infrastructure.s73cloud.com/Infrastructure/tooling-stack)

#### A very special ConfigMap 

A ConfigMap called `cluster-info`, deployed inside the namespace `kube-system`. This resource is used to customize
some deployments according to the cloud provider, the environment, etc.
For example, this repository relies on the information stored there to customize the messages sent by Alertmanager's,
for example to Slack, adding a header with useful information about the cluster.
This information will help on debugging process of the problems, giving a better insight about the source.

Learning by example is funnier, so the content of the configmap is like the following:

```yaml
apiVersion: v1
data:
  account: "111111111111"
  environment: develop
  name: dremel-develop
  provider: AWS
  region: eu-west-1
kind: ConfigMap
metadata:
  name: cluster-info
  namespace: kube-system
```

## How to access Grafana
Each cluster has its own monitoring stack deployed, so the subdomains for each Ingress needed are different between
clusters. Due to this and enforcing the security, we will not deploy an Ingress to access the dashboard.

The dashboard can be accessed using a tunnel as follows:

```console
kubectl port-forward service/grafana -n grafana 8080:80
```

Then, in your browser, you can access the dashboard just accessing [http://localhost:8080](http://localhost:8080)

#### The credentials for Grafana
These credentials are given to the cluster by **External Secrets** from **Vault**, and it can be found on `infrastructure`
section inside **Vault**.

Of course, to access Grafana, as operator you will need them. Just look for it inside Vault or look for a secret called 
`admin-credentials` inside the `grafana` namespace using kubectl as follows

```console
kubectl get secret admin-credentials -n grafana --template={{.data.username}} | base64 -d

kubectl get secret admin-credentials -n grafana --template={{.data.password}} | base64 -d
```

## Some nodes must be tainted
Some tools must be as resilient as possible and monitoring tools are an example for that, which are deployed only on 
Kubernetes nodes specially tainted for them. This is done using tolerations and node affinity on deployment time as follows.

```yaml
# Manifest shorted for better focusing
affinity: &affinity
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
          - key: node-type
            operator: In
            values:
              - monitoring
tolerations: &tolerations
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
# Git repository for Kafka
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: monitoring-stack
  namespace: flux-system
spec:
  interval: 20s
  ref:
    branch: master
  secretRef:
    name: flux-system
  url: ssh://git@gitlab.s73cloud.com:13579/infrastructure/monitoring-stack.git
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

Anyway, as described previously, we did it for you in the templates, i.e. the template for
[launching EKS clusters](https://gitlab.s73cloud.com:29999/Infrastructure/flux-clusters/templates/base-eks/-/tree/master/infrastructure/monitoring-stack)

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

## Full documentation

You have a simple set of instructions on this [README](docs/README.md) to launch the docs locally

## How to collaborate

1. Create a branch and change inside everything you need
2. Launch a cluster pointing monitoring-stack to this repository but to your branch
3. Open a Pull Request to merge your code. Don't be dirty with the commits (squash them), be clear with the commit message, etc
