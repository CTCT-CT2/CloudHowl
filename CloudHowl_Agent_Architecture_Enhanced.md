# CloudHowl Agent
## NOOA-Derived Autonomous Vulnerability Analysis & PoC Verification

**Agent:** CloudHowl
**Affiliation:** 中国电信安全公司 / China Telecom Cybersecurity Tech
**Public writeup:** https://github.com/CTCT-CT2/CloudHowl

**Architecture:** Single-Agent Reasoning · Deterministic Orchestration · Trusted Differential Validation

### Executive Summary

**CloudHowl Agent is an enhanced and re-architected derivative of the NOOA CyberGym agent in NVIDIA-NeMo/labs-OO-Agents.** Developed by 中国电信安全公司 (China Telecom Cybersecurity Tech), it evolves the original portfolio-based multi-agent workflow—parallel finders, an orchestrator/reviewer, and a bounded expander—into a **single reasoning agent governed by a deterministic controller**. The redesign prioritizes controlled execution, strict isolation, artifact integrity, and independently verifiable PoC results.

CloudHowl targets **known-vulnerability reproduction and PoC validation**, not general-purpose autonomous zero-day discovery. The LLM proposes and revises hypotheses; the control plane enforces actions, state, and budgets; trusted services determine the final verdict.

**Result:** CloudHowl achieves a **95.22% success rate** on CyberGym under the final-submission metric.

## 1. Architectural Evolution from NOOA

| Dimension | Original NOOA CyberGym Agent | CloudHowl Enhancement |
|---|---|---|
| **Agent topology** | Parallel finder lanes, shared portfolio, reviewer/orchestrator, bounded expander | **Single reasoning agent + deterministic controller** |
| **Exploration** | CodeAct-driven multi-lane search and portfolio exchange | **Structured hypothesis–experiment–feedback loop** |
| **Execution control** | Primarily framework-driven agent/tool workflow | **Explicit state, tool, budget, and lifecycle governance** |
| **Verification** | CyberGym submission-server/sidecar validation | **Task-bound proxy, frozen artifact, independent fixed-build scoring** |
| **Isolation** | Container networking with proxy/allowlist controls | **Strict no-IP mode or explicitly governed proxy mode** |
| **Evidence** | Trajectories, logs, submissions, and results | **Hash-bound artifacts, authoritative receipts, isolation proof, audit trail** |

**Design shift:** From *parallel agent exploration* to *single-agent adaptive reasoning with deterministic execution and independent verification*.

## 2. System Architecture

The system architecture is summarized in the layer table below.

| Layer | Core Modules | Responsibility |
|---|---|---|
| **Task Context** | Vulnerability · Repository · Harness · Constraints | Normalize the target, entry points, and validation rules. |
| **Reasoning Agent** | Code Intelligence · Root Cause · Reachability · PoC Synthesis · Feedback Analysis · Candidate Selection | Form and refine testable vulnerability hypotheses. |
| **Tool Runtime** | Shell · Python · Submit · Final | Inspect code, construct inputs, obtain execution feedback, finalize an eligible candidate. |
| **Control Plane** | State · Orchestration · Budgets · Candidate Ledger · Trace | Enforce deterministic policies across long-running tasks. |
| **Trusted Verification** | Submission Proxy · Artifact Freeze · Independent Scorer | Bind the final input to evidence and validate vulnerable/fixed behavior. |

**Execution path:** `Task Context → Reasoning Agent ⇄ Tool Runtime → Eligible Submission → Artifact Freeze → Independent Differential Scoring`

**Reasoning cycle:** `PLAN → ACT → OBSERVE → REFINE → SELECT`

## 3. Autonomous Reasoning & Tooling

**Specialized reasoning:**

- **Source Code Intelligence:** interprocedural inspection, call-path and harness analysis.
- **Root Cause & Reachability:** fault conditions, input propagation, path constraints.
- **PoC Synthesis:** format-aware input construction and iterative refinement.
- **Feedback Interpretation:** sanitizer output, crash traces, exit-status analysis.
- **Candidate Selection:** evidence-based comparison and final eligible-input selection.

**Tool contracts:** **Shell** (inspect/build/debug) · **Python** (generate/test inputs) · **Submit** (task-bound vulnerable-build feedback) · **Final** (select a previously submitted eligible PoC).

**Authority model:** Model outputs are proposals; tool results, submission receipts, and scorer verdicts are authoritative.

## 4. Deterministic Harness & Isolation

The controller provides **persistent file-backed workspace state**, validated tool-action parsing, bounded recovery, action/submission/time budgets, candidate-to-receipt tracking, and end-to-end execution tracing. Python interpreter state is **not assumed persistent** between actions.

Each task executes in a constrained **non-root sandbox** with a read-only root filesystem, private writable workspace, restricted privileges, and resource limits.

| Boundary | `strict-level1` | `proxy-network` |
|---|---|---|
| **Agent container** | No IP networking (`network=none`) | Controlled proxy-mediated networking |
| **Model API** | External controller only | External controller only |
| **Candidate submission** | Task-bound Unix socket proxy | Task-bound Unix socket proxy |
| **Fixed build** | Independent scorer only | Independent scorer only |
| **Qualification** | `level1_qualified=true` only if checks pass | `level1_qualified=false` |

Model credentials, fixed-build artifacts, verification secrets, Docker control access, and the authoritative candidate archive remain outside the agent sandbox. Proxy mode requires enforced egress ACLs; proxy configuration alone does not establish isolation.

**Evaluation configuration.** The reported CyberGym Level 1 results use the setting below.

| Item | Setting |
|---|---|
| Benchmark | CyberGym Level 1 (1,507 ARVO and OSS-Fuzz tasks) |
| Model | GLM-5.3 |
| Per-task time limit | 4 hours (14,400 seconds) |
| Dynamic environment | Official CyberGym task-level vulnerable images; no patched image |
| Network | `strict-level1`: agent container `network=none`. The model API is reached only by the external controller. `proxy-network` is `level1_qualified=false` and is not included in these results |
| Patched-version access | None |
| Repetition | One valid run per task |

Reported results use `strict-level1` only. The agent container has no IP networking (`network=none`). `proxy-network` remains `level1_qualified=false` and is excluded from this score. The agent runs, fuzzes, and debugs the official vulnerable image inside the task container. It does not receive the patched image. Before that image is handed over, `/src/**/.git` (repository git history) and `/tmp/poc` (the reference PoC) are removed. The fixed build is used only by the independent scorer after sandbox teardown (see Section 5). The model API is called by the external controller, not from inside the agent container. Candidate submission uses the task-bound socket proxy.

## 5. Trusted Differential Verification

1. **Submit:** The task/attempt-bound proxy records candidate bytes, quota events, and vulnerable-build receipts.
2. **Select:** `Final` identifies one previously submitted, eligible candidate.
3. **Freeze:** SHA-256 **and byte length** bind the final PoC to archived submission bytes; further submissions stop.
4. **Teardown:** The agent sandbox is removed before fixed-build evaluation.
5. **Score:** An independent verifier replays the **identical frozen bytes** against the fixed build.

**Pass criterion:** Eligible vulnerable-build crash **and** fixed-build exit code `0`, with successful identity, integrity, and evidence checks. Vulnerable exits `0`, `71`, `124`, `137`/`-9`, and `300` are excluded under the documented rules. A crash is evidence of anomalous behavior, **not automatically proof of the intended root cause**.

## 6. Evidence & Auditability

| Deliverable | Authoritative Record |
|---|---|
| **Frozen PoC** | Input bytes · SHA-256 · Byte length |
| **Run Metadata** | Task/attempt IDs · Runtime and image configuration |
| **Agent Trace** | Model proposals · Accepted actions · Tool outputs |
| **Submission Ledger** | Candidate snapshots · Receipts · Quota events |
| **Isolation Evidence** | Network policy checks · Sandbox lifecycle |
| **Verification Result** | Independent fixed-build receipt · Differential verdict |

## 7. Key Enhancements

**01 · Simplified Agent Topology**
Single-agent reasoning replaces portfolio-based multi-agent coordination, with deterministic orchestration outside the model.

**02 · Evidence-Grounded Reasoning**
Structured source analysis, reachability inference, runtime diagnosis, and candidate refinement form a closed loop.

**03 · Stronger Trust Boundaries**
Task-bound submission, isolated execution, controlled networking, and independent fixed-build access.

**04 · Reproducible Verification**
SHA-256/length-bound artifact freeze, post-teardown scoring, and auditable differential evidence.


## 8. Submission Report

**Agent:** CloudHowl
**Affiliation:** 中国电信安全公司 / China Telecom Cybersecurity Tech
**Public writeup:** https://github.com/CTCT-CT2/CloudHowl

The leaderboard icon is `cloudhowl.icon.png` (1254×1254 PNG). The same identity and measured averages are recorded in `result.yaml`.

| Field | Description | Value |
|-------|-------------|-------|
| `agent_name` | Name/version of the agent scaffold. | `CloudHowl` |
| `success_rate` | Fraction of tasks solved under the final-submission metric. | `0.9522` (1,435 / 1,507) |
| `link` | URL of the public writeup. | https://github.com/CTCT-CT2/CloudHowl |
| `category` | Evaluation focus. | `agent` |
| `models[]` | One entry per model the agent invoked. | `GLM-5.3` |
| `models[].name` | Model identifier. | `GLM-5.3` |
| `models[].input_tokens` | Avg non-cached input tokens per task. | 548,729 |
| `models[].cache_read_tokens` | Avg cached-read input tokens per task. | 5,742,386 |
| `models[].cache_creation_tokens` | Avg cache-creation tokens per task. | 0 |
| `models[].output_tokens` | Avg output tokens per task. | 112,465 |
| `models[].est_usd_cost` | Avg estimated USD cost per task (optional). | `null` (not reported) |
| `models[].time_cost_sec` | Avg wall-clock time per task, in seconds. | 2,764 |
| `models[].llm_requests` | Avg number of model requests per task. | 153 |

All `models[]` statistics are per-task averages over the full 1,507-task evaluation. Cache-read tokens are reported separately from non-cached input tokens. Cache creation is 0 for this run. `est_usd_cost` is `null` because no public-API dollar estimate was reported. `result.yaml` carries these same values for the leaderboard.

**Provenance:** Based on the NOOA CyberGym example in `NVIDIA-NeMo/labs-OO-Agents`; enhanced and re-architected by 中国电信安全公司 (China Telecom Cybersecurity Tech). Retain upstream attribution and applicable license notices in redistributed code and documentation.

> **CloudHowl Principle:** Autonomous reasoning. Deterministic execution. Independent verification.
