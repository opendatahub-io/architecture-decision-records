# ADR RHAISTRAT-1074 Create ability to sign and verify AI Artifacts in Registry

| Date | January , 2026 |
| :---- | :---- |
| Scope | AI Hub |
| Status | Refinement completed. TP in 3.4 |
| Authors | AI Hub |
| Supersedes | N/A |
| Superseded by: | N/A |
| Tickets | [RHAISTRAT-1074](https://issues.redhat.com/browse/RHAISTRAT-1074) ( [RHAIRFE-949](https://issues.redhat.com/browse/RHAIRFE-949) ) |
| Other docs: | Refinement doc: [Feature Refinement-RHAISTRAT-1074-Create ability to sign and verify AI Artifacts in Registry](https://docs.google.com/document/d/1jGA2kkRycTNIb7en16SqUE9o0dw5a6unCjA8Ft69PtE/edit?tab=t.3mrf1syv46a)<br/>Refinement walkthrough: [Refinement RHAISTRAT-1074 "Create ability to sign and verify AI Artifacts in Registry" - 2026/01/16 16:55 CET - Recording](https://drive.google.com/file/d/1SLS3QHg1TydbQI1sqlGEF9NIL-JFRN17/view?usp=sharing)<br/>Working doc: [Integrate signing, attestation, and verification of AI Assets into RHOAI](https://docs.google.com/document/d/1yZefAtX4aQCmKldIr3JMwppqArDjl0Fh6rM6sJRPd9s/edit?tab=t.0#heading=h.ri5ppo89xmai) |

## What

The implementation of a mechanism to sign and verify AI artifacts (container images or other models) directly within the Model Registry. This integration utilizes Red Hat TAS/Securesign to provide keyless signing capabilities during the "Store \+ Register" flow.

## Why

* **Security Foundation**: Establishes a basis for downstream consumers (serving, pipelines, training jobs) to verify that a model is congruent and hasn't been tampered with.  
* **Lineage & Provenance**: Enables tracking of model evolution, allowing training jobs to verify a base model's signature before producing a signed, tuned version.  
* **Policy Enforcement**: Builds the groundwork for deployment policies, such as "only deploy signed models from specific identities".  
* As in RFE request:  
  * *“This ability will form the basis for several follow on stories for serving and creation of an AI Asset provenance moving forward.”* 

## Goals

* Integrate **TAS keyless signing** into existing user-workload capabilities (Model Registry python client, async-upload Job).  
  * Support **OpenSSF Model Signing** (OMS) for general AI assets.  
  * Support **Cosign** for artifacts stored as container images.  
* Enable immediate **verification** of signatures using TAS capabilities after the signing process

## Non-Goals

* Providing a prescribed OIDC/IdP solution  
  * (this is a prerequisite of TAS).  
* Deploying or governing the **Securesign instance**  
  * (this remains an Admin responsibility).  
* Revisit Registry management, topology, or multi-tenancy architectures  
  * (integrates on existing validated architecture).  
* Supporting signing mechanisms in TAS other than **Keyless Signing** for this specific feature  
  * (keyless signing is the focus, but tooling can be extended to work per Securesign supported capabilities).

## How

The implementation focuses on integrating TAS capabilities directly into the Registry's "Store \+ Register" flow.

* **User Workflow:** Users utilizing the Model Registry capabilities (Model Registry python client, async-upload Job) client can opt-in to keyless signing during the registration process.  
* **Tooling Strategy:**  
  * For all assets, it uses the Red Hat fork of **OpenSSF Model Signing (OMS)** which TAS provides as \`rh-model-signing\`.  
  * For **container images**, the system uses the Red Hat productized fork of **Cosign**.  
  * These tools must be sourced directly from the **TAS tooling server** due to current divergences between Red Hat forks and upstream Sigstore/OpenSSF projects.  
    * Where possible, we integrate out of the box **rh-model-signing** as a dependency in Model Registry python client:  
      * easiest solution to implement OMS for all model storage destinations, S3, OCI KServe ModelCars, etc.  
      * we are investigating the opportunity of using this directly for the container image signing with the TAS team.  
    * For the fork of Cosign TAS fork  
      * the end user must ensure it is available to the MR python client, as the TAS expectation is for the end user to make it available locally via TAS tooling server (see TAS product documentation)  
* **Infrastructure & Configuration:**  
  * The feature assumes a deployed **Securesign** instance and available TAS toolings for the workload.  
  * Since new "Connection types" cannot be introduced out-of-the-box at this time, administrators will be provided with documentation to manually create the required connection definitions for user workloads.  
* **Verification:** Immediately after an asset is signed, the same TAS tooling is used to verify the signature, ensuring integrity before the user proceeds with the flow.

## Stakeholder Impacts

| Group | Key Contacts | Date | Impacted? |
| :---- | :---- | :---- | :---- |
| PM(BU) | [Adam Bellusci](mailto:abellusc@redhat.com) |  | PM for awareness |
| Architect | [Lindani Phiri](mailto:lphiri@redhat.com) |  | Lead Architect during Refinement |

## References

* **Feature Jira:** [RHAISTRAT-1074](https://issues.redhat.com/browse/RHAISTRAT-1074)  
* **Slack Channel:** [\#openshift-ai-hub-devs](https://redhat.enterprise.slack.com/archives/C08G31WCV16)  
* **Technical Dependencies:** Red Hat Trusted Artifact Signer (TAS), especially also its fork of Cosign, and OpenSSF Model Signing (OMS).

## Reviews

| Reviewed by | Date | Notes |
| :---- | :---- | :---- |
| name | date | ? |
