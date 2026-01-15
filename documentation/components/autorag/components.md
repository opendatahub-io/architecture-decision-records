# AutoRAG Kubeflow Pipelines Components Structure

> ⚠️ **Warning:** This document is a work in progress and represents a proposed structure. The final implementation may vary based on actual development requirements, KFP Components repository standards, and community feedback. This is not a final version.

This document outlines the proposed structure for AutoRAG components following the [Kubeflow Pipelines Components repository](https://github.com/kubeflow/pipelines-components) standards.

## Repository Structure

Based on the KFP Components repository structure, AutoRAG components would be organized as follows:

```
kubeflow/pipelines-components/
├── components/
│   └── autorag/
│       ├── data-processing/
│       │   ├── document-loader/
│       │   │   ├── component.py          # Component implementation (includes document-sampling)
│       │   │   ├── metadata.yaml         # Component metadata
│       │   │   ├── README.md              # Component documentation
│       │   │   ├── OWNERS                 # Maintainer information
│       │   │   ├── tests/
│       │   │   │   └── test_component.py  # Unit tests
│       │   │   └── example_pipelines.py  # Usage examples
│       │   │
│       │   ├── test-data-loader/
│       │   │   ├── component.py
│       │   │   ├── metadata.yaml
│       │   │   ├── README.md
│       │   │   ├── OWNERS
│       │   │   ├── tests/
│       │   │   └── example_pipelines.py
│       │   │
│       │   └── text-extraction/
│       │       ├── component.py
│       │       ├── metadata.yaml
│       │       ├── README.md
│       │       ├── OWNERS
│       │       ├── tests/
│       │       └── example_pipelines.py
│       │
│       └── optimization/
│           ├── search-space-preparation/
│           │   ├── component.py
│           │   ├── metadata.yaml
│           │   ├── README.md
│           │   ├── OWNERS
│           │   ├── tests/
│           │   └── example_pipelines.py
│           │
│           └── rag-settings-optimization/
│               ├── component.py
│               ├── metadata.yaml
│               ├── README.md
│               ├── OWNERS
│               ├── tests/
│               └── example_pipelines.py
│
└── pipelines/
    └── autorag/
        └── autorag-pipeline/
            ├── pipeline.py               # Complete AutoRAG pipeline
            ├── metadata.yaml
            ├── README.md
            ├── OWNERS
            ├── tests/
            └── example_usage.py
```

## Component Categories

### Data Processing Components

Components responsible for data ingestion, preparation, and transformation:

1. **document-loader** - Loads documents from S3 or local filesystem and performs document sampling
   - Inputs: `connection_id`, `bucket`, `path`, `test_data` (optional), `sampling_config` (optional)
   - Outputs: `sampled_documents` (Artifact)
   - Dependencies: RHOAI Connections API, ai4rag
   - **Note**: Document sampling functionality is integrated within this component

2. **test-data-loader** - Loads test data from JSON files
   - Inputs: `connection_id`, `bucket`, `path`
   - Outputs: `test_data` (Artifact/DataFrame)
   - Dependencies: pandas

3. **text-extraction** - Extracts text from documents using docling
   - Inputs: `documents` (Artifact)
   - Outputs: `extracted_text` (Artifact)
   - Dependencies: docling library

### Optimization Components

Components responsible for RAG configuration optimization:

> ℹ️ **Info:** The optimization components (`search-space-preparation` and `rag-settings-optimization`) can be aggregated into a single component for simplified pipeline composition. The separation shown here is for modularity and reusability, but a combined component is also a valid implementation approach.

1. **search-space-preparation** - Builds and validates RAG configuration search space
   - Inputs: `constraints`, `models_config`
   - Outputs: `validated_configurations` (Artifact)
   - Dependencies: ai4rag, in-memory vector DB

2. **rag-settings-optimization** - Core optimization component using GAM-based prediction
   - Inputs: `configurations`, `test_data`, `vector_database_id`
   - Outputs: `rag_patterns` (Artifacts), `leaderboard` (Metrics)
   - Dependencies: ai4rag, llama-stack API, Milvus/Milvus Lite

## Component Metadata Structure

Each component should include a `metadata.yaml` file with the following structure:

```yaml
name: document-loader
version: 1.0.0
description: Loads documents from S3 or local filesystem for AutoRAG processing
stability: alpha  # alpha, beta, or stable
category: data-processing
dependencies:
  kubeflow:
    min_version: "2.0.0"
  external:
    - name: RHOAI Connections API
      version: ">=1.0.0"
lastVerified: "2025-01-15"
maintainers:
  - name: AutoRAG Team
    email: autorag-team@example.com
tags:
  - autorag
  - data-processing
  - document-loading
```

## Pipeline Structure

The complete AutoRAG pipeline would compose these components:

```python
from kfp import dsl
from kfp_components.components.autorag.data_processing import (
    document_loader,
    test_data_loader,
    text_extraction
)
from kfp_components.components.autorag.optimization import (
    search_space_preparation,
    rag_settings_optimization
)

@dsl.pipeline(
    name='autorag-pipeline',
    description='Complete AutoRAG optimization pipeline'
)
def autorag_pipeline(
    name: str,
    input_data_reference: Dict,
    test_data_reference: Dict,
    results_reference: Dict,
    vector_database_id: str = None,
    optimization: Dict = None,
    chunking_constraints: List[Dict] = None,
    embeddings_constraints: List[Dict] = None,
    generation_constraints: List[Dict] = None,
    retrieval_constraints: List[Dict] = None
):
    # Data Processing Steps
    # Load test data first (needed for document sampling)
    test_data = test_data_loader(
        connection_id=test_data_reference['connection_id'],
        bucket=test_data_reference['bucket'],
        path=test_data_reference['path']
    )
    
    # Load and sample documents (document-sampling is integrated)
    sampled_docs = document_loader(
        connection_id=input_data_reference['connection_id'],
        bucket=input_data_reference['bucket'],
        path=input_data_reference['path'],
        test_data=test_data.outputs['test_data']
    )
    
    # Extract text from sampled documents
    extracted_text = text_extraction(
        documents=sampled_docs.outputs['sampled_documents']
    )
    
    # Optimization Steps
    configurations = search_space_preparation(
        constraints={
            'chunking': chunking_constraints,
            'embeddings': embeddings_constraints,
            'generation': generation_constraints,
            'retrieval': retrieval_constraints
        }
    )
    
    optimization_result = rag_settings_optimization(
        configurations=configurations.outputs['validated_configurations'],
        test_data=test_data.outputs['test_data'],
        extracted_text=extracted_text.outputs['extracted_text'],
        vector_database_id=vector_database_id,
        optimization_settings=optimization
    )
```

## Quality Standards

Following KFP Components repository standards, each component must:

- ✅ Pass linting and formatting checks (Black, pydocstyle)
- ✅ Include comprehensive docstrings
- ✅ Compile successfully with `kfp.compiler`
- ✅ Include metadata with fresh `lastVerified` date
- ✅ Pass automated CI/CD checks
- ✅ Include unit tests with >80% coverage
- ✅ Provide usage examples in `example_pipelines.py`

## Maintenance

- **Verification**: Components flagged when `lastVerified` is older than 9 months
- **Ownership**: Each component has designated owners in `OWNERS` file
- **Removal Process**: Components not verified within 12 months are proposed for removal

## References

- [Kubeflow Pipelines Components Repository](https://github.com/kubeflow/pipelines-components)
- [KFP Component Specification](https://www.kubeflow.org/docs/components/pipelines/reference/component-spec/)
- [Creating KFP Components](https://www.kubeflow.org/docs/components/pipelines/sdk/v2/component-development/)
