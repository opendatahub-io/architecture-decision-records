# ADR EvalHub evaluation sidecar credential injection

| Field | Value |
|-------|-------|
| Date | April, 2026 |
| Scope | EvalHub (TrustyAI) |
| Status | Draft |
| Authors | TrustyAI |
| Supersedes | N/A |
| Superseded by | N/A |
| Tickets | https://redhat.atlassian.net/browse/RHOAIENG-53360 |
| Other docs |  |

## What

This document describes the details on how credentials for egress requests from within the evaluation containers are injected by the sidecar container.

## Why

Often evaluation jobs need to reach out to external models such as:
- model targetted for evaluations
- llm as a judge
- the attacker
- etc.

All these models may exist in a same provider such as in a RHOAI cluster hence reachable via AI Gateway & MaaS or externally hosted in other providers such as Azure, Bedrock, Anthropic etc.

The evaluation container runs python code and as EvalHub support BYOF (Bring Your Own Framework) we don't really know what code really runs during the evaluation. So far talking with these model simply sharing the user's API_KEY to the eval container so that the evaluation be able to reach out to these model. This is the tension point because the user's API_KEY is visible to this code and there is the risk of leaking these secrets.

## Goals

Provide a generic mechanism that allows inference with any external LLM from within the eval container without actually sharing the secrets to this container. 

## Non-Goals

This ADR does not aim to inject credentials for any possible egress HTTP request. Only for open-ai chat/completions API and possible extended in the future for open responses API.

## How

We actually apply this very pattern today for OCI registry. EvalHub has the support for exporting the eval results as an OCI artifact mainly for auditing reasons. But the eval container again has not visibility of the OCI credentials. This is how it works:

- We configure the eval container with the OCI host to be localhost (the sidecar container is always deployed in the same POD with the eval container).
- The OCI libraray (oras in this case) submits the request with no credentials to localhost.
- The sidecar container intercepts this requests, injects the correct bearer token credentials and forwards the request to the actual OCI registry. 
- The response flows back to the eval container 

With models it is basically the same pattern but a little more complicated because the eval framework can talk with multiple models potentially from different providers. So the framework may require one or more API_KEYS. Nonetheless the assumption here is that all this interaction is based on the openai chat completions api so all requests need to provide the API-KEY in the `Authorization: Bearer {API_KEY here}` header

We use the same pattern when we talk with mlflow from the eval job, or when we need to report job progress from back to the evalhub service. Credentials are never exposed to the eval container, only to the sidecar container. Think of the sidecar container as a proxy.

Currently in the EvalHub [eval job request API](https://eval-hub.github.io/eval-hub/#tag/Evaluations) the credentials are referenced as:

```json
{
   "model":{
      "url":"<url>",
      "name":"<name>",
      "auth": {
         "secret_ref":"<secret_name>"
       }
   }
   ...
}
```

but this addresses today only the model that we are evaluating. 

Here is an example of the secret used today:

```yaml
apiVersion: v1
kind: Secret
metadata:
  creationTimestamp: "2026-04-01T05:35:15Z"
  name: vllm-api-key
  namespace: team-a
  resourceVersion: "55507468"
  uid: 90ebff8a-4941-4be1-9c8f-42e8d726a06c
type: Opaque
data:
  api-key: dGVzdC1hcGkta2V...
  ca_cert: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1J...
  hf-token: aGZfSFJOd2lHV2huVk9XbHFRVE...
```

`hf-token` is not used for the model but rather for the gated eval datasets. 


Our goal here is to not share the actual user's API_KEY to the eval container as today. In order to do this we propose the following flow:

1. Nothing changes the evalhub REST API. The users still provides the same reference to the secret that containes the actual API_KEY
2. EvalHub service generates the following secret:

#### Secret 1 
```yaml
apiVersion: v1
kind: Secret
metadata:
  creationTimestamp: "2026-04-01T05:35:15Z"
  name: vllm-api-key-eval
  namespace: team-a
  resourceVersion: "55507468"
  uid: 90ebff8a-4941-4be1-9c8f-42e8d726a06c
type: Opaque
data:
  api-key: api-key:ref
  hf-token: hf-token:ref
```

This secret is mounted as today in the eval container. Note that the values for the actual api-key and hf-token are different strings, not the actual credentials. However they are just strings.

#### Secret 2 - the user provided secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  creationTimestamp: "2026-04-01T05:35:15Z"
  name: vllm-api-key
  namespace: team-a
  resourceVersion: "55507468"
  uid: 90ebff8a-4941-4be1-9c8f-42e8d726a06c
type: Opaque
data:
  api-key: dGVzdC1hcGkta2V...
  ca_cert: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1J...
  hf-token: aGZfSFJOd2lHV2huVk9XbHFRVE...
```

This is mounted in the sidecar container. This contains the actual credentials. The flow becomes:

1. The model host in the eval container is configured as localhost. The real host name is known by the sidecar container.
2. Nothing in the eval container code changes. When submitting a completions request it still reads the API_KEY from the evn-var or config but the header will be `Authorization: Bearer api-key:ref` not a real hash.
3. The request is received by the sidecar container, it looks at the Authorization header and determines the this is a key ref. It looks up the `api-key` key in its secret and extract the real API-KEY. 
4. The sidecar container mutates the Authorization header to `Authorization: Bearer dGVzdC1hcGkta2V...` and sends the request to the model provider. 

### Multiple models - multiple API_KEYs

Currently EvalHUb REST API allows the user to only specify the model that needs to be evaluated. Other models that an evaluator needs to use internally are not exposed to the REST API so it is a matter of configuring the evaluator at startup. The other models also require API_KEYs. Often these models use the same API_KEY as the model being evaluated but we cannot assume that this is always the case. Garak for example has a yaml config file where one can specify different apikeys for different models. 

In this case the secret that the user provides to the EvalHub REST API needs to contain API_Keys for multiple models. A proposet secret structure is:

```yaml

apiVersion: v1
kind: Secret
metadata:
  creationTimestamp: "2026-04-01T05:35:15Z"
  name: vllm-api-key
  namespace: team-a
  resourceVersion: "55507468"
  uid: 90ebff8a-4941-4be1-9c8f-42e8d726a06c
type: Opaque
data:
  ...
  model-1_api-key: dGVzdC1hcGkta2V...
  model-2_api-key: dGVzdC1hcGkta2V...
```
where model-1-key, model-2-key are just arbitrary secret key names that the user choses so that the evaluator code reads at startup and configures the eval framework accordingly. It is important for these to have the `_api-key` suffix because EvalHub needs to know which secret properties to prepare. So we follow the same process as above:

1. EvalHub service generates the following secret:

#### Secret 1 
```yaml
apiVersion: v1
kind: Secret
metadata:
  creationTimestamp: "2026-04-01T05:35:15Z"
  name: vllm-api-key-eval
  namespace: team-a
  resourceVersion: "55507468"
  uid: 90ebff8a-4941-4be1-9c8f-42e8d726a06c
type: Opaque
data:
  ...
  model-1_api-key: model-1_api-key:ref
  model-2_api-key: model-2_api-key:ref
```
this is the secret mounted in the actual eval container.

#### Secret 2 - the user provided secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  creationTimestamp: "2026-04-01T05:35:15Z"
  name: vllm-api-key
  namespace: team-a
  resourceVersion: "55507468"
  uid: 90ebff8a-4941-4be1-9c8f-42e8d726a06c
type: Opaque
data:
  ...
  model-1_api-key: dGVzdC1hcGkta2V...
  model-2_api-key: dGVzdC1hcGkta2V...
```

This is mounted in the sidecar container. This contains the actual credentials. The flow becomes:


When the adapter sends a request with the header `Authorization: Bearer model-1_api-key:ref` the sidecar will know exactly which secret key to look up and mutate the header to `Authorization: Bearer dGVzdC1hcGkta2V...` for the egress request.



## Risks

N/A - To be documented as risks are identified during implementation.

## Stakeholder Impacts

| Group | Key Contacts | Date | Impacted? |
|-------|--------------|------|-----------|
| PM(BU) | William Caban | | PM for awareness |

## References

#### EvalHub Service architecture
https://github.com/opendatahub-io/architecture-decision-records/blob/main/architecture-decision-records/eval-hub/ODH-ADR-EH-0001-eval-hub-service.md

## Reviews

| Reviewed by | Date | Notes |
|-------------|------|-------|
| name | date | ? |
