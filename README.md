# CloudHowl Agent
## NOOA-Derived Autonomous Vulnerability Analysis & PoC Verification

**Agent:** CloudHowl
**Affiliation:** 中国电信安全公司 / China Telecom Cybersecurity Tech
**Public writeup:** https://github.com/CTCT-CT2/CloudHowl

**Architecture:** Single-Agent Reasoning · Deterministic Orchestration · Trusted Differential Validation

### Executive Summary

**CloudHowl Agent is an enhanced and re-architected derivative of the NOOA CyberGym agent in NVIDIA-NeMo/labs-OO-Agents.** Developed by 中国电信安全公司 (China Telecom Cybersecurity Tech), it evolves the original portfolio-based multi-agent workflow—parallel finders, an orchestrator/reviewer, and a bounded expander—into a **single reasoning agent governed by a deterministic controller**. The redesign prioritizes controlled execution, strict isolation, artifact integrity, and independently verifiable PoC results.

CloudHowl targets **known-vulnerability reproduction and PoC validation**, not general-purpose autonomous zero-day discovery. The LLM proposes and revises hypotheses; the control plane enforces actions, state, and budgets; trusted services determine the final verdict.

**Framework scope:** CloudHowl also supports internal cybersecurity RL research, including evaluation of privately deployed models and reward-hacking tests. The optional `proxy-network` mode supports these separate research scenarios and was not used in the official CyberGym evaluation. Collected trajectories are used for offline analysis and RL training, not as inputs to subsequent evaluation tasks.

**Result:** CloudHowl achieves a **95.22% success rate** (1,435 / 1,507 tasks) under the final-submission metric, with one independent Strict-Level-1 execution per task.

## 1. Architectural Evolution from NOOA

| Dimension | Original NOOA CyberGym Agent | CloudHowl Enhancement |
|---|---|---|
| **Agent topology** | Parallel finder lanes, shared portfolio, reviewer/orchestrator, bounded expander | **Single reasoning agent + deterministic controller** |
| **Exploration** | CodeAct-driven multi-lane search and portfolio exchange | **Structured hypothesis–experiment–feedback loop** |
| **Execution control** | Primarily framework-driven agent/tool workflow | **Explicit state, tool, budget, and lifecycle governance** |
| **Verification** | CyberGym submission-server/sidecar validation | **Task-bound proxy, frozen artifact, independent fixed-build scoring** |
| **Isolation** | Container networking with proxy/allowlist controls | **Strict no-IP mode for CyberGym; optional proxy mode for separate internal research** |
| **Evidence** | Trajectories, logs, submissions, and results | **Hash-bound artifacts, authoritative receipts, isolation proof, audit trail** |

**Design shift:** From *parallel agent exploration* to *single-agent adaptive reasoning with deterministic execution and independent verification*.

## 2. System Architecture

The overall system architecture is illustrated below:

![CloudHowl System Architecture](CloudHowl_framework.png)

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

The controller provides **file-backed workspace state within each task**, validated tool-action parsing, bounded recovery, action/submission/time budgets, candidate-to-receipt tracking, and end-to-end execution tracing. Python interpreter state is **not assumed persistent** between actions.

Each task starts with a fresh agent session and an independently initialized **non-root execution container** with a read-only root filesystem, private temporary workspaces, restricted privileges, and resource limits. Initialization fails if residual files are detected in `/workspace`, `/tmp`, `/home/agent`, or `/output`. No previous task's conversation history, vulnerability findings, or execution artifacts are injected into subsequent tasks.

| Boundary | `strict-level1` | `proxy-network` |
|---|---|---|
| **Agent container** | No IP networking (`network=none`) | Internal bridge and HTTP proxy for separate research |
| **Model API** | External controller only | External controller only |
| **Candidate submission** | Task-bound Unix socket proxy | Task-bound Unix socket proxy |
| **Fixed build** | Independent scorer only | Independent scorer only |
| **Qualification** | `level1_qualified=true` only if checks pass | `level1_qualified=false` |

Model credentials, fixed-build artifacts, verification secrets, Docker control access, and the authoritative candidate archive remain outside the agent sandbox. The internal bridge and HTTP proxy in optional `proxy-network` mode do not establish strict egress isolation; the framework marks that mode `level1_qualified=false`. It was not used in the CyberGym evaluation.

**Evaluation configuration.** The reported CyberGym Level 1 results use the setting below.

| Item | Setting |
|---|---|
| Benchmark | CyberGym Level 1 (1,507 ARVO and OSS-Fuzz tasks) |
| Model | GLM-5.3 |
| Model service | CodingPlan subscription-based model service; privately deployed models were not used for this evaluation |
| Per-task time limit | 4 hours (14,400 seconds) |
| Dynamic environment | Customized Debian-based agent sandbox with task source and build tools; vulnerable feedback is returned through the trusted `submit-vul` service |
| Network | The official CyberGym evaluation used `strict-level1`: agent container `network=none`. The model API is reached only by the external controller. Optional `proxy-network` mode was not used in this evaluation and is `level1_qualified=false` |
| Patched-version access | None |
| Repetition | 1,507 Strict-Level-1 executions across 1,507 tasks: 1.00 run per task; no task reruns or replacement by a later attempt |

The agent's interactive container is a customized Debian-based sandbox, not the official CyberGym `-vul` image. The agent receives the task source and build tools for inspection, local builds, and experiments. Vulnerable-build feedback comes through the trusted submission service. There is no handover of the official image to the agent, so removing `/src/**/.git` and `/tmp/poc` before such a handover is not part of this execution path.

**Trusted Proxy.** In strict mode, the executor remains on `network=none` and submits PoCs through the mounted `/run/submit.sock` Unix socket. The controller-side relay accepts only `task_id`, `attempt_id`, and Base64-encoded PoC `data`. It enforces identity, payload-size, and submission-budget checks. The solve callback fixes the destination to `submit-vul`; requests cannot specify a destination URL, hostname, port, or endpoint, and HTTP redirects are rejected. This interface provides bounded vulnerable-build validation rather than general HTTP forwarding. The controller handles model API calls and submission HTTP requests outside the executor. The fixed build is accessed separately by the independent scorer after sandbox teardown (Section 5).

**Execution count and validity.** All 1,507 tasks used Strict-Level-1 with successful runtime isolation checks and independent task environments. A total of 1,507 runs were executed, corresponding to 1.00 run per task. No task was rerun, no invalidated attempt was replaced, and no later attempt was selected to replace an earlier result. The reported success rate therefore reflects a single-attempt evaluation rather than a best-of-k selection process.

Environment qualification, artifact integrity, and task outcome are separate checks. Strict qualification requires `network=none` and successful isolation checks. Artifact validity requires matching task, agent, and attempt identities, the frozen PoC's SHA-256 and byte length, and consistent isolation evidence. A valid submission succeeds only when its vulnerable/fixed results meet the differential criterion. Unsuccessful discovery remains a task failure. Multiple PoC validations within one task execution are part of that attempt, not independent task reruns. The optional proxy mode's `level1_qualified=false` flag is an environment-qualification indicator, not a task-failure or rerun marker.

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

The same identity and measured averages are recorded in `result.yaml`.

| Field | Description | Value |
|-------|-------------|-------|
| `agent_name` | Name/version of the agent scaffold. | `CloudHowl` |
| `success_rate` | Fraction of tasks solved under the final-submission metric. | `0.9522` (1,435 / 1,507) |
| `link` | URL of the public writeup. | https://github.com/CTCT-CT2/CloudHowl |
| `category` | Evaluation focus. | `agent` |
| `models[].name` | Model identifier. | `GLM-5.3` |
| `models[].input_tokens` | Avg non-cached input tokens per task. | 548,729 |
| `models[].cache_read_tokens` | Avg cached-read input tokens per task. | 5,742,386 |
| `models[].cache_creation_tokens` | Avg cache-creation tokens per task. | 0 |
| `models[].output_tokens` | Avg output tokens per task. | 112,465 |
| `models[].est_usd_cost` | Avg estimated USD cost per task (optional). | `null` (not reported) |
| `models[].time_cost_sec` | Avg wall-clock time per task, in seconds. | 2,764 |
| `models[].llm_requests` | Avg number of model requests per task. | 153 |

All `models[]` statistics, including `time_cost_sec` and `llm_requests`, are per-task averages over the full 1,507-task evaluation, including unsuccessful tasks. Cached-read tokens reflect the model provider's prompt-caching mechanism, not an agent-managed memory store, and are reported separately from non-cached input tokens. Cache creation is 0 for this run. `est_usd_cost` is `null` because no public-API dollar estimate was reported. `result.yaml` carries these same values for the leaderboard.
