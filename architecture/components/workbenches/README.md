
# Workbenches architecture

<!-- sources:
- "Kubeflow Notebooks Architecture" https://www.kubeflow.org/docs/components/notebooks/overview/
- "Kubeflow Architecture" https://www.kubeflow.org/docs/started/architecture/
-->

Workbenches component provides a platform to run web-based development environments inside the openshift cluster. In the ML lifecycle, workbenches are utilize as platform in Model Development Stage. As it provides as avenue for the Data Scientist, to explore and experiment on the development of model.

Key features include:

- Native support for [JupyterLab](https://github.com/jupyterlab/jupyterlab), [RStudio](https://github.com/rstudio/rstudio), and [code-server](https://github.com/coder/code-server).
- Tailored integrated environments equipped with the latest tools and libraries.
- Users can create notebook containers directly in the cluster.
- Admins can provide standard notebook images for their organization with required packages pre-installed.
- Access control is managed by Admins, enabling easier notebook management in the organization.

Components:

- *[Notebooks/workbenches](https://github.com/opendatahub-io/notebooks/wiki/Workbenches)*
  - A collection of notebooks tailored for data analysis, machine learning, research and coding within the Openshift ecosystem. Designed to streamline data science workflows, these notebooks offer an integrated environment equipped with the latest tools and libraries. These notebooks were created to be used with Openshift with the Notebook Controller as the launcher. The following are the out of the box notebook images supported with One year cadence:
    
    - Minimal (includes: jupyterlab)
    - Data-scince (includes: jupyterlab, numpy, scipy, pandas,etc)
    - PyTorch (includes: jupyterlab, torch,etc)
    - TensorFlow(includes: jupyterlab, tensorflow,etc)
    - TrustyAI (includes: jupyterlab, trustyai,etc)
  - GPU support: Nvidia (CUDA drivers), Intel (Habana-Gaudi drivers)

- *[Notebook Controller](https://github.com/opendatahub-io/kubeflow/tree/v1.7-branch/components/odh-notebook-controller)*
  - The combination of two controller which acts as the backend for this component. Based on the upstream kubeflow notebook controller and it is responsible to watch the Notebook custom resource events to start the notebook environment along with the following capabilities:
    - Openshift ingress controller integration.
    - Openshift OAuth sidecar injection.
    - Openshift certs injection

<!-- - *[ODH-Elyra](https://github.com/opendatahub-io/elyra)*
  - The Jupyterlab extension plugin for notebooks, it enable submission of ML Kubeflow pipeline workflows. This component is extenion of open-source [Elyra](https://github.com/elyra-ai/elyra), with support for the data science pipeline v2 API. The extension that are focused by workbenches components:
    
    - Pipeline Visual editor
    - Python editor
    - Code snippet editor -->


## High Level architecture

![Workbenches High level Architecture Diagram](./high-level-workbench-arch.drawio.png)

## Workbenches

### Architecture

The structure of the notebook's build chain is derived from the parent image. To better comprehend this concept, refer to the following graph.

![workbenches Architecture](./workbenches-imagestreams.drawio.png)

Each notebook inherits the properties of its parent. For instance, the TrustyAI notebook inherits all the installed packages from the Standard Data Science notebook, which in turn inherits the characteristics from its parent, the Minimal notebook.

The Rstudio arhitecture is little different than other as the component, is not shipped as image, 
instead shipped as buildconfig, so user can build the Rstudio on their cluster as per their need.
As the RStudio is in Dev Preview.

![rstudio Architecture](./rstudio-imagestream.drawio.png)


## Notebook Controller

### Architecture

![Notebook Controller](./notebook-controller.drawio.png)

### Spec

The user needs to specify the PodSpec for the Workbenches. Based on the selection made by user, the Dashboard component submits the Custom Resource to the cluster.
For example:

```yaml
apiVersion: kubeflow.org/v1
kind: Notebook
metadata:
  name: my-notebook
spec:
  template:
    spec:
      containers:
        - name: my-notebook
          image: standard-data-science:ubi9-python3.9
          args:
            [
              "start.sh",
              "lab",
              "--LabApp.token=''",
              "--LabApp.allow_remote_access='True'",
              "--LabApp.allow_root='True'",
              "--LabApp.ip='*'",
              "--LabApp.base_url=/test/my-notebook/",
              "--port=8888",
              "--no-browser",
            ]
```

The required fields are `containers[0].image` and (`containers[0].command` and/or `containers[0].args`).
That is, the user should specify what and how to run.

All other fields will be filled in with default value if not specified.

By default, when the ODH notebook controller is deployed along with the
Kubeflow notebook controller, it will expose the notebook in the Openshift
ingress by creating a TLS `Route` object.

If the notebook annotation `notebooks.opendatahub.io/inject-oauth` is set to
true, the OAuth proxy will be injected as a sidecar proxy in the notebook
deployment to provide authN and authZ capabilities:

```yaml
apiVersion: kubeflow.org/v1
kind: Notebook
metadata:
  name: example
  annotations:
    notebooks.opendatahub.io/inject-oauth: "true"
```

A [mutating webhook](./controllers/notebook_webhook.go) is part of the ODH
notebook controller, it will add the sidecar to the notebook deployment. The
controller will create all the objects needed by the proxy as explained in the
architecture diagram.

When accessing the notebook, you will have to authenticate with your Openshift
user, and you will only be able to access it if you have the necessary
permissions.

The authorization is delegated to Openshift RBAC through the `--openshfit-sar`
flag in the OAuth proxy:

```json
--openshift-sar=
{
    "verb":"get",
    "resource":"notebooks",
    "resourceAPIGroup":"kubeflow.org",
    "resourceName":"example",
    "namespace":"opendatahub"
}
```

That is, you will only be able to access the notebook if you can perform a `GET`
notebook operation on the cluster:

```shell
oc get notebook example -n <YOUR_NAMESPACE>
```

