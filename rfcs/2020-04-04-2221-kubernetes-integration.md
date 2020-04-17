# RFC 2221 - 2020-04-04 - Kubernetes Integration

This RFC outlines how the Vector will integration with Kubernetes (k8s).

**Note: This RFC is retroactive and meant to serve as an audit to complete our
Kubernetes integration. At the time of writing this RFC, Vector has already made
considerable progress on it's Kubernetes integration. It has a `kubernetes`
source, `kubernetes_pod_metadata` transform, an example `DaemonSet` file, and the
ability automatically reload configuration when it changes. The fundamental
pieces are mostly in place to complete this integration, but as we approach
the finish line we're being faced with deeper questions that heavily affect the
UX. Such as how to properly deploy Vector and exclude it's own logs ([pr#2188]).
We had planned to perform a 3rd party audit on the integration before
announcement and we've decided to align this RFC with that process.**

## Motivation

Kubernetes is arguably the most popular container orchestration framework at
the time of writing this RFC; many large companies, with large production
deployments, depend heavily on Kubernetes. Kubernetes handles log collection
but does not facilitate shipping. Shipping is meant to be delegated to tools
like Vector. This is precisely the use case that Vector was built for. So,
motivation is three-fold:

1. A Kubernetes integration is essential to achieving Vector's vision of being
   the dominant, single collector for observability data.
2. This will inherently attract large, valuable users to Vector since Kubernetes
   is generally used with large deployments.
3. It is currently the #1 requested feature of Vector.

## Guide-level Proposal

**Note: This guide largely follows the format of our existing guides
([example][guide_example]). There are two perspectives to our guides: 1) A new
user coming from Google 2) A user that is familiar with Vector. This guide is
from perspective 2.**

This guide covers integrating Vector with Kubernetes. We'll touch on the basic
concepts of deploying Vector into Kubernetes and walk through our recommended
[strategy](#strategy). By the end of this guide you'll have a single,
lightweight, ultra-fast, and reliable data collector ready to ship your
Kubernetes logs and metrics to any destination you please.

### Strategy

#### How This Guide Works

Our recommended strategy deploys Vector as a Kubernetes [DaemonSet]. This is
the most efficient means of collecting Kubernetes observability data since
Vector is guaranteed to deploy _once_ on each of your Nodes. In addition,
we'll use the [`kubernetes_pod_metadata` transform][kubernetes_pod_metadata_transform]
to enrich your logs with the Kubernetes context. This transform interacts with
the Kubernetes watch API to collect cluster metadata and update in real-time
when things change. The following diagram demonstrates how this works:

TODO: insert diagram

### What We'll Accomplish

- Collect data from each of your Kubernetes Pods
  - Ability to filter by container names, Pod IDs, and namespaces.
  - Automatically merge logs that Kubernetes splits.
  - Enrich your logs with useful Kubernetes context.
- Send your logs to one or more destinations.

### Tutorial

#### Kubectl Interface

1.  Configure Vector:

    Before we can deploy Vector we must configure. This is done by creating
    a Kubernetes `ConfigMap`:

    ...insert selector to select any of Vector's sinks...

    ```bash
    cat <<-CONFIG > vector-configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: vector-config
      labels:
        k8s-app: vector
    data:
      vector.toml: |
        # Docs: https://vector.dev/docs/
        # Container logs are available from "kubernetes" input.

        # Send data to one or more sinks!
        [sinks.aws_s3]
          type = "aws_s3"
          inputs = ["kubernetes"]
          bucket = "my-bucket"
          compression = "gzip"
          region = "us-east-1"
          key_prefix = "date=%F/"
    CONFIG
    ```

2.  Deploy Vector!

    Now that you have your custom `ConfigMap` ready it's time to deploy Vector.
    Create a `Namespace` and apply your `ConfigMap` and our recommended
    deployment configuration into it:

    ```shell
    kubectl create namespace vector
    kubectl apply --namespace vector -f vector-configmap.yaml
    kubectl apply -f https://packages.timber.io/vector/latest/kubernetes/vector-global.yaml
    kubectl apply --namespace vector -f https://packages.timber.io/vector/latest/kubernetes/vector-namespaced.yaml
    ```

    - _See [outstanding questions 3, 4, 5, 6, and 7](#outstanding-questions)._

    That's it!

#### Helm Interface

1.  Install [`helm`][helm_install].

2.  Add our Helm Chart repo.

    ```shell
    helm repo add vector https://charts.vector.dev
    helm repo update
    ```

3.  Configure Vector.

    TODO: address this when we decide on the helm chart internals.

4.  Deploy Vector!

    ```shell
    kubectl create namespace vector

    # Helm v3
    helm install \
      cert-manager vector/vector \
      --namespace vector

    # Helm v2
    helm install \
      --name vector \
      --namespace vector \
      vector/vector
    ```

#### Install using Kustomize

1.  Install [`kustomize`][kustomize].

1.  Prepare `kustomization.yaml`.

    Use the same config as in [Kubectl Interface].

    ```yaml
    # kustomization.yaml
    namespace: vector

    resources:
      - https://packages.timber.io/vector/latest/kubernetes/vector-global.yaml
      - https://packages.timber.io/vector/latest/kubernetes/vector-namespaced.yaml
      - vector-configmap.yaml
    ```

1.  Deploy Vector!

    ```shell
    kustomize build . | kubectl apply -f -
    ```

## Design considerations

### Minimal supported Kubernetes version

The minimal supported Kubernetes version is the earliest released version of
Kubernetes that we intend to support at full capacity.

We use minimal supported Kubernetes version (or MSKV for short), in the
following ways:

- to communicate to our users what versions of Kubernetes Vector will work on;
- to run our Kubernetes test suite against Kubernetes clusters starting from
  this version;
- to track what Kubernetes API feature level we can use when developing Vector
  code.

We can change MSKV over time, but we have to notify our users accordingly.

There has to be one "root" location where current MSKV for the whole Vector
project is specified, and it should be a single source of truth for all the
decisions that involve MSKV, as well as documentation. A good candidate for
such location is a file at `.meta` dir of the Vector repo. `.meta/mskv` for
instance.

#### Initial Minimal Supported Kubernetes Version

Kubernetes 1.14 introduced some significant improvements to how logs files are
organized, putting more useful metadata into the log file path. This allows us
to implement more high-efficient flexible ways to filter what log files we
consume, which is important for preventing Vector from consuming logs that
it itself produces - which is bad since it can potentially result in an
flood-kind DoS.

We can still offer support for Kubernetes 1.13 and earlier, but it will be
limiting our high-efficient filtering capabilities significantly. It will
also increase maintenance costs and code complexity.

On the other hand, Kubernetes pre-1.14 versions are quite rare these days.
At the time of writing, the latest Kubernetes version is 1.18, and, according
to the [Kubernetes version and version skew support policy], only versions
1.18, 1.17 and 1.16 are currently maintained.

Considering all of the above, we assign **1.14** as the initial MSKV.

### Helm vs raw YAML files

We consider both raw YAML files and Helm Chart officially supported installation
methods.

With Helm, people usually use the Chart we provide, and tweak it to their needs
via variables we expose as the chart configuration. This means we can offer a
lot of customization, however, in the end, we're in charge of generating the
YAML configuration that will k8s will run from our templates.
This means that, while it is very straightforward for users, we have to keep in
mind the compatibility concerns when we update our Helm Chart.
We should provide a lot of flexibility in our Helm Charts, but also have sane
defaults that would be work for the majority of users.

With raw YAML files, they have to be usable out of the box, but we shouldn't
expect users to use them as-is. People would often maintain their own "forks" of
those, tailored to their use case. We shouldn't overcomplicate our recommended
configuration, but we shouldn't oversimplify it either. It has to be
production-ready. But it also has to be portable, in the sense that it should
work without tweaking with as much cluster setups as possible.
We should support both `kubectl create` and `kubectl apply` flows.
`kubectl apply` is generally more limiting than `kubectl create`.

### Reading container logs

#### Kubernetes logging architecture

Kubernetes does not directly control the logging, as the actual implementation
of the logging mechanisms is a domain of the container runtime.
That said, Kubernetes requires container runtime to fulfill a certain contract,
and allowing it to enforce desired behavior.

Kubernetes tries to store logs at consistent filesystem paths for any container
runtime. In particular, `kubelet` is responsible of configuring the container
runtime it controls to put the log at the right place.
Log file format can vary per container runtime, and we have to support all the
formats that Kubernetes itself supports.

Generally, most Kubernetes setups will put the logs at the `kubelet`-configured
locations in a .

There is [official documentation][k8s_log_path_location_docs] at Kubernetes
project regarding logging. I had a misconception that it specifies reading these
log files as an explicitly supported way of consuming the logs, however, I
couldn't find a confirmation of that when I checked.
Nonetheless, Kubernetes log files is a de-facto well-settled interface, that we
should be able to use reliably.

#### File locations

We can read container logs directly from the host filesystem. Kubernetes stores
logs such that they're accessible from the following locations:

- [`/var/log/pods`][k8s_src_var_log_pods];
- `/var/log/containers` - legacy location, kept for backward compatibility
  with pre `1.14` clusters.

To make our lives easier, here's a [link][k8s_src_build_container_logs_directory]
to the part of the k8s source that's responsible for building the path to the
log file. If we encounter issues, this would be a good starting point to unwrap
the k8s code.

#### Log file format

As already been mentioned above, log formats can vary, but there are certain
invariants that are imposed on the container runtimes by the implementation of
Kubernetes itself.

A particularity interesting piece of code is the [`ReadLogs`][k8s_src_read_logs]
function - it is responsible for reading container logs. We should carefully
inspect it to gain knowledge on the edge cases. To achieve the best
compatibility, we can base our log files consumption procedure on the logic
implemented by that function.

Based on the [`parseFuncs`][k8s_src_parse_funcs] (that
[`ReadLogs`][k8s_src_read_logs] uses), it's evident that k8s supports the
following formats:

- Docker [JSON File logging driver] format - which is essentially a simple
  [`JSONLines`][jsonlines] (aka `ndjson`) format;
- [CRI format][cri_log_format].

We have to support both formats.

### Helm Chart Repository

We should not just maintain a Helm Chart, we also should offer Helm repo to make
installations easily upgradable.

Everything we need to do to achieve this is outlined at the
[The Chart Repository Guide].

We can use a tool like [ChartMuseum] to manage our repo. Alternatively, we can
use a bare HTTP server, like AWS S3 or Github Pages. A tool like like
[ChartMuseum] has the benefit of doing some things for us. It can use S3
for storage, and offers a convenient [helm plugin][helm_push] to release charts,
so the release process should be very simple.

From the user experience perspective, it would be cool if we expose our chart
repo at `https://charts.vector.dev` - short and easy to remember or even guess.

### Deployment Variants

We have two ways to deploy vector:

- as a [`DaemonSet`][daemonset];
- as a [sidecar `Container`][sidecar_container].

Deploying as a [`DaemonSet`][daemonset] is trivial, applies cluster-wide and
makes sense to as default scenario for the most use cases.

Sidecar container deployments make sense when cluster-wide deployment is not
available. This can generally occur when users are not in control of the whole
cluster (for instance in shared clusters, or in highly isolated clusters).
We should provide recommendations for this deployment variant, however, since
people generally know what they're doing in such use cases, and because those
cases are often very custom, we probably don't have to go deeper than explaining
the generic concerns. We should provide enough flexibility at the Vector code
level for those use cases to be possible.

It is possible to implement a sidecar deployment via implementing an operator
to automatically inject Vector `Container` into `Pod`s (via admission
controller), but that doesn't make a lot of sense for us to work on, since
[`DaemonSet`][daemonset] works for most of the use cases already.

### Deployment configuration

It is important that provide a well-thought deployment configuration for the
Vector as part of our Kubernetes integration. We want to ensure good user
experience, and it includes installation, configuration, and upgrading.

We have to make sure that Vector, being itself an app, runs well in Kubernetes,
and sanely makes use of all the control and monitoring interfaces that
Kubernetes exposes to manage Vector itself.

We will provide YAML and Helm as deployment options. While Helm configuration is
templated and more generic, and YAML is intended for manual configuration, a lot
of design considerations apply to both of them.

#### Managing Object

For the reasons discussed above, we'll be using [`DaemonSet`][daemonset].

#### Data directory

Vector needs a location to keep the disk buffers and other data it requires for
operation at runtime. This directory has to persist across restarts, since it's
essential for some features to function (i.e. not losing buffered data if/while
the sink is gone).

We'll be using [`DaemonSet`][daemonset], so, naturally, we can leverage
[`hostPath`][k8s_api_host_path_volume_source] volumes.

We'll be using `hostPath` volumes at our YAML config, and at the Helm Chart
we'll be using this by default, but we'll also allow configuring this to provide
the flexibility users will expect.

An alternative to `hostPath` volumes would be a user-provided
[persistent volume][k8s_docs_persistent_volumes] of some kind. The only
requirement is that it has to have a `ReadWriteMany` access mode.

#### Vector config files

> This section is about Vector `.toml` config files.

Vector configuration in the Kubernetes environment can generally be split into
two logical parts: a common Kubernetes-related configuration, and a custom
user-supplied configuration.

A common Kubernetes-related configuration is a part that is generally expected
to be the same (or very similar) across all of the Kubernetes environments.
Things like `kubernetes` source and `kubernetes_pod_metadata` transform belong
there.

A custom user-supplied configuration specifies a part of the configuration that
contains parameters like what sink to use or what additional filtering or
transformation to apply. This part is expected to be a unique custom thing for
every user.

Vector supports multiple configuration files, and we can rely on that to ship
a config file with the common configuration part in of our YAML / Helm suite,
and let users keep their custom config part in a separate file.

We will then [mount][k8s_api_config_map_volume_source] two `ConfigMap`s into a
container, and start Vector in multiple configuration files mode
(`vector --config .../common.toml --config .../custom.toml`).

#### Vector config file reloads

It is best to explicitly disable reloads in our default deployment
configuration, because this provides more reliability that [eventually consistent
`ConfigMap` updates][configmap_updates].

Users can recreate the `Pod`s (thus restarting Vector, and making it aware of
the new config) via
[`kubectl rollout restart -n vector daemonset/vector`][kubectl_rollout_restart].

#### Strategy on YAML file grouping

> This section is about Kubernetes `.yaml` files.

YAML files storing Kubernetes API objects configuration can be grouped
differently.

The layout proposed in [guide above](#kubectl-interface) is what we're planing
to use. It is in line with the sections above on Vector configuration splitting
into the common and custom parts.

The idea is to have a single file with a namespaced configuration (`DaemonSet`,
`ServiceAccount`, `ClusterRoleBinding`, common `ConfigMap`, etc), a single file
with a global (non-namespaced) configuration (mainly just `ClusterRole`) and a
user-supplied file containing just a `ConfigMap` with the custom part of the
Vector configuration. Three `.yaml` files in total, two of which are supplied by
us, and one is created by the user.

Ideally we'd want to make the presence of the user-supplied optional, but it
just doesn't make sense, because sink has to be configured somewhere.

We can offer some simple "typical custom configurations" at our documentation as
an example:

- with a sink to push data to our Alloy;
- with a cluster-agnosic `elasticsearch` sink;
- for AWS clusters, with a `cloudwatch` sink;
- etc...

We must be careful with our `.yaml` files to make them play well with not just
`kubectl create -f`, but also with `kubectl apply -f`. There are often issues
with impotency when labels and selectors aren't configured properly and we
should be wary of that.

##### Considered Alternatives

We can use a separate `.yaml` file per object.
That's more inconvenient since we'll need users to execute more commands, yet it
doesn't seems like it provides any benefit.

We expect users to "fork" and adjust our config files as they see fit, so
they'll be able to split the files if required. They then maintain their
configuration on their own, and we assume they're capable and know what they're
doing.

### Annotating events with metadata from Kubernetes

Kubernetes has a lot of metadata that can be associated with the logs, and most
of the users expect us to add some parts of that metadata as fields to the
event.

We already have an implementation that does this in the form of
`kubernetes_pod_metadata` transform.

It works great, however, as can be seen from the next section, we might need
to implement a very similar functionality at the `kubernetes` source as well to
perform log filtering. So, if we'll be obtaining pod metadata at the
`kubernetes` source, we might as well enhance the event right there. This would
render `kubernetes_pod_metadata` useless, as there would be no use case for
it that wouldn't be covered by `kubernetes` source.

What parts of metadata we inject into events should be configurable, but we can
and want to offer sane defaults here.

Technically, the approach implemented at `kubernetes_pod_metadata` already is
pretty good.

One small detail is that we probably want to allow adding arbitrary fields from
the `Pod` object record to the event, instead of a predefined set of fields.
The rationale is we can never imagine all the use cases people could have
in the k8s environment, so we probably should be as flexible as possible.
There doesn't seem to be any technical barriers preventing us from offering
this.

### Origin filtering

We can do highly efficient filtering based on the log file path, and a more
comprehensive filtering via metadata from the k8s API, that is, unfortunately,
has a bit move overhead.

The best user experience is via k8s API, because then we can support filtering
by labels/annotations, which is a standard way of doing things with k8s.

#### Filtering based on the log file path

We already do that in our current implementation.

The idea we can derive some useful parameters from the logs file paths.
For more info on the logs file paths, see the
[File locations][anchor_file_locations] section of this RFC.

So, Kubernetes 1.14+ [exposes][k8s_src_build_container_logs_directory] the
following information via the file path:

- `pod namespace`
- `pod name`
- `pod uuid`

This is enough information for the basic filtering, and the best part is it's
available to us without and extra work - we're reading the files anyways.

#### Filtering based on Kubernetes API metadata

Filtering bu Kubernetes metadata is way more advanced and flexible from the user
perspective.

The idea of doing filtering like that is when Vector picks up a new log file to
process at `kubernetes` source, it has to be able to somehow decide on whether
to consume the logs from that file, or to ignore it, based on the state at the
k8s API and the Vector configuration.

This means that there has to be a way to make the data from the k8s API related
to the log file available to Vector.

Based on the k8s API structure, it looks like we should aim for obtaining the
`Pod` object, since it contains essential information about the containers
that produced the log file. Also, is is the `Pod` objects that `kubelet` relies
on to manage the workloads on the node, so this makes `Pod` objects the best
option for our case, i.e. better than fetching `Deployment` objects.

There in a number of approaches to get the required `Pod` objects:

1. Per-file requests.

   The file paths provide enough data for us to make a query to the k8s API. In
   fact, we only need a `pod namespace` and a `pod uuid` to successfully obtain
   the `Pod` object.

2. Per-node requests.

   This approach is to list all the pods that are running at the same node as
   Vector runs. This effectively lists all the `Pod` objects we could possibly
   care about.

One important thing to note is metadata for the given pod can change over time,
and the implementation has to take that into account, and update the filtering
state accordingly.

We also can't overload the k8s API with requests. The general rule of thumb is
we shouldn't do requests much more often that k8s itself generates events.

Each approach has very different properties. It is hard to estimate which ones
are a better fit.

A single watch call for a list of pods running per node (2) should generate
less overhead and would probably be easier to implement.

Issuing a watch per individual pod (1) is more straightforward, but will
definitely use more sockets. We could speculate that we'll get a smaller latency
than with doing per-node filtering, however it's very unclear if that's the
case.

Either way, we probably want to keep some form of cache + a circuit breaker to
avoid hitting the k8s API too often.

##### A note on k8s API server availability and `Pod` objects cache

One downside is we'll probably have to stall the events originated from a
particular log file until we obtain the data from k8s API and decide whether
to allow that file or filter it. During disasters, if the API server becomes
unavailable, we'll end up stalling the events for which we don't have `Pod`
object data cached. It is a good idea to handle this elegantly, for instance
if we detect that k8s API is gone, we should pause cache-busting until it comes
up again - because no changes can ever arrive while k8s API server is down, and
it makes sense to keep the cache while it's happening.

We're in a good position here, because we have a good understanding of the
system properties, and can intelligently handle k8s API server being down.

Since we'll be stalling the events while we don't have the `Pod` object, there's
an edge case where we won't be able to ship the events for a prolonged time.
This scenario occurs when a new pod is added to the node and then kubernetes API
server goes down. If `kubelet` picks up the update and starts the containers,
and they start producing logs, but Vector at the same node doesn't get the
update - we're going to stall the logs indefinitely. Ideally, we'd want to talk
to the `kubelet` instead of the API server to get the `Pod` object data - since
it's local (hence has a much higher chance to be present) and has even more
authoritative information, in a sense, than the API server on what pods are
actually running on the node. However there's currently no interface to the
`kubelet` we could utilize for that.

##### Practical example of filtering by annotation

Here's an example of an `nginx` deployment.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
      annotations:
        vector.dev/exclude: "true"
    spec:
      containers:
        - name: nginx
          image: nginx:1.14.2
          ports:
            - containerPort: 80
```

The `vector.dev/exclude: "true"`
`annotation` at the `PodTemplateSpec` is intended to let Vector know that it
shouldn't collect logs from the relevant `Pod`s.

Upon picking us a new log file for processing, Vector is intended to read the
`Pod` object, see the `vector.dev/exclude: "true"` annotation and ignore the
log file altogether. This should save take much less resources compared to
reading logs files into events and then filtering them out.

This is also a perfectly valid way of filtering out logs of Vector itself.

#### Filtering based on event fields after annotation

This is an alternative approach to the previous implementation.

The current implementation allows doing this, but is has certain downsides -
the main problem is we're paying the price of reading the log files that are
filtered out completely.

In most scenarios it'd be a significant overhead, and can lead to cycles.

### Configuring Vector via Kubernetes API

#### Annotations and labels on vector pod via downward API

We might want to implement support for configuring Vector via annotations
and/or labels in addition to the configuration files at the `ConfigMap`s.

This actually should be a pretty easy thing to do with a [downward API]. It
exposes pod data as files, so all we need is a slightly altered configuration
loading procedure.

This is how is would look like (very simplified):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubernetes-downwardapi-volume-example
  annotations:
    vector.dev/config: |
      [sinks.aws_s3]
      type = "aws_s3"
      inputs = ["kubernetes"]
      bucket = "my-bucket"
      compression = "gzip"
      region = "us-east-1"
      key_prefix = "date=%F/"
spec:
  containers:
    - name: vector
      image: vector-image
      command:
        ["vector", "--k8s-downward-api-config", "/etc/podinfo/annotations"]
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
  volumes:
    - name: podinfo
      downwardAPI:
        items:
          - path: "annotations"
            fieldRef:
              fieldPath: metadata.annotations
```

The `/etc/podinfo/annotations` file will look something like this:

```text
kubernetes.io/config.seen="2020-04-15T13:35:27.290739039Z"
kubernetes.io/config.source="api"
vector.dev/config="[sinks.aws_s3]\ntype = \"aws_s3\"\ninputs = [\"kubernetes\"]\nbucket = \"my-bucket\"\ncompression = \"gzip\"\nregion = \"us-east-1\"\nkey_prefix = \"date=%F/\"\n"
```

It's quite trivial to extract the configuration.

While possible, this is outside of the scope of the initial integration.

#### Custom CRDs

A much more involved feature than the one above would be making `Vector`
configurable via [`Custom Resource Definition`][k8s_docs_crds].

This feature is not considered for the initial integration with Kubernetes, and
is not even explored, since it is a way more advanced level of integration that
we can achieve in the short term in the near future.

This section is here for completeness, and we would probably like to explore
this in the future.

This includes both adding the support for CRDs to Vector itself, and
implementing an orchestrating component (such things are usually called
[operators][k8s_docs_operator] in the k8s context, i.e. `vector-operator`).

### Changes to Vector release process

We need to ship a particular Vector version along with a particular set of k8s
configuration YAML files and a Helm chart. This is so that we can be sure
all our configurations actually tested and known to work for a particular Vector
release. This is very important for maintaining the legacy releases, and for
people to be able to downgrade if needed, which is one of the major properties
for a system-level component like Vector is.

This means we need to orchestrate the releases of the YAML configs and Helm
Charts together with the Vector releases.

Naturally, it's easiest to implement if we keep the code for both the YAML
configs and the Helm Chart in our Vector repo.

The alternative - having either just the Helm Chart or it together with YAML
files in a separate repo - has a benefit of being a smaller footprint to grasp -
i.e. a dedicated repo with just the k8s deployment config would obviously have
smaller code and history - but it would make it significantly more difficult to
correlate histories with Vector mainline, and it's a major downside. For this
reason, using the Vector repo for keeping everything is preferable.

During the release process, together with shipping the Vector version, we'd
have to also bump the Vector versions at the YAML and Helm Chart configs, and
also bump the version of the Helm Chart as well. We then copy the YAML configs
to the same location where we keep release artifacts (i.e. `.deb`s, `.rpm`s,
etc) for that particular Vector version. We also publish a new Helm Chart
release into our Helm Chart repo.

While bumping the versions is human work, and is hard to automate - copying the
YAML files and publishing a Helm Chart release is easy, and we should take care
of that. We can also add CI lints to ensure the version of Vector at YAML
file and Helm Chart and the one the Rust code has baked in match at all times.
Ideally, they should be bumped together atomically and never diverge.

If we need to ship an update to just YAML configs or a new Helm Chart without
changes to the Vector code, as our default strategy we can consider cutting a
patch release of Vector - simply as a way to go through the whole process.
What is bumping Vector version as well, even though there's no practical reason
for that since the code didn't change. This strategy will not only simplify the
process on our end, but will also be very simple to understand for our users.

### Testing

TODO

- integration tests are cluster-agnostic
- at CI we test against `minikube` and against all versions from MSKV till the latest k8s
- at test harness we run non-ephemeral "real" clusters and test against them (i.e. GCP GKE, AWS EKS, Azure K8s, DO K8s, RedHat OpenShift, Rancher, CoreOS Tekton, etc)
- we integrate our unit tests into test harness in such a way that we can run them as correctness tests
- we want to test our deployment configurations - Helm charts, YAML files and etc, in addition to unit tests

## Prior Art

1. [Filebeat k8s integration]
1. [Fluentbit k8s integration]
1. [Fluentd k8s integration]
1. [LogDNA k8s integration]
1. [Honeycomb integration]
1. [Bonzai logging operator] - This is approach is likely outside of the scope
   of Vector's initial Kubernetes integration because it focuses more on
   deployment strategies and topologies. There are likely some very useful
   and interesting tactics in their approach though.
1. [Influx Helm charts]
1. [Awesome Operators List] - an "awesome list" of operators.

## Sales Pitch

See [motivation](#motivation).

## Drawbacks

1. Increases the surface area that our team must manage.

## Alternatives

1. Not do this integration and rely solely on external community-driven
   integrations.

## Outstanding Questions

### From Ben

1. ~~What is the minimal Kubernetes version that we want to support. See
   [this comment][kubernetes_version_comment].~~
   See the [Minimal supported Kubernetes version][anchor_minimal_supported_kubernetes_version]
   section.
1. What is the best to avoid Vector from ingesting it's own logs? I'm assuming
   that my [`kubectl` tutorial](#kubectl-interface) handles this with namespaces?
   We'd just need to configure Vector to exclude this namespace?
1. I've seen two different installation strategies. For example, Fluentd offers
   a [single daemonset configuration file][fluentd_daemonset] while Fluentbit
   offers [four separate configuration files][fluentbit_installation]
   (`service-account.yaml`, `role.yaml`, `role-binding.yaml`, `configmap.yaml`).
   Which approach is better? Why are they different?
1. ~~Should we prefer `kubectl create ...` or `kubectl apply ...`? The examples
   in the [prior art](#prior-art) section use both.~~
   See [Helm vs raw YAML files][anchor_helm_vs_raw_yaml_files] section.
1. From what I understand, Vector requires the Kubernetes `watch` verb in order
   to receive updates to k8s cluster changes. This is required for the
   `kubernetes_pod_metadata` transform. Yet, Fluentbit [requires the `get`,
   `list`, and `watch` verbs][fluentbit_role]. Why don't we require the same?
1. What is `updateStrategy` ... `RollingUpdate`? This is not included in
   [our daemonset][vector_daemonset] or in [any of Fluentbit's config
   files][fluentbit_installation]. But it is included in both [Fluentd's
   daemonset][fluentd_daemonset] and [LogDNA's daemonset][logdna_daemonset].
1. I've also noticed `resources` declarations in some of these config files.
   For example [LogDNA's daemonset][logdna_daemonset]. I assume this is limiting
   resources. Do we want to consider this?
1. What the hell is going on with [Honeycomb's integration
   strategy][honeycomb integration]? :) It seems like the whole "Heapster"
   pipeline is specifically for system events, but Heapster is deprecated?
   This leads me to my next question...
1. How are we collecting Kubernetes system events? Is that outside of the
   scope of this RFC? And why does this take an entirely different path?
   (ref [issue#1293])
1. What are some of the details that set Vector's Kubernetes integration apart?
   This is for marketing purposes and also helps us "raise the bar".

### From Mike

1. What significantly different k8s cluster "flavors" are there? Which ones do
   we want to test against? Some clusters use `docker`, some use `CRI-O`,
   [etc][container_runtimes]. Some even use [gVisor] or [Firecracker]. There
   might be differences in how different container runtimes handle logs.
1. How do we want to approach Helm Chart Repository management.
1. How do we implement liveness, readiness and startup probes?
1. Can we populate file at `terminationMessagePath` with some meaningful
   information when we exit or crash?
1. Can we allow passing arbitrary fields from the `Pod` object to the event?

## Plan Of Attack

- [ ] Agree on minimal Kubernetes version.
- [ ] Agree on a list of Kubernetes cluster flavors we want to test against.
- [ ] Setup a proper testing suite for k8s.
  - [ ] Support for customizable k8s clusters. See [issue#2170].
  - [ ] Look into [issue#2225] and see if we can include it as part of this
        work.
  - [ ] Stabilize k8s integration tests. See [issue#2193], [issue#2216],
        and [issue#1635].
  - [ ] Ensure we are testing all supported minor versions. See
        [issue#2223].
- [ ] Audit and improve the `kubernetes` source.
  - [ ] Handle the log recursion problem where Vector ingests it's own logs.
        See [issue#2218] and [issue#2171].
  - [ ] Audit the `file` source strategy. See [issue#2199] and [issue#1910].
    - [ ] Merge split logs. See [pr#2134].
- [ ] Audit and improve the `kubernetes_pod_matadata` transform.
  - [ ] Use the `log_schema.kubernetes_key` setting. See [issue#1867].
- [ ] Add a way to load optional config files (i.e. load config file if it
      exists, and ignore it if it doesn't). Required to elegantly load multiple
      files so that we can split the configuration.
- [ ] Add `kubernetes` source reference documentation.
- [ ] Prepare YAML deployment config.
- [ ] Prepare Heml Chart.
- [ ] Prepare Heml Chart Repository.
- [ ] Integrate kubernetes configuration snapshotting into the release process.
- [ ] Add Kubernetes setup/integration guide.
- [ ] Release `0.10.0` and announce.

[anchor_file_locations]: #file-locations
[anchor_helm_vs_raw_yaml_files]: #helm-vs-raw-yaml-files
[anchor_minimal_supported_kubernetes_version]: #minimal-supported-kubernetes-version
[awesome operators list]: https://github.com/operator-framework/awesome-operators
[bonzai logging operator]: https://github.com/banzaicloud/logging-operator
[chartmuseum]: https://chartmuseum.com/
[configmap_updates]: https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#mounted-configmaps-are-updated-automatically
[container_runtimes]: https://kubernetes.io/docs/setup/production-environment/container-runtimes/
[cri_log_format]: https://github.com/kubernetes/community/blob/ee2abbf9dbfa4523b414f99a04ddc97bd38c74b2/contributors/design-proposals/node/kubelet-cri-logging.md
[daemonset]: https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/
[downward api]: https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/#store-pod-fields
[filebeat k8s integration]: https://www.elastic.co/guide/en/beats/filebeat/master/running-on-kubernetes.html
[firecracker]: https://github.com/firecracker-microvm/firecracker
[fluentbit k8s integration]: https://docs.fluentbit.io/manual/installation/kubernetes
[fluentbit_daemonset]: https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/output/elasticsearch/fluent-bit-ds.yaml
[fluentbit_installation]: https://docs.fluentbit.io/manual/installation/kubernetes#installation
[fluentbit_role]: https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-role.yaml
[fluentd k8s integration]: https://docs.fluentd.org/v/0.12/articles/kubernetes-fluentd
[fluentd_daemonset]: https://github.com/fluent/fluentd-kubernetes-daemonset/blob/master/fluentd-daemonset-papertrail.yaml
[guide_example]: https://vector.dev/guides/integrate/sources/syslog/aws_kinesis_firehose/
[gvisor]: https://github.com/google/gvisor
[helm_install]: https://cert-manager.io/docs/installation/kubernetes/
[helm_push]: https://github.com/chartmuseum/helm-push
[honeycomb integration]: https://docs.honeycomb.io/getting-data-in/integrations/kubernetes/
[influx helm charts]: https://github.com/influxdata/helm-charts
[issue#1293]: https://github.com/timberio/vector/issues/1293
[issue#1635]: https://github.com/timberio/vector/issues/1635
[issue#1816]: https://github.com/timberio/vector/issues/1867
[issue#1867]: https://github.com/timberio/vector/issues/1867
[issue#1910]: https://github.com/timberio/vector/issues/1910
[issue#2170]: https://github.com/timberio/vector/issues/2170
[issue#2171]: https://github.com/timberio/vector/issues/2171
[issue#2193]: https://github.com/timberio/vector/issues/2193
[issue#2199]: https://github.com/timberio/vector/issues/2199
[issue#2216]: https://github.com/timberio/vector/issues/2216
[issue#2218]: https://github.com/timberio/vector/issues/2218
[issue#2223]: https://github.com/timberio/vector/issues/2223
[issue#2224]: https://github.com/timberio/vector/issues/2224
[issue#2225]: https://github.com/timberio/vector/issues/2225
[json file logging driver]: https://docs.docker.com/config/containers/logging/json-file/
[jsonlines]: http://jsonlines.org/
[k8s_api_config_map_volume_source]: https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#configmapvolumesource-v1-core
[k8s_api_host_path_volume_source]: https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#hostpathvolumesource-v1-core
[k8s_docs_crds]: https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/
[k8s_docs_operator]: https://kubernetes.io/docs/concepts/extend-kubernetes/operator/
[k8s_docs_persistent_volumes]: https://kubernetes.io/docs/concepts/storage/persistent-volumes
[k8s_log_path_location_docs]: https://kubernetes.io/docs/concepts/cluster-administration/logging/#logging-at-the-node-level
[k8s_src_build_container_logs_directory]: https://github.com/kubernetes/kubernetes/blob/31305966789525fca49ec26c289e565467d1f1c4/pkg/kubelet/kuberuntime/helpers.go#L173
[k8s_src_parse_funcs]: https://github.com/kubernetes/kubernetes/blob/e74ad388541b15ae7332abf2e586e2637b55d7a7/pkg/kubelet/kuberuntime/logs/logs.go#L116
[k8s_src_read_logs]: https://github.com/kubernetes/kubernetes/blob/e74ad388541b15ae7332abf2e586e2637b55d7a7/pkg/kubelet/kuberuntime/logs/logs.go#L277
[k8s_src_var_log_pods]: https://github.com/kubernetes/kubernetes/blob/58596b2bf5eb0d84128fa04d0395ddd148d96e51/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L60
[kubectl_rollout_restart]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-restart-em-
[kubernetes version and version skew support policy]: https://kubernetes.io/docs/setup/release/version-skew-policy/
[kubernetes_version_comment]: https://github.com/timberio/vector/pull/2188#discussion_r403120481
[kustomize]: https://github.com/kubernetes-sigs/kustomize
[logdna k8s integration]: https://docs.logdna.com/docs/kubernetes
[logdna_daemonset]: https://raw.githubusercontent.com/logdna/logdna-agent/master/logdna-agent-ds.yaml
[pr#2134]: https://github.com/timberio/vector/pull/2134
[pr#2188]: https://github.com/timberio/vector/pull/2188
[sidecar_container]: https://github.com/kubernetes/enhancements/blob/a8262db2ce38b2ec7941bdb6810a8d81c5141447/keps/sig-apps/sidecarcontainers.md
[the chart repository guide]: https://helm.sh/docs/topics/chart_repository/
[vector_daemonset]: 2020-04-04-2221-kubernetes-integration/vector-daemonset.yaml
