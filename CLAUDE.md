# CLAUDE.md

Context for Claude Code working in this repository. Read this fully before writing or running anything.

## What this project is

**One sentence:** An LLM agent that, given an arbitrary source tarball, infers and executes a reproducible build recipe — installing dependencies, selecting the toolchain, and running build phases — inside an isolated container.

This is **cold-start build-recipe inference**, NOT build repair. The input is a raw tarball with no working build, no error report, and no guaranteed build-system declaration. The output is a working build plus the recipe that produced it (apt deps, toolchain version, per-phase commands). The recovered recipe *is* the missing dependency specification — the agent is effectively a build oracle.

Undergraduate research project in Paul Gazzillo's lab at UCF. Gazzillo is the advisor; Brent Pappas is a PhD student in the same lab. David Gusmão is the undergraduate researcher — he previously worked under Brent on the Foreman / hardening-build-systems project, building the labeled Apache/GNU/GitHub dataset that is now the evaluation set for this project. This is a direct spin-off of that work.

Anchor every design and code decision back to the one-sentence statement above. If something doesn't tie back to it, flag it.

## Why this matters

Modern projects have sprawling dependency graphs (ffmpeg, Python, etc.). Package managers differ wildly: apt-get just installs, npm has lockfiles, C projects rely on README files that are usually outdated or missing. Human documentation is unreliable. **The dependency list itself is the missing artifact** — an iterative LLM oracle is a plausible way to recover it.

Use cases: enterprise onboarding for new engineers, reproducibility standards at companies, reproducing legacy/abandoned builds, supply-chain audit prep, and (stretch) feeding Foreman with phase specs at scale.

The Foreman / XZ Utils / supply-chain angle is a secondary/stretch pitch — not the core contribution. Do not lead with it.

## Locked design decisions — do not relitigate

- **Tool Bridging (from GradleFixer):** the agent acts through domain-specific tool wrappers (apt, Maven, Gradle, Make, autotools, container ops), NOT raw shell. Build API-like wrappers, each returning raw stdout/stderr/exit code. The constrained action space is the point.
- **Event-sourced execution (from ESAA):** every agent action is a validated, logged intent. Append-only event log for auditability and deterministic replay. Keep LLM reasoning separate from effects.
- **Container isolation is a hard safety property:** the agent NEVER touches the host filesystem. All effects confined to the container. Not a soft preference.
- **Agentless-style fixed pipeline is the primary evaluation baseline:** localize → repair → validate, no autonomous loop. The eventual iterative agent must beat this baseline meaningfully or the contribution is weak.
- **Foreman integration is stretch goal / future work, NOT v1.** Do not build toward it now. If pursued later: record per-phase file accesses during a successful build and emit a phase-permission spec Foreman can consume. Two-pass mode: agent drafts spec from one traced run → human reviews → future builds run under Foreman with that spec. Not real-time interruption. Worth one sentence in the paper; confirm with Gazzillo whether to nail standalone build-recipe inference first and treat Foreman as a second paper.

## Positioning vs. prior work

- **vs. GradleFixer (Son et al., EACL 2026):** GradleFixer repairs existing build failures from commit histories on Android/Gradle only. This is cold-start inference from a tarball across multiple ecosystems. Repair ≠ inference.
- **vs. Lyu et al. "Automatic Dockerfile Generation" (ICSE 2026):** Closest known competitor — top priority for differentiation. Candidate axes: (a) output is richer than a Dockerfile (apt deps + JDK choice + per-phase commands + optional phase-permission specs), (b) multi-corpus scope, (c) optional Foreman integration.
- **vs. Lyu et al. "Automatic Fixing of Missing Dependency Errors" (ASE 2025):** Direct competitor — read and pin down differentiation.
- **vs. EChecker / Breaking-Good:** They detect or explain build errors. This fixes the environment so the build succeeds.
- **vs. AndroidBuildBench:** Single-ecosystem (Gradle/Android). This project is multi-language and multi-corpus.
- **vs. Foreman (Pappas & Gazzillo, ICSE-NIER 2026):** Foreman enforces phase permissions; it does not infer them and currently requires hand-written specs. This agent could generate those specs as a byproduct. The tools compose: agent infers, Foreman enforces.
- **vs. Agentless (Xia et al., 2024):** Agentless argues simple pipelines beat autonomous agents. Implemented as the baseline here. The iterative agent must beat it.

## Repository structure (target)

```
build_agent/
├── baseline/         # Agentless-style three-phase pipeline
│   ├── detect.py     # tarball → {build_system, config_files}
│   ├── infer.py      # config_files → list of candidate recipes (single LLM call)
│   └── execute.py    # recipe → (success/fail, logs)
├── tools/            # Domain-specific tool wrappers (for the full agent later)
├── runner/           # Container management, log capture
├── eval/             # Pass@k harness, metrics, cost tracking
└── data/             # Subset of the existing dataset, in JSON
```

## Repo setup status

All five setup tasks done except LICENSE:

- `.gitignore`, `.env.example`, `requirements.txt`, `README.md` — committed in `961cb6f`.
- **LICENSE — pending.** Confirm with Gazzillo: MIT or Apache-2.0 (Apache-2.0 adds a patent grant). Default to Apache-2.0 if no preference stated.
- **Open question for Friday 2026-05-23 meeting:** keep PR workflow for lab review hygiene, or disable branch protection on this personal repo? Ask Gazzillo.

## Implementation tasks (get the skeleton standing — don't make it good yet)

Deadline: a running executor by Friday's meeting (2026-05-23).

1. **Sandbox executor** (`runner/`). `(image, mount_dir, command, timeout) → (exit_code, stdout, stderr)`. ~50 lines using the Docker Python SDK. Test it by running `apt-get update && apt-get install -y build-essential` in an `ubuntu:24.04` container and confirming you get the exit code back.
2. **Extract + detect** (`baseline/detect.py`). Takes a tarball, extracts to a temp dir, returns `{"build_system": "autotools"|"make"|"cmake"|"maven"|"gradle"|"unknown", "markers": ["configure.ac", "Makefile.am", ...]}`. Pure filename pattern-matching. No LLM.
3. **Ground-truth recipes.** Hand-write JSON recipes for 3 known-good programs from the dataset (one per build-system family if possible) — programs whose exact commands are already known to work.
4. **execute.py** (`baseline/execute.py`). Given a JSON recipe + tarball path: fresh container, install deps, run commands in order, check final exit code. Run it on the 3 known-good cases. If all 3 pass, the executor works.
5. **Recipe inference** (`baseline/infer.py`, after the above). Single LLM call: detection output + marker-file contents → candidate recipe (`apt_deps`, `toolchain`, `commands`) as structured JSON. No iteration yet.

## Recipe / dataset JSON shape

```json
{
  "name": "...",
  "corpus": "Apache|GNU|GitHub",
  "detected_build_system": "Maven|Gradle|Ant|Make|Autotools|CMake|Other",
  "apt_dependencies": ["openjdk-17-jdk", "maven"],
  "tool_versions": {"java": "...", "maven": "...", "gradle": "...", "ant": "..."},
  "commands": {"bootstrap": "...", "configure": "...", "compile": "...", "test": "..."},
  "outcomes": {"bootstrap": "success|failure|n/a", "configure": "...", "compile": "...", "test": "success|failure|unknown|n/a"}
}
```

## The dataset is the evaluation set

A labeled Apache/GNU/GitHub dataset from the prior Foreman work provides ground truth for build system, apt deps, toolchain versions, and phase commands. Treat it as eval, not background. It does NOT contain phase-permission specs. Start the baseline against 3–5 known-good programs, then expand to 20, then 50.

## Operating rules

- Detect the build system before deciding what to run. Always run a baseline first.
- Save logs to a file. Diagnose the **first meaningful error**, not just the final summary line.
- Smallest valid repair only. Allowed levers: apt packages, Java/Maven/Gradle/Ant version, build/env flags. **Never edit source files or POM/build.gradle files** — out of scope and makes a build non-reproducible.
- Do NOT add `-j` to make/cmake (the eval harness handles parallelism).
- Record exact apt package names and exact successful commands, never vague descriptions.
- Modern defaults are not neutral: Ubuntu 24.04 / JDK 21 / Maven 3.8.7 / GCC 13 break older projects. Java version is the biggest lever for Apache. Tests are far more fragile than compilation.

## Assets

- **Research Journal** (Google Doc). Day-to-day notebook. Decisions and bolded primary RQs live there. Most recent entries are most authoritative. When David names a journal tab, only look at that tab — don't pull from others unless specified.
- **Overleaf writeup** (~40 pages). Current scoping doc. Contains ~36 RQs; target is 1 primary + 2 secondary by end of Week 2.
- **Labeled dataset** (Apache/GNU/GitHub) from the Foreman work. Ground truth for build system, apt deps, toolchain versions, phase commands. Does NOT contain phase-permission specs.
- **Brent Pappas's research page:** https://pappasbrent.com/research/hardening-build-systems
- **Papers reviewed (Weeks 1–2):**
  - *Keystone:* GradleFixer (Son et al., EACL 2026), Foreman (Pappas & Gazzillo, ICSE-NIER 2026), Agentless (Xia et al., 2024).
  - *Direct competitors:* Lyu et al. "Automatic Dockerfile Generation" (ICSE 2026), Lyu et al. "Automatic Fixing of Missing Dependency Errors" (ASE 2025), MDfixer, Rosa et al. (Dockerfile gen 2023).
  - *Design templates:* ESAA (event sourcing for agents), LoCoBench-Agent (benchmark structure).
  - *Failure-mode references:* EChecker, Breaking-Good.
  - *Background:* Package Calculus, Randrianaina thesis.
  - *Adjacent work:* K-Repro, Post-Training Local LLM Agents, AndroidBuildBench.

## Commit hygiene (lab requirement, per Gazzillo)

- No `git commit -a`. No massive commits (except possibly the very first skeleton commit). No lazy commit messages.
- Small, well-described, incremental commits. Make code review easy.
- Document dependencies, platform, and usage as you go — replicability means someone reproduces results without asking you anything.

## How to work with Claude on this project

- **Progress reports** follow Ernst's four-part format (https://homes.cs.washington.edu/~mernst/advice/progress-report.html): quoted previous plan → this week's progress → next week's plan → meeting agenda. Every reference to past work must be inlined; no "my original plan" without quoting it.
- **Anchor everything** to the one-sentence project statement. If a paper, task, or design choice doesn't tie back, either explain the link or flag it.
- **Push back on scope creep.** A paper has 2–4 RQs. Flag any new direction against the primary RQs before endorsing it.
- **Anchor every paper to the project:** brief summary, then explicitly — does it inform method, eval, framing, related work, or none? Say "none" when it's "none."
- **Maintain positioning.** When work drifts toward repair, generic LLM-agent work, or security-as-main-pitch, redirect to: cold-start build recipe inference for arbitrary tarballs across multiple ecosystems.
- **Treat the dataset as eval, not background.** Any agent design or evaluation plan should reference what the dataset can and can't test.
- **Honest feedback by default.** "This RQ is weak" beats being praised. Direct, technical, no hedging.
- **For talks/papers:** establish problem → need → approach → specifics. Use concrete examples (ffmpeg, Python dep hell, a specific Apache tarball that failed). Don't read the title slide; summarize it. Include company-onboarding / code-review-standards framing for industry relevance.
