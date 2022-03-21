# Spinnaker Kustomize

Kustomize based installation method for Spinnaker as an alternative to
[https://github.com/spinnaker/halyard](halyard).
This repository was inspired by https://github.com/spinnaker/kustomization-base
and differs in the following ways:

1. Optimize for deployment of a basic Spinnaker installation with one command.
1. No Halyard, kleat or other tools for configuration. Use the services default
   configuration (eg: `clouddriver.yml`) and leverage Spring Profile's for
   customization (eg: `clouddriver-local.yml`).

### Prerequisites

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

## Production ready Spinnaker

Production workloads require higher reliability and scale than the default
configuration will support.

For each of the following areas choose an alternative implementation or supply
your own settings as required.

### Configuration

For `Deck`, custom configuration is mounted at
`/opt/spinnaker/config/settings-local.js`.
For Java services, custom configuration files are mounted in the
`/opt/spinnaker/config/` directory.

You can find the default configuration file for each service in the services
git repository. Check out the branch related to the version you are running.
For release 1.29.0 check out branch `release-1.29.x`.

For Deck, see: https://github.com/spinnaker/deck/blob/master/halconfig/settings.js

For Java services, look in the `<service>-web/config` directory. For example:
https://github.com/spinnaker/clouddriver/blob/master/clouddriver-web/config/clouddriver.yml

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

#### Custom Overlay Configuration

The Java services leverage [Spring Application Properties - Wildcard Locations](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config.files.wildcard-locations).
This means that custom overlays need to mount files at `/opt/spinnaker/config/<overlay_name>/<application>.yml`.
Where `<application>.yml` can be `{application}.yml` or `{application-profile}.yml`.

The convention adopted by this repository is:

1. Add an item to the service's `ConfigMap` using a Patch file.
   For example The MariaDB overlay has a Clouddriver ConfigMap that contains
   a file called `mariadb.yml`.
1. Mount the ConfigMap item in a subdirectory in the container.
   For example the MariaDB overlay mounts `mariadb.yml` at
   `.../config/mariadb/clouddriver.yml`.

#### Differences to Halyard

The Halyard convention for Spinnaker config is to use the `local` Spring
Profile for any custom configuration.

This doesn't work with Kustomize because it's not possible to append
configuration to a Kubernetes `ConfigMap` item, for example:
`data.clouddriver-local.yml`. Helm can do this but has other limitations.

Kustomize also does not support appending to strings either so we can't add
additional custom Spring Profiles to each container's `SPRING_PROFILES_ACTIVE`
environment variable and then mount files at
`/opt/spinnaker/config/clouddriver-my_overlay.yml`.

### State management

Spinnaker supports a variety of backing data stores such as Redis and SQL.

The default `MariaDB` and `Redis` implementations will require changes to
support higher load and greater reliability.

Edit `./kustomize.yml` and comment out or delete:

```
- ./overlays/mariadb  # comment out or delete this line
- ./overlays/redis    # comment out or delete this line
```

Add your own overlay or edit an existing `./overlays/`:

```
- ./overlays/aws-aurora-mysql
- ./overlays/aws-elasticache-redis
- ./overlays/postgres
- ./overlays/redis-external
- /path/to/your/overlay
```
