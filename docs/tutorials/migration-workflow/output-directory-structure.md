---
layout: default
title: "Customizate output directory structure"
permalink: /tutorials/migration-workflow/output-directory-structure
parent: "Migration workflow"
grand_parent: Tutorials
nav_order: 6
---

# Customizate output directory structure

## Big picture

In this example, we illustrate how we could change the output directory structure of the target artifacts generated by Move2Kube. 

1. Let us start by creating a workspace directory `WORKSPACE_DIR`. We will assume all commands are run within this workspace and all sub-directories are created within this workspace.

2. To dump the input at an accessible location, create an input sub-directory `input` and copy the `e2e-demo` into this folder.

3. To see what we get **without** any customization, let us run `move2kube transform -s input/ --qa-skip`. The output folder structure looks like this:
```
myproject/
├── Readme.md
├── deploy
│   ├── cicd
│   ├── compose
│   ├── knative
│   ├── knative-parameterized
│   ├── yamls
│   └── yamls-parameterized
├── scripts
│   ├── builddockerimages.bat
│   ├── builddockerimages.sh
│   ├── buildimages.bat
│   ├── buildimages.sh
│   ├── pushimages.bat
│   └── pushimages.sh
└── source
    └── e2e-demo
```

Let us say, we want to change the output folder structure such that the deployment yaml directories (`yamls` and `yamls-parameterized` sub-directories) get created under `myproject` directory rather than `myproject/deploy` directory. This can be achieved by customizing Move2Kube using custom transformer that is available in [here](https://github.com/konveyor/move2kube-transformers/tree/main/folder-customizer).

4. To use this custom transformer, create a `customizations` sub-directory and copy the `folder-customizer` from [here](https://github.com/konveyor/move2kube-transformers/tree/main/folder-customizer) into this folder.

5. Run the following CLI command: `move2kube transform -s input/ -c customizations/ --qa-skip`. Note that this command has a `-c customizations/` option which is meant to tell Move2Kube to pick up the custom transformer `folder-customizer`. 

Once the output is generated, we can observe the folder structure has changed as shown below. The `myproject/deploy/yamls` folder is moved to `myproject/yamls-elsewhere` and `myproject/deploy/yamls-parameterized` is moved to `myproject/yamls-elsewhere-parameterized`:
```
myproject/
├── Readme.md
├── deploy
│   ├── cicd
│   ├── compose
│   ├── knative
│   └── knative-parameterized
├── scripts
│   ├── builddockerimages.bat
│   ├── builddockerimages.sh
│   ├── buildimages.bat
│   ├── buildimages.sh
│   ├── pushimages.bat
│   └── pushimages.sh
├── source
│   └── e2e-demo
├── yamls-elsewhere
│   ├── config-utils-deployment.yaml
│   ├── config-utils-service.yaml
│   ├── customers-tomcat-deployment.yaml
│   ├── customers-tomcat-service.yaml
│   ├── frontend-deployment.yaml
│   ├── frontend-service.yaml
│   ├── gateway-deployment.yaml
│   ├── gateway-service.yaml
│   ├── inventory-deployment.yaml
│   ├── inventory-service.yaml
│   ├── myproject-ingress.yaml
│   ├── orders-deployment.yaml
│   └── orders-service.yaml
└── yamls-elsewhere-parameterized
    ├── helm-chart
    ├── kustomize
    └── openshift-template
```

## Anatomy of `folder-customizer` transformer
Let us understand, how easy it is to change the folder structure as shown above. The `folder-customizer` transformer contains just a single yaml file (`kubernetes.yaml`) containing the configurations of that transformer.

```
folder-customizer/
└── kubernetes.yaml
```

As can be seen from the contents below, the name of our custom transformer is `KubernetesForFolderChange` (see `name` field in the `metadata` section). In the specification section, let us look at the important fields of interest:
- Specifying that we are going to use an in-built transformer class (see `class` field in `spec` section) `Kubernetes` as the basis of our transformer (Note here that there could different **transformers** implementing the same **transformer class**. The difference could be in the way they are configured.)
- we are asking Move2Kube to pass artifacts of type `IR` (short for Intermediate Representation) to this transformer (see `consumes` field).
- we are stating that the output produced by this transformer is `KubernetesYamls` (see `produce` field). 
- We are also specifying an `override` section which is asking Move2Kube to override the in-built transformer named `Kubernetes` and use our custom transformer in its place.
- Lastly, the key field which makes the folder structure change is the `outputPath` (this is in the `config` section and has a value `"yamls-elsewhere"`) field which provides the folder path prefix for the desired target folder structure for kubernetes yamls.

```yaml
{% raw %}apiVersion: move2kube.konveyor.io/v1alpha1
kind: Transformer
metadata:
  name: KubernetesForFolderChange
  labels:
    move2kube.konveyor.io/inbuilt: true
spec:
  class: "Kubernetes"
  directoryDetect:
    levels: 0
  consumes:
    IR:
      merge: true
  produces:
    KubernetesYamls:
      disabled: false
  override:
    matchLabels: 
      move2kube.konveyor.io/name: Kubernetes
  dependency:
    matchLabels:
      move2kube.konveyor.io/kubernetesclusterselector: "true"
  config:
    outputPath: "yamls-elsewhere"
    ingressName: "{{ .ProjectName }}"
{% endraw %}
```

Next step [Script Injector](/tutorials/migration-workflow/script-inject)