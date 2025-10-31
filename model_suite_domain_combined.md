# MBS Valuation System - Model Suite-Centric Architecture (Domain Layer Inside Model Suites)

## Table of Contents
1. Overview
2. Layer Architecture
3. Presentation & Application Layers
4. Model Suite Layer (Includes Domain Logic)
5. Data Access Layer
6. Cache Layer
7. Cross-Layer Interactions
8. Design Principles and Trade-offs
9. Example Request Flow
10. Conclusion

---

## 1. Overview

This document describes a variant of the MBS Valuation System architecture in which the **Domain Layer (business logic, calculation orchestration, model execution)** is included inside the **Model Suite Layer**. With this structure, each Model Suite encapsulates not only its metadata, registry, and model loading logic, but also the business logic and calculation orchestration for its models.

---

## 2. Layer Architecture

```
┌─────────────────────────────────────────────┐
│ PRESENTATION LAYER │
│ CLI, Config Loader, Input Validation │
└──────��──────────┬───────────────────────────┘
 │
 │ ValuationRequest
 ▼
┌─────────────────────────────────────────────┐
│ APPLICATION LAYER │
│ Request Coordination, Output Handling │
└─────────────────┬───────────────────────────┘
 │
 │ Suite Selection, Model Names
 ▼
┌─────────────────────────────────────────────┐
│ MODEL SUITE LAYER │
│ (Registry, Loading, and Domain Logic) │
│ - Model Suite Registry │
│ - Suite Manager │
│ - Suite-Aware Model Loader │
│ - Portfolio Processor │
│ - Calculation Orchestrator │
│ - Model Calculator │
└─────────────────┬───────────────────────────┘
 │
 │ Data Requests
 ▼
┌─────────────────────────────────────────────┐
│ DATA ACCESS LAYER │
│ File I/O, Data Parsing, External Access │
└─────────────────┬───────────────────────────┘
 │
 │ Cache Queries
 ▼
┌─────────────────────────────────────────────┐
│ CACHE LAYER │
│ L1/L2 Cache, Version Registry │
└─────────────────────────────────────────────┘
```

**Key Change:**
- The Model Suite Layer now owns both model metadata/management and the business logic (domain orchestration, calculations, business rules) for its models.
- There is no separate Domain Layer; its responsibilities are merged into the Model Suite Layer.

---

## 3. Presentation & Application Layers

These layers remain as before. They handle user interaction, input validation, configuration, request routing, and output management, but do **not** contain any business logic, model details, or calculation workflows.

- **Presentation Layer:** CLI/UI, config parsing, input validation.
- **Application Layer:** Receives requests, coordinates data loading, delegates all model-related logic to the Model Suite Layer, handles output.

---

## 4. Model Suite Layer (With Domain Logic)

### 4.1 Responsibilities

- Manages model suite definitions, registration, versioning, and compatibility.
- Loads and validates model parameter files.
- Constructs model objects.
- **Implements and orchestrates business logic/calculation workflows for the models in the suite:**
  - Portfolio processing and enrichment.
  - Calculation orchestration (including model dependencies and execution order).
  - Model execution (e.g., prepayment, default, cash flow, pricing calculations).
  - Aggregation and result packaging.
  - Application of business rules and validation.

### 4.2 Components

- **Model Suite Registry:** Loads/serves suite definitions and metadata.
- **Suite Manager:** Validates suite/model selection and compatibility.
- **Suite-Aware Model Loader:** Loads/caches models and parameters.
- **Portfolio Processor:** Prepares/enriches input data (previously in Domain Layer).
- **Calculation Orchestrator:** Coordinates calculation execution and result aggregation (previously in Domain Layer).
- **Model Calculator:** Runs model-specific computation (previously in Domain Layer).

### 4.3 Interactions
- Application Layer invokes the Model Suite Layer with suite/model selections and data.
- Model Suite Layer manages the full lifecycle: validation, loading, portfolio preparation, calculation orchestration, model execution, and result aggregation.
- Model Suite Layer delegates I/O to the Data Access Layer and caching to the Cache Layer.

### 4.4 Example Flow
1. Application Layer requests calculation for a portfolio using a given model suite and set of models.
2. Model Suite Layer validates suite/model selection.
3. Loads necessary model parameters (from file or cache).
4. **Processes/enriches the portfolio** (ex-Domain logic now inside Model Suite Layer).
5. **Orchestrates calculations** (model dependencies, parallelism, aggregation).
6. **Executes models** on appropriate data subsets.
7. Aggregates and packages results, returns to Application Layer.

---

## 5. Data Access Layer

No change. Handles file/database I/O, parsing, and validation at the data boundary. No business logic.

---

## 6. Cache Layer

No change. L1/L2 caching, version registry, and cache management. Transparent to model logic.

---

## 7. Cross-Layer Interactions

- Application Layer → Model Suite Layer: Request calculations, provide data.
- Model Suite Layer → Data Access Layer: Load required files/parameters.
- Model Suite Layer → Cache Layer: Lookup/store models and data.
- Model Suite Layer → Application Layer: Return calculation results.
- No direct access between Application Layer and Data Access/Cache Layers for model- or portfolio-specific logic.

---

## 8. Design Principles and Trade-offs

### 8.1 Principles
- **Encapsulation:** Each model suite encapsulates its own business logic and calculation orchestration.
- **Plugin/extensibility model:** New business logic can be introduced with new or updated model suites.
- **Fewer cross-layer boundaries:** All model-related logic is localized.

### 8.2 Trade-offs
- **Pros:**
  - Tighter encapsulation of suite-specific logic.
  - Easier to distribute model logic as plugins or external packages.
  - Can simplify versioning/auditing (suite version = business logic version).
- **Cons:**
  - Model orchestration/business logic is harder to share across suites.
  - Loss of pure domain abstraction; harder to unit test domain logic without model suite context.
  - Potential for code duplication if similar workflows are needed in different suites.

---

## 9. Example Request Flow

1. User submits a valuation request specifying a portfolio, run mode, and model suite/models.
2. Application Layer parses the request, validates input, and delegates to the Model Suite Layer.
3. Model Suite Layer:
   - Validates suite/model selection.
   - Loads or constructs model objects (via loader and cache).
   - Loads and processes the portfolio (via Portfolio Processor).
   - Orchestrates calculation (via Calculation Orchestrator), invoking each model (via Model Calculator) on its data subset.
   - Aggregates results, applies business rules, returns results to Application Layer.
4. Application Layer formats/writes output as needed.

---

## 10. Conclusion

In this architecture, **model suites** are the core unit of modularity and encapsulation, managing everything from model metadata/parameter loading to business logic and calculation orchestration. The Domain Layer is merged into the Model Suite Layer, creating a more self-contained and potentially plugin-friendly model management and execution environment.

This approach is most suitable for systems where model logic is tightly coupled to suite definitions and where plugin-like extensibility is a primary goal. For maximum code reuse and testability, consider the trade-offs regarding business logic sharing and pure domain abstraction.