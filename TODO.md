# TODO

1. keel crashes on startup, orca config
1. deck, echo, gate, keel external url's overlay. deck DECK_HOST & API_HOST
1. mariadb password options
1. clouddriver kubernetes provider config
1. clouddriver aws provider config
1. armory observability plugin
1. deck `securityContext`
1. use what useful from old docs

## Old docs

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
