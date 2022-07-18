# How to create a custom supply chain for Tanzu Application Platform

This guide describes how to create a custom supply chain for
[Tanzu Application Platform](https://tanzu.vmware.com/application-platform),
starting from the existing configuration.

In this example, we're about to add a label for the author of every workloads.

## Prerequisites

Make sure you have the following tools available on your workstation:

- `ytt` and `imgpkg` from the [Carvel toolsuite](https://carvel.dev/)
- `kubectl` configured on a Kubernetes cluster having TAP already installed.
- a Docker daemon (you may use [Docker Desktop](https://www.docker.com/products/docker-desktop/))


[Follow these instructions](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.2/tap/GUID-scc-authoring-supply-chains.html)
to download the configuration for the existing out-of-the-box supply chain and its templates.

```shell
imgpkg pull -b $(kubectl get app ootb-templates -n tap-install  -o "jsonpath={.spec.fetch[0].imgpkgBundle.image}") -o ootb-templates
imgpkg pull -b $(kubectl get app ootb-supply-chain-basic -n tap-install  -o "jsonpath={.spec.fetch[0].imgpkgBundle.image}") -o ootb-supply-chain-basic
```

You should end up with 2 directories:

```shell
tree -L 1 -d
.
├── ootb-supply-chain-basic
└── ootb-templates
```

## Creating a custom supply chain

Create directory `custom-supply-chain`, and copy files
`ootb-supply-chain-basic/config/supply-chain.yaml` and
`ootb-templates/config/kpack-template.yaml` by adding some suffix:

```shell
mkdir custom-supply-chain
cp ootb-supply-chain-basic/config/supply-chain.yaml custom-supply-chain/supply-chain-author.yaml
cp ootb-templates/config/kpack-template.yaml custom-supply-chain/kpack-template-author.yaml
```

Edit file `custom-supply-chain/supply-chain-author.yaml` by changing
the supply chain name, the type selector, and also the reference to `kpack-template`:

```yaml
apiVersion: carto.run/v1alpha1
kind: ClusterSupplyChain
metadata:
  name: source-to-url-author
spec:
  selector:
    apps.tanzu.vmware.com/workload-type: web-author
...
  - name: image-builder
    templateRef:
      kind: ClusterImageTemplate
      options:
        - name: kpack-template-author
          selector:
            matchFields:
              - key: spec.params[?(@.name=="dockerfile")]
                operator: DoesNotExist
...
```

Note the use of a different name for the [Kpack](https://github.com/pivotal/kpack) template.

Edit file `custom-supply-chain/kpack-template-author.yaml` by changing
the template name accordingly:

```yaml
apiVersion: carto.run/v1alpha1
kind: ClusterImageTemplate
metadata:
  name: kpack-template-author
...
```

You also need to add this block in the file, inside the `build.env` section:

```yaml
#! ---
#! CUSTOMIZATION: BEGIN
#! Add env variable BP_OCI_AUTHORS using a value from the workload definition.
#!
#@ if hasattr(data.values.workload.metadata.labels, "app.kubernetes.io/created-by"):
- name: BP_OCI_AUTHORS
  value: #@ data.values.workload.metadata.labels["app.kubernetes.io/created-by"]
#@ end
#!
#! CUSTOMIZATION: END
#! ---
```

This block is responsible for setting the env variable `BP_OCI_AUTHORS` at build time.
This variable is used by the buildpack
[paketo-buildpacks/image-labels](https://github.com/paketo-buildpacks/image-labels)
to set the label `org.opencontainers.image.authors` in the resulting image,
using the value of the label `app.kubernetes.io/created-by` set
in the workload definition (if any).

You're almost done: now it's time to submit these new resources to your cluster.

Submit the new Kpack template:

```shell
kubectl apply -f custom-supply-chain/kpack-template-author.yaml
```

Let's focus on the supply chain. Since this supply chain relies on a bunch of
configuration parameters, you need to use `ytt` to interpolate these parameters first:

```shell
ytt --ignore-unknown-comments -f custom-supply-chain/supply-chain-author.yaml \
  -f ootb-supply-chain-basic/values.yaml \
  --data-value registry.server=REGISTRY-SERVER \
  --data-value registry.repository=REGISTRY-REPOSITORY
```

Don't forget to replace `REGISTRY-SERVER` and `REGISTRY-REPOSITORY` with the values
you used in your TAP configuration file (tap-values.yaml)

tap-values.yaml sample 
```yaml
ootb_supply_chain_basic:
  registry:
    server: "harbor.mytanzu.xyz"          <---- REGISTRY-SERVER
    repository: "library/supply-chain"    <---- REGISTRY-REPOSITORY
```

Submit the new supply chain:

```shell
ytt --ignore-unknown-comments -f custom-supply-chain/supply-chain-author.yaml \
  -f ootb-supply-chain-basic/values.yaml \
  --data-value registry.server=REGISTRY-SERVER \
  --data-value registry.repository=REGISTRY-REPOSITORY | kubectl apply -f-
```

From now on, any workload you create using type `web-author` would use the
supply chain `source-to-url-author`.
If you happen to set the label `app.kubernetes.io/created-by` in the workload definition,
then the value would be used in the OCI author label.

For example:

```yaml
apiVersion: carto.run/v1alpha1
kind: Workload
metadata:
  name: tanzu-java-web-app
  labels:
    apps.tanzu.vmware.com/workload-type: web-author
    app.kubernetes.io/part-of: tanzu-java-web-app
    app.kubernetes.io/created-by: AlexR
spec:
  source:
    git:
      url: https://github.com/sample-accelerators/tanzu-java-web-app.git
      ref:
        branch: main
```

## Check that everything is working as expected

Create and submit a workload definition using type `web-author`,
including some value for the label `app.kubernetes.io/created-by`.

Your TAP installation should handle this workload definition,
by creating an image using the custom supply chain you've just created.

**Note**
If you happen to use [Harbor](https://goharbor.io/) as a container registry, please note that
[there's an issue](https://github.com/goharbor/harbor/issues/17190)
preventing from displaying the image author in the UI. That's why we're about to use
the Docker CLI to display the image author.

Get the latest image for your workload:

```shell
LATEST_IMAGE=$(kubectl -n MY-NAMESPACE get cnbimage MY-WORKLOAD -o "jsonpath={.status.latestImage}{'\n'}")
echo $LATEST_IMAGE
my.repo.corp/tap/tanzu-supply-chain/...
```

Make sure you have a Docker daemon running.

Download the container image to your local workstation:

```shell
docker pull $LATEST_IMAGE
```

You can finally check the image label:

```shell
docker inspect -f '{{index .Config.Labels "org.opencontainers.image.authors"}}' $LATEST_IMAGE
AlexR
```

Up to you to create new supply chains!

## Contribute

Contributions are always welcome!

Feel free to open issues & send PR.

## License

Copyright &copy; 2022 [VMware, Inc. or its affiliates](https://vmware.com).

This project is licensed under the [Apache Software License version 2.0](https://www.apache.org/licenses/LICENSE-2.0).
