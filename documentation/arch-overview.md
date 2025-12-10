# RHOAI Architecture - 2.13

# Overview

The [*Red Hat OpenShift AI*](https://www.redhat.com/en/technologies/cloud-computing/openshift/openshift-ai)
(RHOAI) product enables Data Scientists and ML Engineers to execute end-to-end ML
workflows through the integration of Red Hat components, Open Source
software, and Independent Software Vendor (ISV) Marketplace offerings.

RHOAI is a fully Red Hat managed and self-managed data and AI/ML solution based on the
[*Open Data Hub*](http://opendatahub.io/) (ODH) platform. It leverages Kubeflow,
Jupyter Notebooks, Model Serving (traditional ML and GenAI), Tensorflow & Pytorch support,
Pipeline, Monitoring, Integrated Service Vendors (ISV), data access capabilities, with
additional features on the product roadmap.

The RHOAI managed service is available as an add-on to [OpenShift
Dedicated](https://access.redhat.com/documentation/en-us/openshift_dedicated/4/html-single/introduction_to_openshift_dedicated/index#policies-and-service-definition)
(OSD) on AWS and GCP, and ROSA (Red Hat OpenShift Service on AWS). The
RHOAI self-managed product is available on supported OpenShift Container
Platform (OCP) deployments including on-premise. RHOAI also supports
model deployment through integration into external applications, or
exporting the ODH model for hosting in your environment.

For more detailed information about RHOAI, read the
[*service definition*](https://docs.google.com/document/d/1MKLhNu6ALmuA4UdoqfbvEbLrpGSDcSRaKvMTQX28jdU/edit?usp=sharing).

![Architecture Overview](images/RHOAI%20Architecture-Overview.drawio.png)

# Current Architecture

## Components and Services

Services
- RHOAI Dashboard<br />
    Customer-facing dashboard presenting components as a single unified
    experience

    -   [Core components included as supported functionality within RHOAI](./components/dashboard/README.md#supported-components)

    -   Integrated Red Hat applications such as Red Hat OpenShift API Management

    -   Integrated ISV applications such as Starburst, Anaconda, Intel OpenVino and Nvidia GPU Operator

    -   Tutorials, documentation, quick start examples, and a summary of deployed components with endpoints or links to interfaces.

    ![Dashboard Architecture Overview](images/RHOAI%20Architecture%20-%20D4%20-%20Dashboard.png) 

- Notebook Servers

    -   Kubeflow Notebook Controller - responsible for creation, management, and implementation of notebook servers

    -   ODH Notebook Controller - responsible for the routing and security of notebooks

    -   Data Science focused IDEs

        -   JupyterLab - Notebook interface for data scientists to
            develop machine learning models using code.

    ![Workbenches Architecture Overview](images/RHOAI%20Architecture%20-%20D3%20-%20Workbenches.png) 

- Model Serving

    - KServe ModelMesh Controller - responsible for the management the "Multi Model Deployment" model serving artifacts, including the mesh metadata stored, serving runtimes, and inference services and their associated resources (deployments, services, etc)

        -   etcd is included to persist mesh metadata and can be rebuilt
            in the instance of failure.
   - Open Data Hub Model Mesh Controller - responsible for the routing and security of served models

    - KServe Controller - responsible for the management of the "Single Model Deployment" model serving artifacts, including the mesh metadata stored, serving runtimes, and inference services and their associated resources (deployments, services, etc)
        -   OpenShift Serverless / OpenShift Service Mesh are dependencies used by this component.
 
    ![Model Serving Architecture Overview](images/RHOAI%20Architecture%20-%20D6a%20-%20Model%20Serving.png) 

    ![Model Serving Architecture Overview](images/RHOAI%20Architecture%20-%20D6b%20-%20Model%20Serving.png) 

    ![Model Serving Architecture Overview](images/RHOAI%20Architecture%20-%20D6c%20-%20Model%20Serving.png) 


    - TrustyAI Service - responsible for storing model inference data and providing fairness metrics

    ![Trusty AI Architecture Overview](images/RHOAI%20Architecture%20-%20D7%20-%20Trusty.png) 

- Data Science Pipelines

    -   For an overview of the Data Science Pipelines architecture, see the dedicated [*Data Science Pipelines Architecture*](https://docs.google.com/document/d/1OA9PZpJ8pYxflCFbzLOuVZ3UQyvyGupIcsv6ci2SC5Y/edit?usp=sharing) document.

    ![Data Science Pipelines Architecture Overview](images/RHOAI%20Architecture%20-%20D2%20-%20DSP.png) 

- Distributed Workload
  
  - Ray - responsible for handling distributed computing tasks by allowing scaling machine learning workloads across clusters. It simplifies task distribution, making it ideal for parallel execution of training, inference, and hyperparameter tuning across nodes.

  - CodeFlare - responsible for orchestrating complex machine learning pipelines to streamline the execution of distributed workflows. It simplifies the management of multi-step pipelines, optimizing resource usage and performance on hybrid cloud environments.

  - Kueue - responsible for job queuing and prioritization within Kubernetes, Kueue manages the scheduling and execution of distributed workloads. It ensures efficient resource allocation for batch jobs while respecting Kubernetes constraints and policies.

  ![Distributed Workloads Architecture Overview](images/RHOAI%20Architecture%20-%20D5%20-%20Distr%20Workloads.png) 

- Model Registry
   - To be included yet

- Feature Store 
  - Feast Feature Store is a centralized repository for feature management within OpenShift AI. It is responsible for storing and serving features for model training and serving.
  - Feature Store Controller - responsible for creation, management, and deployment of Feast Feature Servers (online store, offline store, registry).

  ![Feature Store Architecture Overview](images/RHOAI%20Architecture%20-%20D9%20-%20Feature%20Store.png)

                 

Management

-   RHOAI Operator

    -   Meta-operator for all RHOAI components and sub-operators. Responsible for deploying and maintaining all components.
  
    ![Operator Architecture Overview](images/RHOAI%20Architecture%20-%20D1%20-%20Operator.png) 


-   Prometheus

    -   Gathers metrics from the deployed RHOAI components for Red Hat monitoring and billing purposes.


API

- Most of API are saved as Kubernetes custom resources ( [*Documentation*](https://access.redhat.com/articles/7047935) ) 

    - Application deployment

        -   kfdefs.kfdef.apps.kubeflow.org

    - Dashboard

        -   [odhdashboardconfigs.opendatahub.io](./components/dashboard/README.md#odhdashboardconfig-singleton)
        -   [odhapplications.dashboard.opendatahub.io](./components/dashboard/README.md#odhapplication)
        -   [odhdocuments.dashboard.opendatahub.io](./components/dashboard/README.md#odhdocument)
        -   [odhquickstarts.console.openshift.io](./components/dashboard/README.md#odhquickstart)
        -   [acceleratorprofiles.dashboard.opendatahub.io](./components/dashboard/README.md#acceleratorprofiles)

    - Notebooks

        -   notebooks.kubeflow.org

    - Model Serving

        -   servingruntimes.serving.kserve.io
        -   inferenceservices.serving.kserve.io
        -   Predictors.serving.kserve.io

    - Data Science Pipelines

        - Datasciencepipelinesapplications.datasciencepipelinesapplications.opendatahub.io
        - scheduledworkflows.kubeflow.org
        - viewers.kubeflow.org

    - Distributed Workload
        - To be done 

    - Feature Store
        -  featurestores.feast.dev
  

## Deployment

RHOAI Self-managed is deployed as operator directly from OCP console UI, from Menu >
Operators > OperatorHub page.
RHOAI as managed component is deployed as addon using the Red Hat Cluster Manager through a customer's
[*OSD*](https://access.redhat.com/documentation/en-us/openshift_dedicated/4.5/html-single/openshift_dedicated_policies/index?lb_target=production)
subscription and is currently available in AWS and GCP. The addon
triggers RHOAI operator installation into redhat-ods-operator namespace.
In turn, the RHOAI operator creates three new namespaces:
redhat-ods-applications, RHOAI-notebooks, redhat-ods-monitoring.

1.  redhat-ods-operator: namespace for controllers and artifacts for RHOAI management, including the RHOAI operator.

2.  redhat-ods-applications: where the component configured via DataScienceCluster resource is installed.

3.  rhods-notebooks: shared namespace where user notebooks can be instantiated and run but individual users do not have access. This enables a simple workflow without the need to create a project.

4.  redhat-ods-monitoring: where prometheus and other monitoring is installed (To be removed).

The installation can also be done from OCP console UI, from Menu >
Operators > OperatorHub page.

### Network

![](images%2FRHODS%20Architecture%20-%20Network%20Diagram.png)


#### Components

##### Distributed Workloads

![](images%2Fnetwork%2FDistributedWorkloads_KubeRay.png)
![](images%2Fnetwork%2FDistributedWorkloads_KubeFlow_Training_Operator.png)

##### Workbenches

![](images%2Fnetwork%2FWorkbenches.png)

##### Data Sciene Pipelines

![](images%2Fnetwork%2FDataScienePipelines.png)

##### Dashboard

![](images%2Fnetwork%2FDashboard.png)

##### Model Serving

![](images%2Fnetwork%2FModelServing.png)

##### TrustyAI

![](images%2Fnetwork%2FTrustyAI.png)

##### Model Registry

![](images%2Fnetwork%2FModelRegistry.png)


### Storage and Data

Persistent data for RHOAI includes both user data and system data:

-   User Data

    -   Jupyter notebook PVCs (used to store individual users’ notebooks and other files/datasets they upload to the notebook environment. These are also backed by the persistent volume storage provider of the given OpenShift platform (in AWS this is EBS).

    -   Users have the ability to connect to any other arbitrary storage provider they wish in a notebook, such as AWS S3 for both notebooks and model services.

-   System Data

    -   The model mesh controller makes use of its own etcd instance to store metadata. It is highly available but will also restore if lost without significant user impact.

    -   All custom resources deployed by RHOAI will be stored on the cluster etcd storage. This includes all CRs in the groups opendatahub.io, kubeflow.org, and kserve.io

## Monitoring

### Managed Service

As part of supporting RHOAI as a managed service, RHOAI must be
monitored for uptime as part of its SLA, performance, and billing
metrics. The self-managed installation does not require the monitoring
stack.

The RHOAI service will be monitored using Prometheus, Alertmanager, the
Prometheus Blackbox Exporter, and cluster level monitoring. Prometheus
will scrape all metrics from all RHOAI components, and the blackbox
exporter will perform regular synthetic probe checks against services to
ensure their availability. There are additional kube-state-metrics that
the RHOAI-deployed Prometheus can’t scrape directly. These additional
metrics will be “federated” from the in-cluster monitoring stack where
necessary. Alerts will be sent from Prometheus via Alertmanager to
PagerDuty when SRE attention is necessary.

The monitoring stack will be used to measure operational performance of
the RHOAI service (availability of individual services, error rates,
latency, etc.) as well as for billing purposes by measuring users’
consumption of specific features.

The above monitoring tools are intended to be accessed and used solely
by Red Hat support and SRE teams. RHOAI customer users will not have
access, nor will users with RHOAI-admin level of permissions. Should a
customer administrator give themselves cluster admin or dedicated admin
permissions, then they will be able to access these services. The RHOAI
service definition will include verbiage stating that accessing and
modifying deployments in this namespace constitutes a breach of SLA.

Individual RHOAI services produce logging output that can be viewed in
the logs output of individual pods, either in the OpenShift web console
or via the OpenShift CLI tool. These logs will be accessed by Red Hat
support and SRE staff. Customers with the RHOAI-admin level of
permissions will also be able to view these logs for services deployed
in the redhat-ods-applications namespace. Additionally, customer
administrators who have chosen to give themselves cluster admin or
dedicated admin level of permissions, will be able to see these logs.
Finally, customers who choose to enable the cluster logging or log
forwarding features of OSD will be able to access these logs in their
chosen aggregation tool. Retention period and access controls around
these logs depend on the customer’s chosen log aggregation solution.

## Dependencies

-   OpenShift Internal Container Registry
-   Cluster Storage for PVCs

# Installation

### System Requirements

**NOTE: **Any time updates are made to the resources in this section,
please make sure to update the resource requirements summary under
“Resource Recommendations” in the [*RHOAI Service Definition
review*](https://docs.google.com/document/d/1MKLhNu6ALmuA4UdoqfbvEbLrpGSDcSRaKvMTQX28jdU/edit?hl=en&forcehl=1#heading=h.sjyi5zdfjyjn)
document.

The default resource limits for RHOAI are as follows.

| Application            | #Pods | Component                          | vCPU Req / Limit              | Memory Req / Limit             |
|------------------------|-------|------------------------------------|-------------------------------|--------------------------------|
| RHOAI Operator         | 1     | RHOAI-operator                     | 50m / 100m                    | 500Mi / 6Gi                    |
| Dashboard              | 5     | RHOAI-dashboard<br />oauth-proxy   | 500m / 1000m<br />100m / 100m | 1Gi / 2Gi<br />256 Mi / 256 Mi |
| Notebooks              | 1     | notebook-controller-deployment     | 500m / 500m                   | 256Mi / 4Gi                    |
|                        | 1     | odh-notebook-controller-manager    | 500m / 500m                   | 256Mi / 4Gi                    |
| Model Serving          | 3     | modelmesh-controller               | 50m / 1000m                   | 96Mi / 2Gi                     |
|                        | 3     | odh-model-controller               | 10m / 500m                    | 64Mi / 2Gi                     |
|                        | 1     | etcd                               | 200m / 300m                   | 100Mi / 200Mi                  |
| Monitoring             | 1     | prometheus<br />oauth-proxy        | 200m / 400m<br />100m / 100m  | 2Gi / 4Gi<br />256Mi / 256Mi   |
|                        | 1     | blackbox-exporter                  | 100m / 100m                   | 256Mi / 256Mi                  |
| Data Science Pipelines | 1     | data-science-pipelines-operator    | 10m / 1000m                   | 64Mi / 4Gi                     |
| Feature Store          | 1     | feast-operator-controller-manager  | 5m / 5000m                    | 64Mi / 128Mi                   |


The RHOAI infrastructure components will generally require 5 vCPUs and
11GB of memory.

### Software

-   Red Hat supported web browser
-   Access to one of the supported OpenShift Dedicated [*Identity
    Providers*](https://docs.openshift.com/dedicated/4/authentication/dedicated-understanding-authentication.html)
    for user authentication

### Setup

### Configuration

-   Authentication and Authorization

    -   Authentication is performed by both the dashboard and Jupyter notebooks by use of the OpenShift authentication service. The dashboard and notebooks each have their own sidecar proxy setup that takes care of utilizing the OpenShift authentication service on behalf of the services.

        -   Users and administrators belong to assigned OpenShift
            groups.
        -   All users have access assigned to them as normal OpenShift
            users. In addition they can use the RHOAI UI.
        -   Administrators have access to RHOAI settings and
            configuration options.

-   User management

    -   User authentication is provided by one or more of the [*Identity Providers*](https://access.redhat.com/documentation/en-us/red_hat_openshift_data_science/1/html-single/getting_started_with_red_hat_openshift_data_science/index#identity-management-options-for-openshift-data-science_RHOAI) that are supported by OpenShift Dedicated. Once an Identity Provider is configured, the RHOAI administrator is responsible for granting user access to the necessary OpenShift Data Science components

        -   [*Enabling users for OpenShift Data Science using default
            user
            groups*](https://access.redhat.com/documentation/en-us/red_hat_openshift_data_science/1/html-single/managing_users_and_user_resources/index#adding-users-for-openshift-data-science_useradd)

-   Operator Configuration

    -   The operator does not require any special configuration, but the operator image is packaged with a specific released version of the manifests, which in-turn are there to configure the components that are deployed.

    -   When the operator starts-up, an init container (odh-deployer) handles the creation of the redhat-ods-applications and redhat-ods-monitoring projects. It also creates the KfDef custom resource which will be used by the operator to deploy the dashboard and notebook controllers.

    -   The deployer also generates random secrets (via openssl rand -hex 32) for the Prometheus api token, the dashboard oauth cookie, the Prometheus proxy, the Alertmanager proxy, and the Grafana proxy. Those secrets are added to the appropriate OpenShift secrets so that they can be used at runtime.

    -   The deployer creates deployments for the necessary runtime services in the redhat-ods-monitoring project (Grafana, Prometheus, Blackbox exporter).

    -   The deployer also creates the ImageStream objects for all of the CUDA notebook images that are built once when RHOAI is first installed. The corresponding BuildConfigs are also created so that the build chain will be triggered exactly once at install-time.

-   Dashboard Configuration

    -   Dashboard configurations are stored in a custom resource of type odhdashboardconfigs.opendatahub.io named odh-dashboard-config in redhat-ods-applications.

    -   The deployment comes with the “dedicated-admins” user group set up as the value for “admin\_groups”. It also specifies that “system:authenticated” is the group for “allowed\_groups”.

    -   Notebook servers by default are configured with the following sizes:

        -   Small (Request: 8Gi memory, 1 cpu, Limit: 8Gi memory, 2
            cpu),
        -   Medium (Request: 24Gi memory 3 cpu, Limit: 24Gi memory, 6
            cpu),
        -   Large (Request 56Gi memory 7 cpu, Limit: 56Gi memory, 14
            cpu).
        -   X Large (Request 120Gi memory 15 cpu, Limit: 120Gi memory,
            30 cpu).

-   Notebook Controller Configuration

    -   Controller configuration is set in configmap notebook-controller-config

    -   Notebook idle culling settings are set in configmap notebook-controller-culler-config

-   Model Serving Configuration

    -   Users are allowed to control resources for individual serving runtimes in their own user projects. T-shirt sizes for quick sizing are configurable at the cluster level.

    -   Model mesh defaults are stored in configmap “model-serving-config-defaults”

    -   Serving runtimes are configured using the configmap “servingruntimes-config”

-   Monitoring, Logging and Alerting Configs

    -   The deployer is responsible for creating the redhat-ods-monitoring project at install-time. It creates deployments for grafana, prometheus, and blackbox-exporter. All required secrets are randomly generated and stored in either ConfigMaps or Secrets.

    -   The grafana setup also includes grafana dashboards that can be used to readily view critical runtime information about behavior of the RHOAI installation.

    -   The prometheus setup also includes an alertmanager service that is used in the alerting system should an alert need to be thrown.

-   Data Science Pipeline Configuration:

    -   Information on configuration options for Data Science Pipelines is available [*here*](https://docs.google.com/document/d/1OA9PZpJ8pYxflCFbzLOuVZ3UQyvyGupIcsv6ci2SC5Y/edit#heading=h.ohwknwr8rm2h).

  - Feature Store Configuration:

    -   A Feature Store CR will run an OnlineStore Feature Server container by default.
    
        -   Users can configure the OnlineStore, OfflineStore and Registry based on their needs.
    
        -   If they choose to, users can also run server containers for the OfflineStore, Registry, and Feast UI.
    
    -   Users can leverage PVCs for Feast’s supported file-based data stores. The operator can either create PVCs for the user, or it can use existing volumes.
    
    -   Users can configure Feast RBAC to control access to Feast objects.
    
    -   Users can configure TLS for inbound and outbound requests. In an OpenShift cluster, the operator will configure inbound TLS by default.
    
    -   Users can fetch the Feast project directory and its feature definitions from a Git repository.
    
    -   For more details on configuration, see the [Operator API documentation](https://github.com/opendatahub-io/feast/blob/master/infra/feast-operator/docs/api/markdown/ref.md).

-   Hardware/Infrastructure

-   OpenShift Dedicated (AWS, GCP)

    -   RHOAI will run on any of the OSD OCP versions that are officially supported by Red Hat. Please refer to the [*OSD life cycle dates*](https://docs.openshift.com/dedicated/osd_architecture/osd_policy/osd-life-cycle.html#sd-life-cycle-dates_osd-life-cycle) for information about currently supported versions.

## [*Image Components*](https://lucid.app/lucidchart/23cb076f-7a56-42a8-9e0c-51fc5b7d6a85/edit?page=oIuXOACaRr4E#)

### Upstream Repositories

-   Manifests:
    [*https://github.com/opendatahub-io/odh-manifests*](https://github.com/opendatahub-io/odh-manifests)

-   Operator:
    [*https://github.com/opendatahub-io/opendatahub-operator*](https://github.com/opendatahub-io/opendatahub-operator)

-   Dashboard:
    [*https://github.com/opendatahub-io/odh-dashboard*](https://github.com/opendatahub-io/odh-dashboard)

-   Kubeflow:
    [*https://github.com/opendatahub-io/kubeflow*](https://github.com/opendatahub-io/kubeflow)

    -   [*https://github.com/opendatahub-io/kubeflow/tree/master/components/notebook-controller*](https://github.com/opendatahub-io/kubeflow/tree/master/components/notebook-controller)

    -   [*https://github.com/opendatahub-io/kubeflow/tree/master/components/odh-notebook-controller*](https://github.com/opendatahub-io/kubeflow/tree/master/components/odh-notebook-controller)

-   Model Serving

    -   [*https://github.com/opendatahub-io/modelmesh-serving*](https://github.com/opendatahub-io/modelmesh-serving)

    -   [*https://github.com/opendatahub-io/modelmesh*](https://github.com/opendatahub-io/modelmesh)

    -   [*https://github.com/opendatahub-io/modelmesh-runtime-adapter*](https://github.com/opendatahub-io/modelmesh-runtime-adapter)

    -   [*https://github.com/opendatahub-io/rest-proxy*](https://github.com/opendatahub-io/rest-proxy)

    -   [*https://github.com/opendatahub-io/odh-model-controller*](https://github.com/opendatahub-io/odh-model-controller)

    -   [*https://github.com/trustyai-explainability/trustyai-explainability/tree/main/explainability-service*](https://github.com/trustyai-explainability/trustyai-explainability/tree/main/explainability-service)

-   Data Science Pipelines

    -    Information on source code locations for Data Science Pipelines is available [*here*](https://docs.google.com/document/d/1OA9PZpJ8pYxflCFbzLOuVZ3UQyvyGupIcsv6ci2SC5Y/edit#heading=h.moaj376kawl2).

-   Feature Store: [*https://github.com/opendatahub-io/feast*](https://github.com/opendatahub-io/feast)
    -   [*https://github.com/opendatahub-io/feast/tree/master/infra/feast-operator*](https://github.com/opendatahub-io/feast/tree/master/infra/feast-operator)

### Downstream Repositories

-   Manifests:
    [*https://github.com/red-hat-data-services/odh-manifests*](https://github.com/red-hat-data-services/odh-manifests)

-   Operator:
    [*https://github.com/red-hat-data-services/opendatahub-operator*](https://github.com/red-hat-data-services/opendatahub-operator)

-   Deployer:
    [*https://github.com/red-hat-data-services/odh-deployer*](https://github.com/red-hat-data-services/odh-deployer)

-   Dashboard:
    [*https://github.com/red-hat-data-services/odh-dashboard*](https://github.com/red-hat-data-services/odh-dashboard)

-   Kubeflow:
    [*https://github.com/red-hat-data-services/kubeflow*](https://github.com/red-hat-data-services/kubeflow)

    -   [*https://github.com/red-hat-data-services/kubeflow/tree/master/components/notebook-controller*](https://github.com/red-hat-data-services/kubeflow/tree/master/components/notebook-controller)

    -   [*https://github.com/red-hat-data-services/kubeflow/tree/master/components/odh-notebook-controller*](https://github.com/red-hat-data-services/kubeflow/tree/master/components/odh-notebook-controller)

-   Model Serving

    -   [*https://github.com/red-hat-data-services/modelmesh-serving*](https://github.com/red-hat-data-services/modelmesh-serving)

    -   [*https://github.com/red-hat-data-services/modelmesh*](https://github.com/red-hat-data-services/modelmesh)

    -   [*https://github.com/red-hat-data-services/modelmesh-runtime-adapter*](https://github.com/red-hat-data-services/modelmesh-runtime-adapter)

    -   [*https://github.com/red-hat-data-services/rest-proxy*](https://github.com/red-hat-data-services/rest-proxy)

    -   [*https://github.com/red-hat-data-services/odh-model-controller*](https://github.com/red-hat-data-services/odh-model-controller)

    -   [*https://github.com/red-hat-data-services/trustyai-explainability/tree/main/explainability-service*](https://github.com/red-hat-data-services/trustyai-explainability/tree/main/explainability-service)

-   Data Science Pipelines

    -   Information on source code locations for Data Science Pipelines is available [*here*](https://docs.google.com/document/d/1OA9PZpJ8pYxflCFbzLOuVZ3UQyvyGupIcsv6ci2SC5Y/edit#heading=h.moaj376kawl2).
    
-   Feature Store: [*https://github.com/red-hat-data-services/feast*](https://github.com/red-hat-data-services/feast.git)
    -  [*https://github.com/red-hat-data-services/feast/tree/main/infra/feast-operator*](https://github.com/red-hat-data-services/feast/tree/main/infra/feast-operator)

### Change-flow

As documented in the [*RHOAI CI
Pipeline*](https://lucid.app/lucidchart/23cb076f-7a56-42a8-9e0c-51fc5b7d6a85/edit?shared=true&page=oIuXOACaRr4E#?folder_id=home&browser=icon)
chart, RHOAI is released on a 3 week cadence. As part of the short-term
release process, we synchronize from upstream repositories to their
corresponding downstream repositories and create a release branch. That
release branch can include cherry-picked commits from upstream including
fixes without allowing features not ready for production to be part of
the release build. Once changes are brought over, we create a new tagged
release for the repository. The remainder of the build process is
completed via CPaaS.

The odh-deployer is an exception as there is no upstream repository.

Detailed information about the entire process can be found at [*RHOAI
Release
Process*](https://docs.google.com/document/d/1RV0pn9LoCS2ltzQ9JqtkGHEJsCLY41F-MjX-GxBdeNM/edit#heading=h.ur22zd5tzist).

### Downstream images

We have productized (downstream) builds for the following components:

-   RHOAI deployer container (bootstraps the operator)
-   RHOAI operator container
-   RHOAI dashboard container
-   Notebook controller container
-   ODH Notebook controller container
-   Minimal notebook
-   Generic data science notebook
-   TrustyAI Service
-   Feast Feature Store 

The product images are currently delivered to OSD from repositories in
quay.io.

The hardware acceleration images come from:

-   Public quay.io repositories for nfd and the operator init container
-   NVIDIA’s registry (ngc.nvidia.com) for the rest

Details for the downstream builds are summarized in this
[*spreadsheet*](https://docs.google.com/spreadsheets/d/11gup7rQPrbN3yD0ru9pZ2kMHnL-nCJXF4TlJO4vfksQ/edit#gid=0)

### Upstream images

#### RHOAI core application images

#### Notebook images

All notebook images are prebuilt in CPaaS and stored in quay.io

-   s2i-minimal-notebook
    [*quay.io/modh/odh-minimal-notebook-container*](https://quay.io/repository/modh/odh-minimal-notebook-container)
-   s2i-generic-data-science-notebook
    [*quay.io/modh/odh-generic-data-science-notebook*](https://quay.io/repository/modh/odh-generic-data-science-notebook)
-   Minimal-gpu
    [*quay.io/repository/modh/cuda-notebooks*](https://quay.io/repository/modh/cuda-notebooks)
-   PyTorch
    [*quay.io/repository/modh/odh-pytorch-notebook*](https://quay.io/repository/modh/odh-pytorch-notebook)
-   Tensorflow
    [*quay.io/repository/modh/s2i-tensorflow-gpu-notebook*](https://quay.io/repository/modh/s2i-tensorflow-gpu-notebook)


### Build Pipeline

RHOAI uses CPaaS to pull artifacts from the upstream Open Data Hub
github repositories and build downstream artifacts to be deployed into
OpenShift Dedicated. Details about the pipeline are
[*here*](https://docs.google.com/document/d/1crPEEsZkJByBpMPXAAGahChLGR2OtLyo-lyd8G9LRXk/edit?usp=sharing)
and image dependencies are
[*here*](https://docs.google.com/spreadsheets/d/1eC4CGDHqWiz7kMr-7_Uq_WlEyOAjCY4bgKA6H4b8FJU/edit#gid=0).

## [*Image dependencies*](https://docs.google.com/spreadsheets/d/1eC4CGDHqWiz7kMr-7_Uq_WlEyOAjCY4bgKA6H4b8FJU/edit#gid=0)

# Customer Provisioning

## Available Options

RHOAI managed service can be deployed in OpenShift Dedicated using AWS
infrastructure, Google Cloud Platform (GCP), or Red Hat OpenShift on AWS
(ROSA). Future (post-GA) releases of RHOAI will support additional cloud
providers including Microsoft Azure Red Hat OpenShift (ARO).

RHOAI self-managed deployments can be deployed to OCP clusters including
on-premise deployments baremetal, behind proxy, and disconnected
configurations as well as clusters on cloud providers. All architectures
and hardware are expected to be supported in time. This would include
x86, ARM, all GPU types. However, at this time only x86 and Nvidia GPUs
have been tested.

## Billing

[*Pricing Summary
Document*](https://docs.google.com/document/d/1UKrm4aVTMqTLbkPJRtNERVj5BoZJNygf_2tLatJu8Qo/edit)

RHOAI is offered via a traditional annual subscription with a fixed cost
based on the number of vCPUs in the OpenShift cluster for both the
managed service and self-managed configurations.

In addition RHOAI supports consumption-based billing models which charge
based on CPU-hour usage of pods such as Jupyter notebooks or data
science models on as part of the managed service.

-   Details of how our consumption based billing flow works can be seen
    [*here*](https://lucid.app/lucidchart/invitations/accept/inv_88c7a6d1-723c-4076-bf7e-eaf60def82d6)
    and referenced
    [*here*](https://docs.google.com/document/d/1MKLhNu6ALmuA4UdoqfbvEbLrpGSDcSRaKvMTQX28jdU/edit?hl=en&forcehl=1#heading=h.wwoufoy6hi2j).

## Partner Integrations (ISV)

-   Partner Component Installation Options:

    -   *Self Managed*: Installed from Red Hat Marketplace directly. Examples: Seldon, IBM Watson.

    -   *Red Hat Managed*: Installation is a combination of OCM addon and marketplace.

    -   *Partner Managed*: Generally no installation is required. Customer would sign up for the partner cloud service and then follow the quick start guide in RHOAI console for activation and integration options. Example: Anaconda, Starburst Galaxy.

## RHOAI Entitlement

Once a customer has purchased RHOAI, the entitlement will be associated
with their Red Hat account and it will appear as an option in the “Add
Ons” tab for an OpenShift cluster in OCM for the managed service.

## Installation of RHOAI

### Managed Service

Once an OpenShift cluster has been provisioned, a customer will install
RHOAI on the cluster using the “add-on flow” in OCM. In the
configuration tool for a given cluster, access the “Add Ons” tab, click
the tile for Red Hat OpenShift Data Science, then click “Install”. The
installation will take approximately 20 minutes. The installation will
follow the Beta channel and receive frequent updates.

### Self-Managed

Customer will install the RHOAI operator from OperatorHub. The
installation is identical to the Managed Service minus the monitoring
stack. These installations should use the Stable channel.

# System Management

## Service Upgrades

[*Service Upgrade Policy
Doc*](https://docs.google.com/document/d/1ZFBkwtBY3Z0e5QbX3Bk7ZI9mPS5pLsuMesmqRkxnCr0/edit#)

RHOAI will run on any of the OSD OCP versions that are officially
supported by Red Hat. Please refer to the [*OSD life cycle
dates*](https://docs.openshift.com/dedicated/osd_architecture/osd_policy/osd-life-cycle.html#sd-life-cycle-dates_osd-life-cycle)
for information about currently supported versions.

RHOAI will support the current version (N), and the previous 3 released
versions (N-3). Customers outside those versions will not be supported.
Fixes will only be made in the currently released version with some
exceptions for security which may be backported. Customers can choose
automatic updates or manual updates.

# Useful links/Appendix

## Troubleshooting

[*Installation*](https://access.redhat.com/documentation/en-us/red_hat_openshift_data_science/1/html/installing_openshift_data_science/troubleshooting-common-installation-problems_install)

[*Notebooks for
Administrators*](https://access.redhat.com/documentation/en-us/red_hat_openshift_data_science/1/html-single/getting_started_with_red_hat_openshift_data_science/index#troubleshooting-common-problems-in-jupyter-for-administrators_get-started)

[*Notebooks for
Users*](https://access.redhat.com/documentation/en-us/red_hat_openshift_data_science/1/html-single/getting_started_with_red_hat_openshift_data_science/index#troubleshooting-common-problems-in-jupyter-for-users_get-started)

## Support

[*RHOAI - General Availability Support
Plan*](https://docs.google.com/document/u/0/d/183gUCweXQqqoXfS33NimmEzn3T3n6HDOxCjmdEs9c7E/edit)
