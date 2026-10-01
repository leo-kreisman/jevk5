# Local coding stack: portable implementation walkthrough

Updated 2026-10-01. Home repository: <https://github.com/leo-kreisman/jevk5>.

## 1. Goal and actual status

Build a local coding assistant that spends its context and computation on the source needed to solve an issue:

```text
Issue + repository revision
  -> CodeGraph: locate symbols, callers, dependencies and relevant tests
  -> JevGrep + local JevK5: resolve ambiguous relevance when needed
  -> MiniCPM: diagnose, request further evidence and edit
  -> Tests: evaluate the patch and identify regressions
  -> Store outcome + source revision + supporting evidence
  -> Continue or return the tested patch
```

**This is an implementation and handoff guide, not a claim that the integrated application already runs.** The JevK5 model release is published and checksum verified. The combined stack, its memory footprint and its coding success rate have not been benchmarked. No SWE-bench score or speedup is established for this combination.

| Piece | Role | What is available / what remains |
|---|---|---|
| CodeGraph | Structural source retrieval | Upstream MCP server; integration with the chosen agent host remains |
| JevGrep | Fragment search and relevance orchestration | Upstream CLI/MCP; local provider and graph-candidate connection need validation/implementation |
| JevK5 GGUF | Bounded relevance decisions | Published weights and upstream Python GGUF readout; typed HTTP bridge still needed for JevGrep |
| MiniCPM5-2B | Diagnosis and patch generation | Upstream weights/serving instructions; local tool parsing and agent loop need validation |
| Agent host | Connect models, MCP, edits and tests | Must be selected or implemented; neither a model server nor mini-AGI supplies this complete integration |
| Evidence store | Reuse verified findings across turns/machines | Proposed versioned records below; application integration remains |

The intended graph project is **[codegraph-ai/CodeGraph](https://github.com/codegraph-ai/CodeGraph)**. The Reddit JevGrep demonstration describes the same JevGrep component, not an additional service.

## 2. Service boundaries and budgets

Use explicit local endpoints so each piece can be tested independently. These ports are proposed defaults, not existing deployed services:

| Endpoint / process | Interface | Consumer |
|---|---|---|
| `127.0.0.1:8081` | JevK5 llama-server `/tokenize` and `/completion` | Python `JevK5GGUF` readout |
| `127.0.0.1:8082` | Proposed typed-decision bridge | JevGrep local provider |
| `127.0.0.1:8083/v1` | MiniCPM chat completion endpoint | Agent host |
| CodeGraph | MCP over stdio | Agent host / retrieval coordinator |
| JevGrep | MCP over stdio or CLI | Agent host / retrieval coordinator |
| Evidence directory or SQLite | Revisioned local records | Host, later sessions and evaluation tools |

Start with one active issue, one scoring request at a time, one test worker and modest CPU thread counts. A small model file does not guarantee that two model servers, their contexts, an index and tests fit together.

The original target remains **8 GiB aggregate inference memory**. Record model mappings, file cache, contexts, graph/index memory, host and tool processes. If test execution is outside that budget, report its peak separately and say so. On Linux, a parent cgroup can enforce the service budget; per-process RSS is insufficient and may double-count shared pages. macOS requires a separate measured memory-pressure experiment; these commands do not enforce a hard macOS ceiling.

On Apple Silicon, GPU allocations consume unified memory. On NVIDIA PCs, report VRAM separately from system RAM. An accelerated run is a separate configuration from the CPU-only target. The desktop's 64 GB RAM and two 16 GB GPUs do not demonstrate an 8 GiB fit.

If concurrent residency fails the budget, test sequential model residency and include reload latency. Do not silently turn swapping into an apparent fit. Keep operating-system and interactive-desktop headroom outside the service budget.

## 3. Prepare a portable working directory

Commands below use Bash on macOS/Linux, or an Ubuntu WSL2 terminal on Windows. Native Windows builds exist for some components, but this walkthrough does not validate a native PowerShell installation.

Prerequisites: Git, Python 3.11+, curl, CMake and a C/C++ compiler; Rust for a source build of CodeGraph; Node 24+ and npm for JevGrep. Use a recent llama.cpp build supporting the selected models. Pin working revisions after the initial compatibility checks.

```bash
git clone https://github.com/leo-kreisman/jevk5.git
cd jevk5
export STACK_ROOT="$PWD"
mkdir -p models/jevk5 models/minicpm vendor evidence runs
python3 -m venv .venv
source .venv/bin/activate
```

Set `STACK_ROOT` again in new terminals. Set `TARGET_REPO` to the absolute path of the repository being repaired; it is independent of this stack checkout. Keep model binaries, virtual environments, indexes and private task evidence out of Git. Share only the artifacts you intentionally choose.

## 4. Download and reconstruct the published JevK5 model

[Published release](https://github.com/leo-kreisman/jevk5/releases/tag/model-jevk5-4b-v0.3-q4-k-m): `jevk5-4b-v0.3-Q4_K_M.gguf`.

- Final size: **2,708,804,000 bytes**.
- SHA-256: **`94ca0d7745c47f79091b0892ca657c81d9dc9e4ed0238ba0a7ea261d8938c882`**.
- Original Hugging Face revision: `ec67b0bfce5119a8b11a2cdb430bb43e3fa3e82a`.
- The three assets are **raw byte pieces**. Concatenate them before loading; they are not independent GGUF model shards.
- Allow roughly 7 GB free disk for downloading and reconstruction. Retaining both parts and the assembled file occupies about 5.42 GB, before other stack assets.

```bash
cd "$STACK_ROOT/models/jevk5"
release_url='https://github.com/leo-kreisman/jevk5/releases/download/model-jevk5-4b-v0.3-q4-k-m'
for asset in \
  jevk5-4b-v0.3-Q4_K_M.gguf.part001 \
  jevk5-4b-v0.3-Q4_K_M.gguf.part002 \
  jevk5-4b-v0.3-Q4_K_M.gguf.part003 \
  SHA256SUMS.parts SHA256SUMS model-manifest.json \
  LICENSE NOTICE UPSTREAM-MODEL-CARD.md RELEASE-NOTES.md; do
  curl --fail --location --retry 5 --continue-at - \
    "$release_url/$asset" --output "$asset" || exit 1
done
```

The following reconstruction works on macOS without GNU `sha256sum`, uses bounded buffers and verifies the final pinned digest. Run in the download directory:

```bash
python3 - <<'PY'
import hashlib
import json
from pathlib import Path

expected = '94ca0d7745c47f79091b0892ca657c81d9dc9e4ed0238ba0a7ea261d8938c882'
name = 'jevk5-4b-v0.3-Q4_K_M.gguf'
manifest = json.loads(Path('model-manifest.json').read_text())
assert manifest['filename'] == name and manifest['sha256'] == expected
assert [p['name'] for p in manifest['parts']] == [name + f'.part{i:03}' for i in range(1, 4)]

def digest(path):
    h = hashlib.sha256()
    with path.open('rb') as f:
        for block in iter(lambda: f.read(1024 * 1024), b''):
            h.update(block)
    return h.hexdigest()

out = Path(name)
if out.exists():
    if out.stat().st_size != 2708804000 or digest(out) != expected:
        raise SystemExit('Existing assembled file failed verification; inspect it before replacing.')
    print('Already verified:', out)
else:
    for part in manifest['parts']:
        p = Path(part['name'])
        assert p.stat().st_size == part['size'] and digest(p) == part['sha256'], p
    tmp = Path(name + '.assembling')
    with tmp.open('wb') as dst:
        for part in manifest['parts']:
            with Path(part['name']).open('rb') as src:
                for block in iter(lambda: src.read(1024 * 1024), b''):
                    dst.write(block)
    assert tmp.stat().st_size == 2708804000 and digest(tmp) == expected
    tmp.replace(out)
    print('Reconstructed and verified:', out)
PY
```

Keep the release's license, notice and provenance with redistributed weights. `git clone` downloads documentation; it does not download release assets.

## 5. Bring up each model independently

Install or build [llama.cpp](https://github.com/ggml-org/llama.cpp). For a reproducible source build, record the chosen commit and follow its platform build instructions. Use CPU first; a Metal/CUDA build can be evaluated separately. Set `LLAMA_SERVER` to the absolute executable path.

Example initial JevK5 service, foreground in its own terminal:

```bash
"$LLAMA_SERVER" \
  -m "$STACK_ROOT/models/jevk5/jevk5-4b-v0.3-Q4_K_M.gguf" \
  --host 127.0.0.1 --port 8081 -c 8192 -ngl 0 -t 4 -np 1
```

Install the upstream GGUF client in the virtual environment, using the code revision recorded with the release:

```bash
python -m pip install --no-deps \
  'git+https://github.com/allebee/jevk5.git@f26426d16f59e8bbe1470e5b162cc89329e29b29'
python - <<'PY'
from jevk5 import JevK5GGUF
model = JevK5GGUF(url='http://127.0.0.1:8081',
                  temperature=1.22, knockout_temperature=0.93)
print(model.decide(
    'A payment was charged twice. The refund handler checks transaction IDs.',
    {'type': 'noul', 'instructions': 'Is the refund handler relevant to investigating duplicate charges?'}))
PY
```

Use this artifact's calibration values from `model-manifest.json`; generic client defaults may refer to another release. A successful response proves the scoring path runs, not its accuracy on code. The readout uses option-token probabilities; missing option coverage must be checked before trusting scores. The current GGUF client disables prompt caching, so shared-prefix acceleration is a future validated change, not an assumed benefit. [GGUF client source](https://github.com/allebee/jevk5/blob/main/jevk5/gguf.py)

For MiniCPM, download a chosen Q4 GGUF from the [official MiniCPM5-2B GGUF repository](https://huggingface.co/openbmb/MiniCPM5-2B-GGUF/tree/main). Record its exact filename, repository revision and SHA-256. Set `MINICPM_GGUF` to that local file. This is a separate download; it is not included in our JevK5 release.

```bash
"$LLAMA_SERVER" -m "$MINICPM_GGUF" -a MiniCPM5-2B \
  --host 127.0.0.1 --port 8083 -c 8192 -ngl 0 -t 4 -np 1 --jinja
```

Use the model card's initial sampling settings (`temperature: 1.0`, `top_p: 0.95`, `min_p: 0.0`) when establishing a reference. Tool calling needs its proper template/parser: the model uses an XML-style tool format. A working chat response does not establish working tool calls. Before integrating an agent, test a harmless file-read request, confirm the parsed name/arguments, return the result with the right call identity, and verify the model continues. Reject malformed calls rather than executing raw generated text. [MiniCPM5-2B serving and tool documentation](https://huggingface.co/openbmb/MiniCPM5-2B)

Metal/CUDA trials can change the offload setting after confirming the build/backend. Log the actual device placement and memory; do not infer it from `-ngl` alone.

## 6. Connect CodeGraph

Use graph-only mode initially to avoid adding an embedding model. Structural navigation remains useful; semantic embedding search is unavailable in that mode. Upstream supplies prebuilt binaries; a source build is another option:

```bash
cd "$STACK_ROOT/vendor"
git clone https://github.com/codegraph-ai/CodeGraph.git codegraph
cd codegraph
CARGO_BUILD_JOBS=2 cargo build --release -p codegraph-server
git rev-parse HEAD
```

A host's MCP configuration should launch the resulting binary with the target workspace. Substitute absolute paths; JSON below is a configuration shape, not a universal host-specific filename:

```json
{
  "mcpServers": {
    "codegraph": {
      "command": "/absolute/stack/vendor/codegraph/target/release/codegraph-server",
      "args": ["--workspace", "/absolute/target/repository", "--graph-only", "--profile", "all"]
    }
  }
}
```

Discover actual tool schemas through MCP. Initially expose only symbol search, contextual source, callers/callees and related tests to the model; the host must implement that filtering if it starts the `all` profile. Alternatively choose an upstream narrower profile and accept its restricted tool set. Avoid multiple processes competing over the same index merely to obtain different profiles. [CodeGraph installation, profiles and tools](https://github.com/codegraph-ai/CodeGraph)

Acceptance: find a known symbol, inspect a known caller, edit a fixture and confirm retrieval updates. Preserve file paths, line ranges, source hashes and revision with every result. Missing dynamic calls or unsupported syntax require ordinary text search and source inspection.

## 7. Integrate JevGrep with local JevK5 — implementation required

Install the correct upstream project, not an unrelated unscoped npm package:

```bash
cd "$STACK_ROOT/vendor"
git clone https://github.com/nassim-arifette/jevgrep.git
cd jevgrep
npm ci
npm run build
git rev-parse HEAD
```

The package is `@nassim-arifette/jevgrep`. Its provider configuration has an endpoint setting, but changing that alone is not proof of local compatibility. Preserve `remote_evaluation_enabled: false` while setting up and auditing the route. Read the pinned provider's enablement semantics before enabling evaluation; never assume this flag independently distinguishes localhost from cloud. [JevGrep](https://github.com/nassim-arifette/jevgrep), [configuration example](https://github.com/nassim-arifette/jevgrep/blob/main/docs/examples/jevgrep.config.json)

Implement and verify these two seams:

| Seam | Required behavior | Acceptance check |
|---|---|---|
| Typed-decision bridge on port 8082 | Translate the pinned provider contract to `JevK5GGUF.decide`; preserve question IDs, answer types and errors | Replay a fixture request; compare its scores with direct Python calls |
| Graph-to-fragment adapter | Pass bounded CodeGraph candidate excerpts to scoring with original file/span/hash metadata | A small known candidate set is scored without silently rescanning the entire repository |

Inspect [JevK5's typed server](https://github.com/allebee/jevk5/blob/main/jevk5/server.py) and the pinned JevGrep provider together. The existing default JevK5 server is not automatically the GGUF bridge. If a provider requires authentication, implement an explicit loopback-only development credential contract. Keep hosted providers disabled for this local profile.

Bridge engineering requirements:

1. Tokenize the complete request with the model's tokenizer, including instructions and choices. Enforce the configured 8,192-token context with output headroom; do not silently truncate code.
2. Begin with one question per request and bounded excerpts. Larger JevGrep batch defaults are not evidence that this model can accept them efficiently.
3. Return stable IDs and the pinned model identity. Record scoring time, evaluated tokens, cache hits, missing option coverage and failures.
4. Cache only with complete identity: query, excerpt hash, model digest, scoring configuration and relevant instruction/schema version.
5. Apply backpressure and timeouts. A scoring failure should retain the structural candidate or trigger broader retrieval, not erase evidence.
6. Treat probabilities as rankings until code-relevance calibration is measured. Do not discard every low-scoring candidate if that breaks recall.

Once the adapter is validated, the host can launch the JevGrep MCP entry point:

```bash
node /absolute/stack/vendor/jevgrep/dist/cli.js mcp \
  --config /absolute/stack/local-jevgrep.config.json
```

That configuration must target the verified localhost bridge. This guide deliberately does not supply a fictional ready-to-run adapter configuration.

## 8. Agent host and the complete issue loop

Choose a host with configurable local chat serving, MCP clients, bounded file reads/edits, test execution and trace logging. If writing a small host, implement these capabilities explicitly. A chat server alone cannot operate this workflow.

Proposed initial limits: one issue, eight diagnosis/edit iterations, at most twelve candidate fragments before refinement, bounded test output and a per-task deadline. These are tunable starting choices, not measured optima. Reserve context for tool definitions, patch output and the next test result.

1. **Capture the starting state.** Record repository URL, commit, existing dirty diff and issue text. Work in an isolated branch/worktree. Define the relevant tests and success criteria before editing.
2. **Locate evidence.** Ask CodeGraph for symbols and relationships suggested by the issue. For a duplicate-charge bug, inspect the charge path, idempotency key handling and tests together.
3. **Resolve ambiguity.** If several handlers could own the behavior, score bounded excerpts through JevGrep/JevK5. Keep structurally required callers and widen retrieval when the evidence is incomplete.
4. **Diagnose with MiniCPM.** Supply issue, source spans, relationships and observed failure. Request a concrete hypothesis and the smallest justified patch. The host checks edits against the source hash used for diagnosis.
5. **Run tests.** Reproduce the bug, then run the targeted test after the edit and a relevant regression set. Limit workers initially. Store full logs on disk; return the failing assertion and necessary stack/context to the model.
6. **Refresh state.** Invalidate edited fragment scores and update the graph. Re-read changed source before another patch; do not reuse stale line ranges blindly.
7. **Continue with evidence.** Failed tests become new observations. On success, record exactly which tests passed. On deadline or unresolved failure, retain the partial patch and mark the task unresolved.
8. **Return the patch and record.** Report changes, validation, limitations and artifacts. Never equate the model saying “fixed” with a passing evaluation.

The host should expose only workspace-scoped actions and parse tool arguments using schemas. Repository text, comments and test logs are evidence, not instructions that can redefine the host's behavior.

## 9. Store outcomes so another computer can resume

Use a per-task directory plus JSON records, or SQLite with artifact paths. The format below is a proposed contract:

```json
{
  "schema_version": 1,
  "task_id": "duplicate-charge-001",
  "repository": "https://example.org/team/project.git",
  "base_commit": "FULL_COMMIT_SHA",
  "initial_dirty_patch_sha256": null,
  "issue": "Duplicate retries can charge twice",
  "models": {"jevk5_sha256": "94ca0d7745c47f79091b0892ca657c81d9dc9e4ed0238ba0a7ea261d8938c882", "minicpm_sha256": "RECORD_ACTUAL_DIGEST"},
  "runtime_revisions": {"llama_cpp": "RECORD_SHA", "codegraph": "RECORD_SHA", "jevgrep": "RECORD_SHA", "host": "RECORD_SHA"},
  "evidence": [{"path": "src/payment.py", "lines": [40, 85], "sha256": "RECORD_SOURCE_HASH", "reason": "retry path"}],
  "patch": "patch.diff",
  "tests": [{"argv": ["python", "-m", "pytest", "tests/test_payment.py"], "cwd": ".", "exit_code": 0, "log": "tests.log"}],
  "outcome": "targeted_tests_passed",
  "metrics": {"wall_seconds": null, "peak_memory_bytes": null, "retrieval_seconds": null},
  "remaining_questions": ["Full regression suite not run"]
}
```

The values are illustrative, not a test result. Add hardware, OS, tool versions, sampling settings, source/test dependency versions, cold/warm state and full artifact hashes to real records. Keep sensitive source/logs in the project's intended storage rather than automatically uploading them to this public repository.

A matching repository URL alone is not a cache key. On another machine, restore the exact commit and patch, verify source hashes, rebuild platform-specific indexes, remap workspace paths and then reuse matching records. Treat old diagnoses as hypotheses when source changes. A stored failure is valuable if its triggering inputs and evidence remain reproducible.

## 10. Evaluate benefit before expanding the system

Compare the same tasks, model weights, tool budget and execution environment:

| Variant | What it establishes |
|---|---|
| MiniCPM + ordinary search/edit/tests | Reference coding agent |
| Reference + CodeGraph | Value of structural retrieval |
| Reference + CodeGraph + local JevGrep/JevK5 | Whether learned relevance pays for its additional cost |
| Full variant + evidence reuse | Whether prior outcomes help on new turns without stale-state errors |

Measure resolved tasks and **time to a correct patch**, with index creation, model loading, retrieval, scoring, prompt processing, generation and tests separated. Include failures/timeouts. Report cold and warm runs independently; repeat tasks or use paired runs to expose timing variability. Measure generated tokens/s separately—it is not interchangeable with decision latency or task completion speed.

For SWE-style evaluation, use the official task isolation and test harness for the chosen benchmark, with a recorded split and fixed resource limits. Do not expose reference patches or held-out outcomes through the evidence cache. Upstream model scores do not transfer to this quantization, runtime and retrieval stack.

A useful relevance gate must preserve the source necessary to solve the issue. Evaluate relevant-fragment recall alongside ranking precision: a fast filter that discards the actual bug loses overall.

## 11. How prior SSD work, mini-AGI and Gigatoken fit

This coding stack retrieves **source evidence**. Our earlier SSD executor retrieves **model parameters**. Both can reduce wasted movement, but they operate at different levels. CodeGraph does not identify neural experts simply by finding a source symbol.

Retain earlier engineering candidates for a separately measured runtime integration: resident address directories, selected-row reads, executable expert tiles, deduplicated reads, bounded caches, weight reuse across ready requests and joint state/weight admission. The present stock llama.cpp commands do not automatically inherit those custom executor changes. The earlier roughly 40-to-18 GB traffic comparison is not a measured result for this new stack.

[mini-AGI](https://github.com/volotat/mini-AGI) is a research reference for expert placement, caching and scheduling experiments, not a replacement coding-agent host or a converter for arbitrary model checkpoints. Borrow mechanisms only after checking mathematical/model compatibility and measuring full-task costs. Restricting routing choices is a model-behavior change, not merely an I/O optimization.

[Gigatoken](https://github.com/marcelroed/gigatoken) remains part of a future training-data preparation path: tokenize approved, versioned examples once, verify tokenizer/template/loss-mask compatibility and create bounded shards. It is not needed to run this published model and does not accelerate inference by itself. Training should follow evidence of a specific quality gap and a held-out evaluation set.

## 12. Handoff checklist and next implementation order

1. Verify the published JevK5 digest and direct local scoring example.
2. Verify MiniCPM text generation and one complete tool-call round trip.
3. Connect CodeGraph MCP and prove index refresh after an edit.
4. Implement/test the GGUF decision bridge and CodeGraph-to-JevGrep candidate adapter.
5. Connect the host's bounded issue/edit/test loop and evidence writer.
6. Run one reproducible issue from failure to tested patch.
7. Measure the three principal variants before adding more concurrency or training.
8. Enforce/measure the 8 GiB service target; profile the largest remaining costs.

At handoff, provide the component commit manifest, model hashes, local endpoint settings, reproduction task, patch, logs and memory/timing results. Do not copy a running process's state and assume it is portable between CUDA, Metal and CPU.

On the next Mac or PC, pull this repository for documentation updates and fetch model release assets separately:

```bash
git pull --ff-only
```

**Completion criterion:** a repeatable, useful tested patch with recorded end-to-end cost and resource usage. Smaller downloads, faster isolated scores and successful model loading are intermediate checks.
