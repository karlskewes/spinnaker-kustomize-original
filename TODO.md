# TODO

1. no crash on startup
1. <service>-local.yml support overlay.
1. deck, echo, gate, keel external url's overlay. deck DECK_HOST & API_HOST
1. mariadb password options
1. resource limits, replicas, jvm Xmx|Xms
1. clouddriver kubernetes provider config
1. clouddriver aws provider config
1. armory observability plugin
1. deck `securityContext`
1. use what useful from old docs

## Old docs

#### Configure Redis

A number of Spinnaker services require a Redis connection; this install pathway
requires there to be a service `redis` in the `spinnaker` namespace.

In the most common case, you will have an external redis; the template repo has
a `Service` configured with an `ExternalName`; you can update this to point to
the DNS name of your redis service.

For more complex cases, please refer to the following
[blog post](https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-mapping-external-services)
on best practices for mapping external services. In general, the only
requirement of your solution is that you have a service named `redis` in the
`spinnaker` namespace that routes to a valid `redis` backend.

Regardless of the approach you choose, add all the relevant redis Kubernetes
objects to your customization via `kustomize edit add resource redis/*.yml`.

#### Add any secret files

The `secrets` folder is intended to hold any secret files that are referenced by
your config files. The files in this folder will be mounted in `/var/secrets` in
each pod.

To add a secret, put it in the `secrets/` folder and add it to the `files` field
of the `spinnaker-secrets` entry in the `kustomization.yaml`. In this example,
we'll add a _kubeconfig_ file called `k8s-kubeconfig`:

```shell script
cp <path to kubeconfig> secrets/k8s-kubeconfig
```

and update the secret in the `kustomization.yaml` file as:

```yaml
- behavior: merge
  name: spinnaker-secrets
  files:
    - secrets/k8s-kubeconfig
```

#### Enable optional services

To enable Fiat, ensure that `security.authz` is set to `true` in your hal
config

To enable Keel, ensure that `managedDelivery.enabled` is set to `true` in your hal
config

#### Set the Spinnaker version

By default, this base kustomization deploys the `master-latest-validated`
version of each microservice, which is most recent version that has passed our
integration tests. To deploy a different version, you'll need to override the
version of each microservice in the `images` block in the `kustomization.yml`
file.

To deploy a specific version of Spinnaker, override each image's tag with
`spinnaker-{version-number}`. For example, to deploy Spinnaker 1.21.0, override
the tag for each microservice to be `spinnaker-1.21.0`:

```yaml
images:
  - name: us-docker.pkg.dev/spinnaker-community/docker/clouddriver
    newTag: spinnaker-1.21.0
  - name: us-docker.pkg.dev/spinnaker-community/docker/deck
    newTag: spinnaker-1.21.0
# ...
```

#### Replace Gate's readiness probe

If you have not enabled SSL for Gate, override Gate's readiness probe with
the following patch:

```yaml
readinessProbe:
  $patch: replace
  exec:
    command:
      - wget
      - --no-check-certificate
      - --spider
      - -q
      - http://localhost:8084/health
```

Reference the patch in your base `kustomization.yml` by adding the following to
a `patches` block:

```yaml
- target:
    kind: Deployment
    name: gate
  path: path/to/my/readiness/probe/patch.yml
```

#### (Optional) Use a specific version of kustomization

With a reference of the version, you can use a specific version with conviction, after examining if the version works well with your configurations. Without a reference, a resource link always references `master`. You can check out the available versions [here](https://github.com/spinnaker/kustomization-base/releases).

For example:

```yaml
resources:
  - github.com/spinnaker/kustomization-base/core?ref=v0.1.0
```

For further details, see the [documentation](https://kubectl.docs.kubernetes.io/references/kustomize/resource/) for the `resources` field.

#### (Optional) Add any -local configs

In addition to the main `service.yml` config file, each microservice also reads
in the contents of `service-local.yml` to support settings that are not
configurable in the _halconfig_.

If you would like to add a `-local.yml` config file for any service, add it to
the `local/` directory, and update that service's config in the
`kustomization.yaml` to also mount that `local.yml` file.

For example, to configure for clouddriver, add these settings to
`local/clouddriver-local.yml`, and update the `clouddriver-config` entry in the
`kustomization.yaml` to:

```yaml
- behavior: merge
  files:
    - kleat/clouddriver.yml
    - local/clouddriver-local.yml
  name: clouddriver-config
```

#### (Optional) Enable monitoring

The Spinnaker monitoring daemon runs as a sidecar in each Deployment (excluding
Deck). To enable monitoring, copy the [monitoring](/monitoring) directory from
this repository into the `base` directory of your fork of spinnaker-config.

Add the `monitoring` directory to your base kustomization.yml's `resource`
block. This will pull in the kustomization.yml that includes configuration that
each microservice's monitoring sidecar will use to discover the endpoint to poll
for metrics.

Next, copy the [example `patches` block](/monitoring/patches.yml) into your base
kustomization.yml. These patches will add the monitoring sidecar and appropriate
volumes to each Deployment.

To include custom
[metric filters](https://www.spinnaker.io/setup/monitoring/#configuring-metric-filters),
add them to the included `metric-filters` directory in your fork of
spinaker-config, and reference them in the spinnaker-monitoring-filters
`secretGenerator` entry in the root kustomization.yml.
