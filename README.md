# Spinnaker Kustomize

Kustomize based installation for Spinnaker.

Repository goals:

1. Optimize for deployment of a basic Spinnaker installation with one command.
1. No Halyard, kleat or other tools for configuration. Use the services default
   configuration (eg: `clouddriver.yml`) and leverage Spring Profile's for
   customization (eg: `clouddriver-local.yml`).
1. Minimize duplication and maintainence. Only minimal configuration and
   patterns are defined in this repository. See
   [spinnaker.io](https://spinnaker.io) Docs and service source code for all
   available options.

Please fork this repository and develop any customizations locally.

Please contribute documentation updates to https://github.com/spinnaker/spinnaker.io

## Prerequisites

Before you start, you will need:

- A Kubernetes cluster and `kubectl` configured to communicate with that
  cluster or for testing you can use the provided example `kind` cluster.

## Quick start

If required, start a local [https://kind.sigs.k8s.io/](KinD) cluster:

```
make create

# kind create cluster --name spinnaker --config kind.yml
```

Generate Kubernetes yaml:

```
make build

# kubectl kustomize -o ./spinnaker.yaml
```

Check what Kubernetes cluster are pointing to:

```
kubectl config current-context
```

Install Spinnaker into the cluster:

```
make apply

# kubectl apply -f ./spinnaker.yaml
```

## Customizing Spinnaker

Production workloads require higher reliability and scale than the default
configuration will support. See [spinnaker.io](https://spinnaker.io) for
configuration options.

1. Fork this repository
1. (Optional) modify `./kustomization.yml`
1. (Optional) add configuration to `./overlays/config/files/`
1. (Optional) add components and overlays

## Configuration

Spinnaker reads custom configuration from: `/opt/spinnaker/config/`.

See: `./overlays/config/files/clouddriver-local.yml`.

This file is added to the `clouddriver` ConfigMap and mounted into the
container at `/opt/spinnaker/config/clouddriver-local.yml`.

You can find the default configuration file for each service in the services
git repository. Check out the branch related to the version you are running.
For release 1.29.0 check out branch `release-1.29.x`.

The Java services leverage Spring Boot framework so some configuration is
defined via Spring Boot [common application properties](https://docs.spring.io/spring-boot/docs/2.4.13/reference/html/appendix-application-properties.html#common-application-properties).

Configuration sources merge per Spring Boot [external configuration](https://docs.spring.io/spring-boot/docs/2.4.13/reference/html/spring-boot-features.html#boot-features-external-config).

Note both of the above Spring Boot links are subject to change as Spinnaker
upgrades Spring Boot versions. See: [Spinnaker Dependency
Versions](https://github.com/spinnaker/kork/blob/master/spinnaker-dependencies/spinnaker-dependencies.gradle)

Secrets can be supplied in the following ways:

1. Environment variables, mounted via Kubernetes Secret
1. Files, mounted via Kubernetes Secret
1. [Secret Engines](https://spinnaker.io/docs/reference/halyard/secrets/#non-halyard-configuration)
   such as S3, GCS and potentially others.

### Developing Kustomize Components

The Java services leverage [Spring Application Properties - Wildcard Locations](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config.files.wildcard-locations).
This means that custom `components` or `overlays` can also mount files at
`/opt/spinnaker/config/<example>/<service>.yml`.

Where `<service>.yml` can be `{application}.yml` or `{application}-{profile}.yml`.

For example, adding MariaDB support to Clouddriver:

1. In the `./components/mariadb/` directory
1. `files/clouddriver.yml` contains Clouddriver SQL configuration
1. `kustomization.yml` generates a ConfigMap `clouddriver-mariadb` with the
   above file
1. The ConfigMap is added to the Clouddriver Deployment Projected Volume
   sources, mounting the file at: `/opt/spinnaker/config/mariadb/clouddriver.yml`

The quick start MariaDB and Redis components spawn a ConfigMap per component.
This convention enables the components to be standalone in this repository.

Production grade Spinnaker installations tend to use cloud services or more
sophisticated database services.

Separating configuration across many files and ConfigMap's can make development
and troubleshooting difficult so try to put configuration directly into a
single file such as `clouddriver-local.yml`.

If this is insufficient then consider adapting the MariaDB component pattern
and sharing a ConfigMap via [configMapGenerator](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/configmapgenerator/)
`create` and `merge` options.

### Kustomize limitations

Additional files can be added (merged) into a `ConfigMap`. However, it is not
possible to append lines to a Kubernetes `ConfigMap` item, for example:

```
  data:
    clouddriver-local.yml: |
      # Some existing configuration

      # << Kustomize can't merge or append lines to clouddriver-local.yml
```

Many Spinnaker installations use custom Spring Profile's to load additional
configuration.

Unfortunately Kustomize `ValueAddTransformer` has limited functionality and
it is not possible to append strings. For example, appending custom Spring
Profiles to each container's `SPRING_PROFILES_ACTIVE` environment variable.

It is possible to replace strings so that could be suitable for some use cases.
