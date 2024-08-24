# ModelServing architecture

## Components
- *[KSERVE](https://github.com/opendatahub-io/kserve)*
  - This supports a single model serving platform. For deploying large models such as large language models (LLMs), OpenShift AI includes a single model serving platform that is based on the KServe component. Because each model is deployed from its own model server, the single model serving platform helps you deploy, monitor, scale, and maintain large models that require increased resources.
- *[MODEL MESH](https://github.com/opendatahub-io/modelmesh-serving)*
  - This supports a multi-model serving platform. For deploying small and medium-sized models, OpenShift AI includes a multi-model serving platform that is based on the ModelMesh component. On the multi-model serving platform, you can deploy multiple models on the same model server. Each of the deployed models shares the server resources. This approach can be advantageous on OpenShift clusters that have finite compute resources or pods.
- *[ODH-MODEL-CONTROLLER](https://github.com/opendatahub-io/odh-model-controller)*
  - This component facilitates seamless integration between RHOAI's various components and model serving components, enhancing the interoperability and synergy within the RHOAI ecosystem. It streamlines the integration process, enabling smoother communication and interaction between different modules and services, thereby optimizing the overall performance and functionality of the RHOAI platform


## ModelServing Componets Architecture Diagram 
![ModelServing Componets Architecture Diagram](./modelserving-architecture-High-Level%20Components%20Architecture.jpg)
