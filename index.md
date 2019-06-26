# FIAAS

FIAAS is a system that adds PaaS like functionality on top of Kubernetes.

FIAAS provides abstractions for specifying applications and what is required for
them to run in a given infrastructure.  These abstractions provide the ability
to separate concerns where application developers can provide a specification
for their applications and cluster operators can provide specification
describing the underlying infrastructure hosting the applications. FIAAS enables
integration of applications with the underlying infrastructure and adapts and
applies changes to both applications and the infrastructure transparently. This
allows for seamless reconfiguration and redeployment of all applications if the
underlying infrastructure is changed and enables applications to be deployed and
run on infrastructures that are not described or specified in the application.
This means that an application can be deployed to two very different Kubernetes
clusters, and still work. In this case, FIAAS acts like an “Interface”, defining
how the connection should look, and the cluster provides the implementation of
that “Interface”.

FIAAS has built-in default application configuration options that will apply to
all applications being deployed. This is used for driving applications
towards a common convention and standardization that is sensible. These options
have built-in defaults but can be adjusted or overwritten as needed on a per
application basis.

FIAAS is designed with continous delivery in mind and enables connecting
continuous integration systems with kubernetes clusters for deploying updated
versions of applications as they become available.

Further information:

* [motivation for creating fiaas](https://github.com/fiaas/fiaas-deploy-daemon/blob/master/docs/fiaas.md).
* [operator guide](https://github.com/fiaas/fiaas-deploy-daemon/blob/master/docs/operator_guide.md)

## Getting Started

Add the FIAAS helm repository

```
helm repo add fiaas https://fiaas.github.io/helm
```

Install [Skipper](https://github.com/fiaas/skipper) which in turn will
bootstrap an instance of [Deploy
Daemon](https://github.com/fiaas/fiaas-deploy-daemon) into
the default namespace

```
helm install fiaas/fiaas-skipper --name=fiaas-skipper --set addFiaasDeployDaemonConfigmap="true"
```

Now there is a Deploy Daemon running in the default namespace.
Deploy Daemon will create a custom resource definition called Application that
contains the specification for a given application.
Deploy Daemon will watch for new and changed Application resources and
provision kubernetes resources as needed.

Since Deploy Daemon has many of the defaults built in application
specifications can be quite simple but defaults can be overwritten as needed. A
basic application spec can look as simple as this:

```
version: 3
```

Specifying only the version implies that only defaults should be used. This
example overwrites some of the defaults:

```
version: 3
healthchecks:
  liveness:
    http:
      path: /_/health
ingress:
- host: hello-world.ingress.local
ports:
- target_port: 5000
replicas:
  maximum: 1
  minimum: 1
```

The application spec needs to be translated into a kubernetes manifest before
it can be applied to the cluster. Generally developers would provide the
application spec as an artifact from the output of their build and use some
utility for generating a kubernetes manifest from it that can then be applied
directly. [Mast](https://github.com/fiaas/mast) is a reference implementation
for generating application manifests and can be used in a continuous
integration setup. It has integration with Artifactory for obtaining
application spec artifacts that it is then able to transform into a manifest.

The manifest can then be applied directly to the cluster

```
cat <<EOF | kubectl apply -f -
apiVersion: fiaas.schibsted.io/v1
kind: Application
metadata:
  labels:
    app: hello-world
    fiaas/deployment_id: 8898f334-e6b0-4212-9ac4-c0b2bd8ce74e
  name: hello-world
  namespace: default
spec:
  application: hello-world
  config:
    healthchecks:
      liveness:
        http:
          path: /_/health
    ingress:
    - host: hello-world.ingress.local
    ports:
    - target_port: 5000
    replicas:
      maximum: 1
      minimum: 1
    version: 3
  image: fiaas-test-app:latest
EOF
```

Now the Deploy Daemon will pick up the application resource and create the
required Kubernetes resources such as replicaset, service, ingress etc. as
specified by the application specification. The application is now available on
hello-world.ingress.local.

The [v3 application
spec](https://github.com/fiaas/fiaas-deploy-daemon/blob/master/docs/v3_spec.md)
contains more information about the different configuration options that can be
expressed through the application specification.

## Components

[Deploy Daemon](https://github.com/fiaas/fiaas-deploy-daemon) is the main
component of FIAAS that will continuously look for application descriptions and
apply them, effectively deploying applications and provisioning the necessary
kubernetes resources that it requires.

[Skipper](https://github.com/fiaas/skipper) is used for managing FIAAS in a
given Kubernetes cluster. With Skipper a cluster operator can install and
upgrade instances of Deploy Daemon across different namespaces.

[Mast](https://github.com/fiaas/mast) can be used for generating the necessary
manifest that can be applied to the cluster. The manifest includes an
application resource that the Deploy Daemon will deploy. Mast has endpoints that
can be consumed by CI systems in combination with Kubernetes api.

## Documentation

[k8s](https://k8s.readthedocs.org/en/latest/) is a python client library for the
Kubernetes api developed by the maintainers of FIAAS.
