# VIctory project guidance — merge with existing upstream instructions

This is a template. Preserve the repository's existing instructions and asset policy when adopting it. Use the checked-out source and command help as the source of truth for implemented capabilities.

## Goal

Develop a stable Windows D3D12 compatibility layer using small, reproducible changes. Required runtime operations must execute with the intended source semantics or produce an explicit diagnostic failure. Do not report an incomplete render as correct execution.

## Before editing

- Read applicable root and nested instructions and the assigned task specification.
- Report assigned worktree, `git status --short`, current HEAD and expected base SHA. Stop editing and report a workspace mismatch if the necessary committed files are absent.
- Work only in the assigned task branch/worktree. Preserve user changes. Do not reset, clean, force-push or change another task's branch.
- Read current implementations before designing replacements. VIctory already has SSA, direct DXIL emission, offline translation and shader test infrastructure.

## Semantics and failure handling

- No missing instruction lowered to NOP, unsupported draw/dispatch silently dropped, or output fabricated to make a test pass.
- A conservative implementation is acceptable only when it preserves the required semantics; a temporary approximation must remain explicitly unverified and cannot satisfy strict acceptance.
- Loop/watchdog exhaustion is a recorded failure, not successful shader completion. Preserve watchdog protection; diagnose the fault.
- `run --strict` concerns host pass-through; do not present it as a GPU no-skip setting.
- Correct Windows API pass-through is allowed where the API contract matches. Do not disable all pass-through in the name of correctness.
- Do not change expected outputs, FP tolerances, test scope or the definition of PASS without explaining why and obtaining independent review of that change.

## Existing commands to verify against the selected SHA

```text
cargo run --locked -p xtask -- policy
cargo fmt --all -- --check
cargo clippy --workspace --all-targets --locked -- -D warnings
cargo clippy --workspace --all-targets --all-features --locked -- -D warnings
cargo build --release --locked -p victory
cargo build --profile diag --locked -p victory --features diagnostics
cargo test --workspace --locked
```

Use the task's targeted verification first. Run broader gates when required for integration or to resolve a concrete remaining risk. Baseline GPU-sensitive tests belong to the GPU queue. A missing prerequisite or early-return skip is not a successful semantic test.

## Evidence and tests

- Record base/candidate SHA, binary hash, test inputs/hash, seed, toolchain/compiler/validator, OS, driver, adapter identity, flags and test status.
- Distinguish PASS, FAIL, BLOCKED, UNSUPPORTED and NOT_RUN. Report required tests that did not run. A passed negative test must identify its expected diagnostic failure separately from runtime success.
- Exit code 0 from the current offline translator is insufficient: inspect the failure tally. `--cache` is separate from `--validate`/`--list`.
- Start with existing small synthetic tests and one concrete failing reproducer. Independently derive the reference result.
- Two GPUs running code emitted by the same broken translator can agree. Agreement alone is not proof of Xbox equivalence.
- Validate defined integer behavior exactly; choose FP comparisons per source instruction contract. Test legal atomic outcomes/invariants where scheduling is nondeterministic.

## GPU execution

- `gta6-gpu` owns hardware runs, game launches, pipeline warm-up and captures. Run one such workload at a time across tools/clients.
- During bootstrap, keep game-file analysis offline until MS003's explicit runtime failure policy is accepted. Use synthetic applications or official samples for MCP capture checks. This is the selected project sequence, not a claim that upstream cannot launch the title.
- Identify the actual DXGI adapter and driver; never assume enumeration index 0 or nonzero dedicated memory identifies the intended device.
- An unsupported native Wave64 feature on NVIDIA is not permission to skip an executed game shader.
- Correctness and performance runs use explicitly recorded instrumentation modes. Heavy GBV/profiling changes timing; do not treat those timings as release performance.
- Keep cache, captures and save data outside all repositories/worktrees, isolated by candidate and execution context. Do not share mutable save data or pipeline libraries between concurrent tasks.

## Data, tools and integration

- Follow upstream asset policy. Commit only source, documentation and permitted synthetic fixtures. Keep game binaries, dumps, captures, credentials and derived proprietary content outside Git.
- Use official API/ISA documentation and source pinned to a commit. Treat issue reports and agent conclusions as evidence to verify, not authority to change contracts.
- Preserve license/notices and verify conditions before importing code from another project.
- Use native terminal/file tools and the configured MCP servers. Do not install extra integrations unless the current task needs them.
- Use `kanban_*` tools for board operations when available. Persist progress, exact blockers and candidate SHA before handing off.
- Reviewer and GPU QA check the exact candidate. Integration records the resulting SHA and runs the relevant regression checks.
- After CHANGES, address the specific findings within the existing task scope. Do not restart an unrelated implementation.

## Completion report

State the problem, change, evidence, tests actually executed, tests not executed and why, remaining uncertainty, and next required action. Do not give an FPS forecast or declare universal correctness from a limited test set.
