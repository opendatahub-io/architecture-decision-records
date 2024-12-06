# Implement TTL in DSP pipeline runs

|                |                                                                 |
| -------------- | --------------------------------------------------------------- |
| Date           | 2024-11-12                                                      |
| Scope          |                                                                 |
| Status         | In Progress                                                     |
| Authors        | [Ricardo Martinelli](@rimolive)                                 |
| Supersedes     | N/A                                                             |
| Superseded by: | N/A                                                             |
| Tickets        | [RHOAIENG-8470](https://issues.redhat.com/browse/RHOAIENG-8470) |
| Other docs:    | none                                                            |

## What

This document describes the strategies the Data Science Pipelines team is planning on implementing TTL in pipeline runs. TTL was a KFPv1 feature, but it was not implemented in v2. Because this is becoming a hot topic, DSP team decided to take that task from the upstream project.

## Why

As described in a [community issue](https://github.com/kubeflow/pipelines/issues/10899), when there are lots of pipelines that run in a certain period of time, the Workflow CRs are kept persisted thus managed by the Argo Workflow controller. Because there is not a TTL feature implemented in KFPv2, this causes cluster outages given the high number of CRs to manage, overloading etcd. A workaround proposed is to create a CronJob that deletes these Workflows periodically.

## Goals

* Propose an implementation for TTL within the upstream community requirements

## Non-Goals

* Bulleted list of non-goals

## How

As discussed with the Kubeflow Pipelines community, the new TTL feature to be implemented in KFPv2 must follow these requirements:

* We should reuse Argo TTL implementation, by adding a way to set TTL in the Python code then translates to an IR yaml document so finally the Argo backend compiler can translate this to the actual [TTL configuration](https://argo-workflows.readthedocs.io/en/latest/fields/#ttlstrategy) in Argo Workflow CR.
* Because KFP/DSP stores run information on Database, the record stored about the TTL'd Workflow must be deleted from table

For the first requirement, it is a simple implementation that allows SDK and the Argo backend compiler to translate the Python code into IR yaml, and then into the actual Argo Workflow CR. With the new [PipelineConfig class introduced in v2](https://github.com/kubeflow/pipelines/pull/11333), we can introduce TTL as a pipeline-level config thus reflecting the TTL configuration in Argo Workflows. An example code is shown below:

```
import os

from kfp import dsl
from kfp import compiler
from kfp.dsl.pipeline_config import PipelineConfig

@dsl.component()
def hello_world(text: str) -> str:
    print(text)
    return text

# Creates the PipelineConfig class and set TTL to 60 seconds
config = PipelineConfig()
config.set_ttl(60)

# Sets the PipelineConfig object to the dsl.pipeline decorator
@dsl.pipeline(name='hello-world', description='A simple intro pipeline', pipeline_config=config)
def pipeline_hello_world(text: str = 'hi there'):

    consume_task = hello_world(text=text)
```

The resulting IR yaml should be the following:

```
# Pipeline IR yaml
---
platforms:
  kubernetes:
    pipelineConfig:
      pipelineTtl: 60
```

Finally, when the pipeline is uploaded and a pipeline run is created, it sets the `ttlStrategy.secondsAfterCompletion` under the Workflow CR.

For the second requirement, our understanding is that we should not interfere with the Argo Workflow Controller, leaving to one of the KFP components to implement the DB cleanup task. Based on the KFP components responsibilities, the Persistence Agent should do that as per the KFPv2 System Design doc (see References):

```
The Pipeline Persistence Agent watches the Kubernetes resources created by the Pipeline Service and persists the state of these resources in the ML Metadata Service
```

That said, DSP team should implement an async task (or extend the existing one) to also check if the workflow CRs gets TTL'd and delete the DB record accordingly by doing the following:

1. When a record in `run_details` table is created, it should also record the calculated time the pipeline run must be TTL'd
2. An async task will query the table periodically to find the DB records that should be deleted in that cycle
3. PA will delete these records

These requirements together brings the following concern: Because it's Argo Workflow Controller responsibility to delete the TTL'd Workflow CRs and the DB cleanup task must be KFP/DSP responsibility, it brings the following scenario:

1. KFP starts a pipeline run
2. A new record is stored in the Database
3. Pipeline run finishes its execution
4. As configured in the Workflow CR that belongs to the pipeline run, the CR will be removed after a period of time
5. KFP runs the DB cleanup async task to remove the TTL'd pipeline run

Because 4. is done by Argo Workflow Controller and 5. is done by KFP, user would still be able to see the run details of a TTL'd pipeline run. That said, we leave with 2 different and independent TTL cycles managed by different components so we must be cautious to ensure users understand this is an expected behavior given the KFP Architecture.

Our proposal is to explicitly introduce a Poll period and document in KFP upstream docs so users understand that Workflow CR and DB record cleanup are different taks done by different components, so they should find a balance between Argo TTL configuration and PA poll period to have DB records to live in the least time as possible compared to Workflow CR.

## Open Questions

Optional section, hopefully removed before transitioning from Draft/Proposed to Accepted.

## Alternatives

One alternative we considered for the DB cleanup task is implementing some special hooks in Argo so KFP gets informed of the removal event and cleanup the DB record. We believe that this will affect KFPv2 backend agnostic design, relying on a specific pipeline engine feature.

For the TTL calculation to query the table with the TTL'd pipeline runs that must be deleted, we found the possibility to do this calculation based on the information stored in the run_details table (KFP stores the full Workflow CR in yaml format on table). Because this might introduce a huge breaking change, our decision is to add a new column with the calculated TTL time.

## Security and Privacy Considerations

N/A

## Risks

Optional section. Talk about any risks here.

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| DSP team                      | key contact name | date       | ? |


## References

* [KFPv2 System Design doc](https://docs.google.com/document/d/1fHU29oScMEKPttDA1Th1ibImAKsFVVt2Ynr4ZME05i0/edit?tab=t.0)

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| name                          | date       | ? |
