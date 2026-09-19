# Jev (from GLiNER2) Decision Service
## Architecture and Developer Handoff Specification

**Document version:** 1.0 — English edition  
**Date:** September 19, 2026  
**Selected base model:** `fastino/gliner2-base-v1`  
**System objective:** An independent decision service with a Jev-compatible interface for `Choice`, `Score`, and `Noul`.  
**Status:** Design for implementation. No service was implemented, no model was trained, and no performance measurements were conducted as part of this document.

**Translation note:** This is the English edition of the original architecture document. It preserves the design, examples, and source references. No new external verification or runtime testing was performed for this translation.

---

## 1. Core Design Decision

Build an HTTP service that translates each question into a dynamic GLiNER2 classification task, obtains a raw score for every option, computes a probability distribution, and constructs a structured response from that distribution. Options are supplied in the request; they are not a fixed list of classes embedded in the model weights.

**No text-generating model, language-model-based JSON generation, or separate regression head for Score is required.** The model performs content-understanding decisions. Input validation, normalization, score calculation, and response construction are handled by deterministic code.

Distinguish three types of commitment:

| Compatibility level | Target | Verification method |
|---|---|---|
| Interface and field semantics | The same question types and response structures for a defined input profile | Contract tests and numerical checks |
| Service behavior | Question isolation, error handling, limits, and client compatibility | Integration tests and documented differences |
| Model quality and performance | Useful target-domain quality, appropriate calibration, and acceptable cost | Measurement on ground-truth data and specified hardware |

**Interface compatibility is not model equivalence to Jev.** There is no commitment to identical answers, probabilities, speed, or context windows. This document designs an alternative; it does not establish a reconstruction of Jev's internal architecture.

Every section describing a new component, policy, or acceptance test is a **design decision for this service**. Externally verified properties are identified by references `[S01]`–`[S13]` at the end of the document.

## 2. Technical Foundation and First-Version Boundaries

The selected checkpoint uses `microsoft/deberta-v3-base`. Its model card describes approximately 205 million parameters and labels it as an English model. GLiNER2 classification uses a contextual representation for each label and a head that produces a logit from that representation. These are starting points, not evidence of general instruction-following or reasoning capabilities equivalent to Jev. [S07, S08, S09]

The paper specifies a 2,048-token window. The service limit must be determined from the checkpoint, tokenizer, and processor actually tested. The reviewed Jev documentation specifies 32k for the state plus the longest question, and 64k for the entire request. These systems also use different tokenizers: copying token counts does not establish equivalence. [S06, S09]

| Capability | Base version | Separate extension |
|---|---|---|
| Choice / Score / Noul | All three primitives | Quality improvements through training |
| Dynamic instructions and labels | Translation into a GLiNER2 schema | Adaptation for complex instruction understanding |
| Independent questions | A separate model sample for each question | Shared state with isolated attention |
| Complete probabilities | Extraction and normalization of all logits | Separate calibration for each model version |
| Structured input | Consistent serialization without summarization | Training on complex structures |
| Long input | Explicit rejection beyond the input budget | Long-context adaptation and training |
| 255 Choice options | Subject to the actual input budget | A validated mechanism for large option sets |
| Hebrew and additional languages | Unicode accepted; quality not guaranteed | Evaluation and adaptation for each language |

**Do not silently switch to GLiNER2.5 or another checkpoint.** The current repository also contains examples for newer models. A feature in a current tutorial does not establish that it is supported by `gliner2-base-v1`. [S10]

## 3. System Architecture

```text
Client
  |
  v
HTTP Gateway + Authentication + Rate Limits
  |
  v
Request Validator + Model Resolver
  |
  v
Canonical Serializer + Question Compiler
  |
  v
Token Budget Validator
  |
  v
Question-Isolated Microbatch Scheduler
  |
  v
GLiNER2 Model Adapter -> Raw logits + option mapping
  |
  v
Probability Normalizer + Versioned Calibrator
  |
  v
Choice / Score / Noul Decoder + Confidence Profile
  |
  v
Response Validator + JSON Serializer
  |
  v
Client

Model Registry --------> Resolver / Adapter / Calibrator
Metrics ----------------> All stages, without raw request content
```

### Component Responsibilities

| Component | Input | Output and responsibility |
|---|---|---|
| `gateway` | HTTP request and credentials | Client context, request ID, limits, and request deadline |
| `contract` | JSON body | Validated request; no silent field changes |
| `compiler` | State and one question | Internal task: instructions, labels, descriptions, and output mapping |
| `budgeting` | Task and complete tokenization | Actual length including the schema; acceptance or error |
| `scheduler` | Validated tasks | Microbatches based on length, budget, and load |
| `model_adapter` | Batch | All logits for every question, without thresholding or top-k filtering |
| `probability` | Logits and masks | Normalized distribution and an identified calibration profile |
| `primitives` | Distribution and mapping | Choice / Score / Noul answers and confidence |
| `response` | Answers and usage counters | Valid JSON, with one answer for every original question ID |
| `registry` | Version manifest | Compatible weights, tokenizer, compiler, calibration, and limits |

The service does not execute the selected action. For example, `choice="refund"` is output for the calling system, not an instruction to issue a refund.

## 4. External HTTP Contract

### 4.1 Endpoint

```text
POST /v1/systemone
Authorization: Bearer <LOCAL_SERVICE_API_KEY>
Content-Type: application/json
```

The route and request structure are based on the TypeSafe interface: `state`, `model`, and `questions`. The response returns `model`, `answers`, and `usage`; every answer includes `type`. A question ID is used only for matching and is not passed to the model. [S01]

The local model name will be, for example, `gliner2-decision-base-v1.0.0`. An alias such as `jev-latest` is permitted only in an explicit migration mode. Even then, the response's `model` field must identify the actual local model. Do not send a TypeSafe API key to the local service or a local service key to TypeSafe.

### 4.2 Types and Validation Decisions

In this document, `JsonContent` means a string, JSON object, or JSON array. Objects and arrays may contain valid scalar values, including numbers, booleans, and null. Preserve the structure; do not convert every value into a string.

| Field | Rule in the proposed service |
|---|---|
| `state` | Required `JsonContent`; a number, boolean, or null at the root is not accepted |
| `model` | A string resolving to an approved registry entry |
| `questions` | A nonempty map of question IDs to questions |
| `type` | One of `choice`, `score`, or `noul` |
| `instructions` | Required `JsonContent` |
| Question ID | Returned unchanged; not included in model text or the task schema |

TypeSafe documents structured state, instructions, and criteria. Some API documentation sections present narrower types; in particular, the expanded Score example returns objects inside `legend`. Preserve the original values and test compatibility with the client's specific SDK as well. [S04, S05, S03]

The local policy is to reject unknown external fields, JSON with duplicate keys, excessive nesting, and nonfinite numbers. Do not attribute this policy to Jev without contract testing. Size and depth limits must be deployment settings, not hidden restrictions.

### 4.3 Primitives

**Choice.** `criteria` is a map from an option to its description. The service preserves structured descriptions or null; when the description is null, the option name itself supplies the content. Output includes `choice`, `probabilities`, `confidence`, and `type`. Jev documents up to 255 options. The local service policy is 1–255 options, subject to the input budget; compatibility for a single-option question requires separate testing. [S02, S05]

**Score.** `criteria` is an ordered list of 2–10 level descriptions. The position in the list determines the numerical value. Output includes `score`, `legend`, `probabilities`, `confidence`, and `type`. Level keys in JSON are strings such as `"0"`. The legend preserves the original description, including structured descriptions. [S03]

**Noul.** `criteria` is optional and contains descriptions for `true` and `false`. When a description is missing, the service generates a neutral affirmative or negative description relative to the instructions, using a fixed, documented template. Output contains `type` and `noul`: a probability between 0 and 1, not a boolean, with no separate confidence field. [S05, S11]

### 4.4 Example Request

This example illustrates the data contract only; it is not implementation code.

```json
{
  "state": {
    "message": "I was charged twice. Please refund the duplicate payment.",
    "account_status": "active"
  },
  "model": "gliner2-decision-base-v1.0.0",
  "questions": {
    "department": {
      "type": "choice",
      "instructions": "Which team should handle the request?",
      "criteria": {
        "billing": "Charges, invoices and refunds",
        "technical": "Software bugs and connectivity",
        "sales": "New purchases and upgrades"
      }
    },
    "urgency": {
      "type": "score",
      "instructions": "Evaluate how urgently the customer wants a response.",
      "criteria": [
        "No urgency expressed",
        "A timely response is requested",
        "Immediate attention is explicitly requested"
      ]
    },
    "asks_refund": {
      "type": "noul",
      "instructions": "Does the customer request a refund?",
      "criteria": {
        "true": "The customer requests money back",
        "false": "The customer does not request money back"
      }
    }
  }
}
```

### 4.5 Example Response

**All numbers below are synthetic.** They test only the response structure and formulas; they are not predictions for the example text. The token count is also illustrative.

```json
{
  "model": "gliner2-decision-base-v1.0.0",
  "answers": {
    "department": {
      "type": "choice",
      "choice": "billing",
      "probabilities": {
        "billing": 0.75,
        "technical": 0.15,
        "sales": 0.10
      },
      "confidence": 0.625
    },
    "urgency": {
      "type": "score",
      "score": 1.6,
      "legend": {
        "0": "No urgency expressed",
        "1": "A timely response is requested",
        "2": "Immediate attention is explicitly requested"
      },
      "probabilities": {
        "0": 0.10,
        "1": 0.20,
        "2": 0.70
      },
      "confidence": 0.40
    },
    "asks_refund": {
      "type": "noul",
      "noul": 0.82
    }
  },
  "usage": {
    "input_tokens": 180,
    "output_tokens": 0
  }
}
```

**Local usage semantics:** `input_tokens` is the sum of nonpadding input tokens actually processed across all question samples, including repeated state and schemas. `output_tokens=0` because the service does not generate output tokens through a decoder. This is a documented semantic difference from TypeSafe's accounting, not an attempt to imitate its billing counter. Retries and additional physical-work counters appear only in internal telemetry.

## 5. Compiling a Question into the Model Schema

### 5.1 Internal Representation

Each question becomes an internal object with the following fields. These are proposed service field names, not an API attributed to the GLiNER2 library:

| Field | Purpose |
|---|---|
| `request_id`, `question_id` | Matching and monitoring only; never passed to the model |
| `question_type` | Selection of compilation and decoding policies |
| `state_text` | Consistent representation of the state |
| `instruction_text` | Serialized question instructions |
| `options` | Internal labels, external names, and descriptions |
| `output_mapping` | Original mapping to Choice options, Score levels, or true/false |
| `token_count` | Complete sample length after processing |
| `compiler_version` | Version of the input template and structured-content handling |

### 5.2 Serialization Rules

Preserve strings without summarization or translation. Convert JSON into a consistent representation: stable object-key ordering, preserved array order, numbers retained as numbers, and no omitted fields. Present instructions and state in separate, clearly marked regions.

For Choice, establish the internal option order through stable sorting of the keys, so the order in which the JSON map was written does not change the model input order. For Score, never sort the levels: list order is part of their meaning. For Noul, use the internal order `[false, true]`.

Do not use the question ID as a semantically meaningful task name for the model. Use a fixed internal task name; the decision content must come from the instructions and criteria.

### 5.3 Using the Original Processor

The adapter must use the schema mechanism and processor from the pinned library version. Do not manually assemble special tokens based on an example in a paper. Verify that user text resembling an internal marker cannot create additional labels or modify the schema structure. Any escaping must be consistent and must not silently delete information.

**Mandatory feasibility check before service implementation:** Determine exactly how the selected version incorporates instructions and label descriptions, and how to access logits before decoding. Do not assume parameters such as `return_logits=True` or `instruction=` exist in a particular API without checking it.

If no public API returns all logits, implement a minimal adapter extension or a documented fork. Do not invent a probability distribution from the winning label and its confidence.

## 6. Baseline Inference Path

Create a separate sample for every question:

```text
sample(q1) = schema(q1) + instructions(q1) + state
sample(q2) = schema(q2) + instructions(q2) + state
sample(q3) = schema(q3) + instructions(q3) + state
```

Pack the samples into a batch, but do not share attention between them. Do not submit all questions as a single shared schema to the bidirectional encoder: their representations could then influence one another. The target is the independence documented for TypeSafe questions. [S04]

The adapter returns a logit matrix of shape `[B, K_max]` and a valid-option mask. Here, `B` is the number of question samples in the microbatch, not the number of external requests. Each row receives its own softmax, restricted to its valid options.

Run the model in evaluation mode, without gradients and with dropout disabled. The reference version must use numerically stable computation. Reduced precision or quantization requires regression testing and recalibration when necessary.

**The state is re-encoded for every question in this version.** Batching is not equivalent to processing the state once, and it does not guarantee constant latency as questions are added. Tokenization caching is allowed; caching hidden states that are not mathematically equivalent to the original computation path is not allowed.

Avoid splitting the options of one question into multiple groups and then arbitrarily normalizing their results: logits may depend on the options present in each group. Reject a request that does not fit the budget until a trained and validated path exists for that case.

## 7. Probabilities, Primitives, and Confidence

### 7.1 Normalization and Calibration

Given finite logits `z[k]` and a positive calibration parameter `T`:

```text
u[k] = z[k] / T
p[k] = exp(u[k] - max(u)) / sum_j exp(u[j] - max(u))
```

Perform this calculation in at least float32, applying the mask before normalization. Do not apply softmax a second time to probabilities, and do not normalize sigmoid scores as though they were logits. Verify that the output contains exactly all valid options, finite values, and a sum close to 1.

Start development with `T=1` and label the version `uncalibrated_baseline`. After fitting the temperature on a separate calibration set, pin the calibration together with the model weights and input template. Temperature scaling changes the probability distribution but, for a shared positive T, does not change the argmax. Do not describe it as instruction-understanding training. [S12]

### 7.2 Primitive Decoding

```text
Choice:
  choice = option with highest p
  return every option probability

Score:
  score = sum(k * p[k]) for k = 0,...,K-1
  legend[str(k)] = original criteria[k]

Noul:
  noul = p[true]
```

For an exact tie in Choice, select according to the canonical order. For Score, when a mode is needed for confidence calculation, select the lowest tied index. Do not round probabilities to two decimal places before calculating the score. A threshold for executing an action belongs to the application layer, not to Noul itself.

### 7.3 Confidence Profile

The development profile is `adapter-reference-v1`, based on the open TypeSafe adapter algorithm retrieved during the original conversation at this commit:

```text
adffc2eab300a4fa3c0e92252d4ffd6ceaa53700
src/system_one_adapter/_utils/confidence_metrics.py
```

**This is a separate reference, not proof of the numerical formula used by the current Jev service.** The official documentation promises confidence derived from the distribution, but that page does not publish a binding formula. Compare real responses against a pinned version before claiming numerical compatibility. [S01, S13]

For Choice:

```text
K = number of valid options
if K = 1: confidence = 1
otherwise:
  confidence = (max(p) - 1/K) / (1 - 1/K)
```

For Score:

```text
m = first argmax(p)
D = (1/K) * sum(abs(k - (K-1)/2)) for k = 0,...,K-1
confidence = max(0, 1 - sum(p[k] * abs(k-m)) / D)
```

The decoder must receive only valid distributions. A numerical failure must not be converted into a uniform distribution or a Noul value of 0.5: return an error instead. A tiny numerical correction is permitted only after validation, with a documented tolerance.

| Test profile | Distribution | Expected result |
|---|---|---|
| Choice | `[0.75, 0.15, 0.10]` | confidence = `0.625` |
| Choice | `[1/3, 1/3, 1/3]` | confidence = `0` |
| Choice | `[1]` | confidence = `1` |
| Score | `[0.10, 0.20, 0.70]` | score = `1.6`; confidence = `0.4` |
| Score | `[0, 1, 0]` | score = `1`; confidence = `1` |
| Score | `[0.5, 0, 0.5]` | score = `1`; confidence = `0` |

Confidence describes the shape of a distribution. It is not a verifier, proof of correctness, or a guarantee of calibration.

## 8. Limits, Errors, and Atomicity

Calculate input length **after** inserting the instructions, labels, descriptions, and processor markers. Do not measure the state alone. A request with 255 options may satisfy the option-count limit while exceeding the token budget.

Requests are atomic: all questions must pass validation and budgeting before inference; a successful response returns every answer. An inference failure must not be returned as HTTP 200 with missing questions. Return `request_id` in a header. An error must include a code, a safe message, and the field path, without copying sensitive content into logs.

| Condition | Proposed status | Internal code |
|---|---|---|
| Invalid JSON / duplicate key | 400 | `invalid_json` |
| Missing or invalid authentication | 401 | `unauthorized` |
| Request body exceeds transport-size limit | 413 | `request_too_large` |
| Invalid schema, model, or input budget | 422 | `validation_error` / `context_limit_exceeded` |
| Per-client usage limit | 429 | `rate_limited` |
| Full queue or temporary overload | 529 in compatibility mode | `overloaded` |
| Model failure or nonfinite values | 500 | `inference_error` |
| Request deadline exceeded | 504 | `deadline_exceeded` |

TypeSafe documents 401, 422, 429, and 529. The remaining mappings and exact error body are service design decisions until tested against the client contract. [S01]

No silent truncation, automatic summarization of long inputs, fabricated results after failure, or fallback to an external API without explicit enablement and authorization is permitted.

## 9. Extension A — Shared-State Processing

This extension is not required for the base version and is not a simple cache on top of the original checkpoint. It changes the attention path and requires adaptation and reevaluation.

### 9.1 Proposed Attention Mask

| Token group | Allowed to read |
|---|---|
| State | State only |
| Question `q_i`, including instructions and options | State and the branch belonging to `q_i` |
| Question `q_i` | Must not read `q_j` when `i != j` |

Preventing state tokens from reading questions is essential: otherwise, they could transfer information between branches in later layers. Position IDs, relative positions, pooling, and normalization must also be defined so that adding a branch does not change an existing branch.

Process the state once across the layers. At each layer, branches read the corresponding state representations and update only their own representations. Reusing K/V and other representations depends on the DeBERTa implementation and positional relationships; verify it numerically against a reference implementation using the same mask.

For illustration only, the dense-attention component's approximate cost changes from `sum_i (S + Q_i)^2` to `S^2 + sum_i (Q_i*S + Q_i^2)`, where S is the state length and Q_i is the length of a question branch. This expression excludes other model costs and is not a guarantee of hardware speedup.

### 9.2 Acceptance Conditions

Compare the shared-state path with a reference path using the same mask, test branch independence, and train or adapt the model to the changed context it observes. Do not require numerical equality with the original bidirectional model after changing the mask.

Where state caching is used, its identity must depend on the client, model version, compiler, tokenizer, positional configuration, and complete content. Cross-client caching is disabled by default.

## 10. Extension B — Long Context and Large Option Sets

The objective is to support questions that depend on distant information within the same state, not merely to accept large text through HTTP.

The preferred research direction is to extend the encoder with local and global attention and a path for exchanging information between distant regions, while preserving isolated question branches. This requires positional adaptation, training at varying lengths, and cross-document-region reasoning tests. Simply setting `max_length` to 32k is insufficient.

Chunking and independently classifying each chunk may serve only as an experimental baseline. Combining results with a maximum or average is not equivalent to making a decision after reading the entire context. An important acceptance case is a rule at the beginning of a document, a fact in the middle, and an exception at the end, where only their combination determines the answer.

For large option sets, test both criteria length and the ability to distinguish between options. Separate option branches with combined logits are a possible architectural change, but require training and evidence that their scores are comparable. Do not use this shortcut in the base version.

Publish limits according to the local tokenizer, results by input length, and success rates on long-range tasks. Accepting a large input alone does not satisfy the compatibility objective.

## 11. Training and Evaluation Plan

### 11.1 Work Sequence

First measure the baseline without changing the weights. Then perform supervised adaptation for instruction-conditioned decision tasks, followed by separate calibration. Retain the baseline as a reference and do not assume that fine-tuning necessarily improves generalization.

A training sample is:

```text
(state, instructions, criteria, question_type) -> target option distribution
```

The base objective is cross-entropy over valid options. Use a one-hot target for a single label, or a soft target when a reliable target distribution exists. For Score, base training still classifies a level; compute the expectation afterward. An additional ordinal loss is a separate experiment, not an API requirement.

Reproducing a proprietary training method is not required to implement the primitives. There is also no basis for guaranteeing that the proposed training will reproduce Jev's performance.

### 11.2 Dataset Composition

Include different instructions over the same state, negation, conditions, similar labels, long descriptions, structured JSON, missing information, distractors, and cases belonging to none of the options. When an `other` or `unknown` option is needed, it must be part of the client-requested schema; do not add a hidden output class.

Include paraphrased instructions and labels, along with schemas of different sizes. Multistep logical tests must form a separate evaluation group: the existence of a Noul primitive does not establish the ability to solve them.

Use data that the system owner is authorized to use. Jev outputs may be used for authorized comparison; using them for training or distillation is not part of the default plan and requires review of permissions and applicable terms.

### 11.3 Splits and Leakage Prevention

Maintain separate `train`, `validation`, `calibration`, and `test` splits. Variations of the same state or original sample must not be split between training and testing. Also maintain groups of domains and schemas unseen during training. The calibration set is not the final test set.

Use English as the baseline evaluation group. Hebrew or another language requires separate metrics and must not be enabled through silent translation. Replacing the backbone with a multilingual model is a new design decision, not a transparent change to the existing version.

### 11.4 Metrics

| Objective | Proposed metric |
|---|---|
| Correct selection | Accuracy and macro-F1 by domain and number of options |
| Distribution quality | NLL and Brier score; explicitly document the Brier definition |
| Calibration | Reliability diagrams and ECE with a predefined binning scheme |
| Score | MAE against the appropriate target, alongside distribution metrics |
| Abstention policy | Risk/coverage using a threshold selected on the validation set |
| Comparison with Jev | Answer agreement and distribution differences, in addition to ground-truth evaluation |
| Performance | p50/p95/p99 latency, throughput, memory, and errors under a defined load |

Improved agreement with Jev does not necessarily mean improved correctness. Report quality metrics with sample sizes and confidence intervals based on example groups, not just a single overall average.

## 12. Operations, Security, and Observability

### 12.1 Initial Deployment

Use a single service package with local model weights and a configured inference worker. No database is required to make decisions. The gateway and worker may initially run in the same container, but the internal inference interface must be separate to allow later scaling.

The scheduler must use a bounded queue and microbatches based on length and memory budget. Limit pending requests, questions per request, request-body size, JSON nesting depth, and queue wait time. These values are deployment policies to be measured on target hardware, not copied from TypeSafe's limits.

`/health/live` checks that the process is running. `/health/ready` confirms that the model, tokenizer, calibration, and manifest have been loaded and validated. Failure to load a version or a mismatch between artifacts must not silently trigger an alternative model.

### 12.2 Sensitive Information

Defaults: do not log state, instructions, or answers; make no external calls during inference; do not download weights during a request; and do not execute URLs, code, or instructions contained in the state. Telemetry includes lengths, timings, error codes, and versions, without raw content.

Prepare weights and tokenizer files in advance from an approved source. Check crash dumps, traces, and caches as well, so they cannot bypass the logging policy. This is an engineering policy to verify in deployment, not a promise of absolute privacy.

### 12.3 Measurement and Reproducibility

Every response must be attributable to a version of the weights, compiler, tokenizer, calibration, and confidence profile. Store this information in the manifest and telemetry; do not add undocumented fields to the response body in compatibility mode. Capabilities may be exposed through a protected internal endpoint.

Measure tokenization, queue wait, inference, post-processing, and serialization separately. Do not present model-only latency as end-to-end service latency.

## 13. Versioning and Compatibility Contract

In the first phase, the developer must create a `compatibility_manifest` containing:

| Item | What to pin |
|---|---|
| `service_version` | Local service version |
| `model_revision` | Full checkpoint revision, not `main` |
| `runtime_lock` | Actual GLiNER2, Transformers, PyTorch, and dependency versions |
| `tokenizer_revision` | Tokenizer and processor files |
| `compiler_version` | Instruction templates, label ordering, and serialization |
| `calibration_id` | Calibration file and whether the version remains a baseline |
| `confidence_profile` | `adapter-reference-v1` or another validated profile |
| `verified_limits` | Complete input, questions, options, bytes, and JSON depth |
| `reference_contract` | Dated snapshot of TypeSafe documentation and the tested SDK version |
| `release_status` | `experimental`, `pilot`, or `production`, according to test results |

The reviewed TypeSafe documentation lists `jev-1.13.0`, but an alias such as `jev-latest` can change. During a live comparison, record the actual model identifier returned by the service. [S06]

This document does not pin an untested library version or model-weight revision. Do not copy `transformers_version` from a configuration file and automatically treat it as the development environment's lockfile.

## 14. Acceptance Test Plan

### 14.1 Deterministic and Contract Tests

| ID | Scenario | Pass condition |
|---|---|---|
| C01 | Request containing all three question types | One answer per ID; matching type |
| C02 | Change only a question ID | Identical model input; only the response key changes |
| C03 | Add an unrelated question | Original question prediction remains unchanged within numerical tolerance |
| C04 | Reorder questions / split the batch | No semantic change in predictions |
| C05 | Choice maps with different key orders | Identical canonical schema and output |
| C06 | Change the order of Score levels | Mapping and score respect the new order |
| C07 | Structured JSON and Unicode | Values, structure, and legend preserved without loss |
| C08 | Logit fixtures | Softmax applies only to valid labels; probabilities sum to 1 |
| C09 | Confidence fixtures | Matches the formula test table in this document |
| C10 | Exceed length / option-count limits | Explicit error; no truncation |
| C11 | NaN / Inf / invalid mapping | Safe failure; no fabricated probabilities |
| C12 | Load, timeout, cancellation | No partial HTTP 200; no leaked tasks or memory |
| C13 | Internal markers inside user text | No additional labels or control over the model schema |
| C14 | Two client identities | No mixing of caches, answers, or permissions |
| C15 | Model alias / registry failure | Truthful model identity; no silent fallback |
| C16 | Existing client or SDK | Response parsing, headers, and errors conform to the tested contract |

The proposed tolerance for mathematical tests in float64 is `1e-10`. For probability sums in a service response, use `1e-6`. Repeated-inference tolerances must be derived from the dtype and backend and pinned in the report. Bitwise identity across hardware is not guaranteed.

### 14.2 Release Gates

All deterministic contract and safety tests must pass. Measure quality, risk, and latency on a defined workload before a production release. Since no target domain, hardware, or quality thresholds were supplied, this document does not invent accuracy or latency targets.

Release the first service version as `experimental`, for development and evaluation only. Moving to pilot or production requires an `acceptance_manifest` containing the dataset, hardware, quality thresholds, failure policy, and actual results. This is a release requirement, not a blocker to implementing the base version.

## 15. Development Phases and Deliverables

| Phase | Work | Deliverable required to proceed |
|---|---|---|
| P0 — Pin the foundation | Load the checkpoint, identify logits, verify processor behavior and length, and freeze the contract | Manifest, feasibility report, and reference examples |
| P1 — Base service | Three primitives, isolation, normalization, errors, and metrics | Local service with tests C01–C16 |
| P2 — Quality and calibration | Baseline, decision data, controlled fine-tuning, and calibration | Evaluation report and reproducible model version |
| P3 — Shared state | New mask, reference implementation, model adaptation, and measurement of savings | Validated optional extension |
| P4 — Long context | Appropriate attention, training, and evaluation on distant information and large criteria sets | New limits demonstrated experimentally |

Do not make P1 depend on completing P3 or P4. However, do not describe P1 as a full replacement for Jev's input limits and performance.

### Proposed Project Structure

This maps future responsibilities; it is not code that has already been created:

```text
project/
  service/
    api/
    contract/
    compiler/
    model_adapter/
    scheduling/
    probability/
    primitives/
    registry/
    observability/
  configs/
    compatibility_manifest
    deployment_limits
  artifacts/
    model_manifest
    calibration
  tests/
    contract/
    numerical/
    isolation/
    integration/
    security/
  evaluation/
    datasets_manifest
    benchmark_protocol
    reports/
  training/
    data_contract
    experiment_configs
  docs/
    architecture
    compatibility_matrix
    operations_runbook
```

## 16. Mandatory Developer Decisions

**Use the selected GLiNER2 model as the foundation, not an alternative generative model.** Use its dynamic classification head and all logits, keeping the library API isolated inside an adapter. Return complete probability distributions, calculate Score as an expectation, and preserve Noul as a probability.

**Build isolated questions first.** Do not combine questions in a shared schema and call that isolation. Shared state and long context are explicit extensions requiring training and testing, not transparent optimizations.

**Do not hide gaps through post-processing.** Do not replace model errors with fabricated output, invent confidence, truncate inputs, add a hidden option, or call Jev behind the scenes. Numerical compatibility and quality require measurement separate from JSON compatibility.

**First requested deliverable:** An independent service that can be run and evaluated, with all three primitives, a pinned checkpoint and dependency versions, contract tests, a baseline report, and a documented gap analysis. Not a declared clone of Jev, and not a performance guarantee before experimentation.

---

## 17. Sources and Verification Record

The original document records that the following official sources were reviewed on September 19, 2026 to define the specification. They support the external contracts and baseline properties; they do not endorse the proposed service design. The JSON snippets and numerical examples were authored for this document. This English translation retains that record and does not claim a new verification of those sources.

**[S01] TypeSafe — API reference.** Request and response structure, question IDs, and documented error statuses.  
`https://docs.typesafe.ai/api`

**[S02] TypeSafe — Choice.** Selection, complete probability distributions, and the 255-option limit.  
`https://docs.typesafe.ai/primitives/choice`

**[S03] TypeSafe — Score.** 2–10 levels, expectation over indices, and structured legends.  
`https://docs.typesafe.ai/primitives/score`

**[S04] TypeSafe — State.** Shared state, structured content, and question independence.  
`https://docs.typesafe.ai/concepts/state`

**[S05] TypeSafe — Advanced: structure.** Structured instructions and criteria.  
`https://docs.typesafe.ai/primitives/advanced`

**[S06] TypeSafe — Models.** Reported service version, aliases, and context limits.  
`https://docs.typesafe.ai/models`

**[S07] Fastino — model card.** Selected checkpoint, model description, and language tag.  
`https://huggingface.co/fastino/gliner2-base-v1`

**[S08] Fastino — checkpoint config.** `model_name: microsoft/deberta-v3-base`; the version recorded in the file is not a lockfile.  
`https://huggingface.co/fastino/gliner2-base-v1/blob/main/config.json`

**[S09] GLiNER2 paper, arXiv:2507.18546v1.** Encoder, dynamic classification, per-label logits, and the window specified in the paper.  
`https://arxiv.org/html/2507.18546v1`

**[S10] Fastino — GLiNER2 repository.** Library repository; also includes models and features newer than the selected checkpoint.  
`https://github.com/fastino-ai/GLiNER2`

**[S11] TypeSafe — Noul.** Probability of an affirmative answer, optional criteria, and no separate confidence field.  
`https://docs.typesafe.ai/primitives/noul`

**[S12] Guo et al., On Calibration of Modern Neural Networks, ICML 2017.** Reference for temperature scaling.  
`https://proceedings.mlr.press/v70/guo17a.html`

**[S13] TypeSafe — confidence reference.** The official page explains that confidence is derived from the distribution. The development formulas in Section 7 are based on the adapter file at the commit below, whose content was retrieved earlier in the original conversation. A subsequent browser retrieval attempt failed; verify the file and its agreement with the service during P0. This does not verify Jev's internal formula.  
`https://docs.typesafe.ai/confidence`  
`https://github.com/typesafe-ai/system-one-adapter-python/blob/adffc2eab300a4fa3c0e92252d4ffd6ceaa53700/src/system_one_adapter/_utils/confidence_metrics.py`

**Handoff note:** No live Jev call or GLiNER2 execution was performed as part of this document. Do not interpret the examples as model results, and do not mark acceptance gates as passed before running the tests.
