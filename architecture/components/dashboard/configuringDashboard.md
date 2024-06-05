[OdhDashboardConfig]: ./README.md#odhdashboardconfig-singleton

# Configuring the Dashboard

* [Configuring Features On/Off](#configuring-features-onoff)
  * [UI-K8s Features](#ui-k8s-feature-eg-ds-projects-feature)
  * [UI-Backend Component Features](#ui-backend-component-feature-eg-ds-pipelines-feature)
* [Configuring Aspects of Features](#configuring-aspects-of-features)

## Configuring Features On/Off

The Dashboard uses a concept we call "areas" to control the flow of features being visible in the Dashboard UI. It effectively is a 2-flag system around every feature we add.

1. A flag for the API installation
2. A flag for the UI installation
    * You may want to disable a UI flow, and interact via an API-driven flow, so you want the backend, but not the frontend

"Areas" are a combination of system settings which can be simplified to these questions:

* Does the Operator, specifically the DataScienceCluster (DSC), have the particular backend component installed?
* Does the [OdhDashboardConfig] have the feature flag enabled?
* Are other areas reliant (a foundation) for this feature & are they installed?

If the "area" is configured with any combination of the 3 questions above, that combination must all be true for the feature to be visible in the UI.

A couple examples are visible below, using snippets of the Dashboard configuration.

### UI-K8s Feature (eg. DS Projects feature)

Our configuration would look something like this:
```javascript
const configurations = {
  [SupportedArea.DS_PROJECTS_VIEW]: {
    featureFlags: ['disableProjects'],
  },
  [SupportedArea.DS_PROJECTS_PERMISSIONS]: {
    featureFlags: ['disableProjectSharing'],
    reliantAreas: [SupportedArea.DS_PROJECTS_VIEW],
  },
  // ...
}
```

* The block above simply says "our feature flag `disableProjects` needs to be enabled to show the DS Projects View (the navigation based view / list page"
* The second configuration `DS_PROJECTS_PERMISSIONS` allows us to configure a sub-portion of the DS Projects feature that you can disable our Project sharing feature (if you're an admin of a project, you can share your project with another user / group on the cluster; aka invite the to join your project) -- this feature is reliant on the Project View being visible (as it's a sub feature) -- but this allows us to

As you can see, there are no backend components listed here. This is because this feature is built on K8s backend itself -- we are interacting with the K8s resources without needing a dedicated OpenShift AI backend.

Note: this would effectively mean there is only active 1 layer of our 2-layered flag system for features like this.

> Tip: To know what features are designed this way, consider the output of each feature. eg. [Creating a DS Project is just a OpenShift Project](#projects---openshift-console-vs-data-science-differences), Creating Custom Images are just ImageStreams on the cluster, etc

### UI-Backend Component Feature (eg. DS Pipelines feature)

Our configuration would look something like this:
```javascript
const configurations = {
  [SupportedArea.DS_PIPELINES]: {
    featureFlags: ['disablePipelines'],
    requiredComponents: [StackComponent.DS_PIPELINES],
  },
  // ...
}
```

* Similar to DS Projects, we have a feature flag
* But now we see `requiredComponents`, this indicates the need of the DSC to provide a successful install of this component otherwise the Dashboard feature flag means nothing -- no backend, can't use the UI anyways

> Tip: To know what features are designed this way, the simple question is... is the feature it in the DSC?

## Configuring Aspects of Features

Many features have various different configurations that give them the ability to be slightly shifted to fit the needs of the customer; these usually are small 'nudges' in the way a feature works -- dropdown values, ordering of importance, display information. These aspects are almost exclusively UI flavouring and not something for the OpenShift AI stack to consume.

See the comprehensive list below for Dashboard-specific aspects.

> Note: These are not a comprehensive list of what each backend feature can do, just what we configure in the Dashboard for our needs.

> Note: For this list, each item will be flagged with **(UI)** or **(API)**. UI features have a UI flow and can be configured through API as needed (aka outside of the UI). API features are not able to be configured inside the UI and can only be configured outside of the UI.

* Workbench Container Sizes **(API)**
  > Visible during creation of a Workbench & Jupyter tile's Notebooks
    * Configured through the [OdhDashboardConfig] `.spec.notebookSizes` an array of resource objects (memory & cpu limits/requests)
    * We have a fallback default if not provided
* Model Serving Container Sizes **(API)**
  > Visible during the creation of a Model Server (or KServe model)
    * Configured through the [OdhDashboardConfig] `.spec.modelServingSizes` an array of resource objects (memory & cpu limits/requests)
    * We have a fallback default if not provided
* Jupyter Tile configurations **(UI)**
    * PVC Size through the [OdhDashboardConfig] `spec.notebookController.pvcSize`
* Telemetry
