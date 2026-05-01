# Automotive Agent — SWE-bench Lite

<p align="center">
  <img src="assets/banner.png" alt="Automotive Agent" width="600"/>
</p>

<p align="center">
  <a href="https://github.com/suhail-akhtar/automotive-swe-bench"><strong>Results & Code</strong></a> &nbsp;|&nbsp;
  <a href="https://suhail-akhtar.github.io/automotive-swe-bench"><strong>Website</strong></a>
</p>

---

## Overview

**Automotive** is a fully autonomous, multi-agent AI orchestration system built for end-to-end software engineering tasks. This repository documents our submission to the [SWE-bench Lite](https://www.swebench.com) leaderboard.

Automotive routes each SWE-bench task to a specialized **Developer Agent** — a focused sub-agent equipped with file-system tools, code search, shell execution, and a structured test-execution feedback loop. The agent is powered by **claude-sonnet-4.6** via the GitHub Copilot SDK.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  Automotive Orchestrator                  │
│              (claude-sonnet-4.6, Copilot SDK)            │
└──────────────────────────┬──────────────────────────────┘
                           │  delegates per task
              ┌────────────▼────────────┐
              │     Developer Agent      │
              │  (claude-sonnet-4.6)     │
              │                          │
              │  Tools:                  │
              │  • grep / glob           │
              │  • view (read files)     │
              │  • edit / create files   │
              │  • run_command (shell)   │
              │  • bash (pytest, git)    │
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │   Test-Execution Loop    │
              │                          │
              │  1. Read issue           │
              │  2. Explore codebase     │
              │  3. Write patch          │
              │  4. Run pytest           │
              │  5. Read failure output  │
              │  6. Fix → repeat         │
              └─────────────────────────┘
```

### How It Works

1. **Issue Understanding** — The agent reads the problem statement and identifies the affected module using `grep` and `glob` to search the codebase.
2. **Root Cause Analysis** — The agent reads the relevant source files, traces the execution path, and identifies the exact line(s) causing the bug.
3. **Patch Generation** — A unified diff patch is written targeting the precise location of the fix.
4. **Test Execution** — The agent runs pytest on the repository and reads failure output to verify the fix is correct.
5. **Iterative Refinement** — If tests fail, the agent reads the traceback, adjusts the patch, and re-runs until all tests pass or the iteration limit is reached.
6. **Patch Formatting** — The final patch is validated for correct unified diff format before submission (hunk counts, trailing newline, LF encoding).

### Key Design Decisions

- **Single agent per task** — Each task runs in an isolated developer agent context. No cross-task contamination.
- **Test-execution discipline** — The agent is explicitly instructed: *"never claim a fix is complete without running the tests and seeing them pass."*
- **No test ID leakage** — The agent receives only the problem statement and codebase. `FAIL_TO_PASS` and `PASS_TO_PASS` test identifiers are NOT provided to the agent.
- **No web browsing** — The agent operates entirely on the local cloned repository. No internet access is used during patch generation.
- **Pass@1** — One prediction per task instance. No re-attempts or ensemble selection.

---

## Results

### SWE-bench Lite — 20-Task Pilot

> Initial validation run on a representative sample of 20 tasks (6 astropy, 14 django).
> Verified using the official [SWE-bench Docker harness](https://github.com/swe-bench/SWE-bench).

| Repository | Tasks | Resolved | Score |
|------------|-------|----------|-------|
| astropy | 6 | 6 | 100% |
| django | 14 | 14 | 100% |
| **Total** | **20** | **20** | **100%** |

**Official harness output:**
```
Evaluation: 100%|██████████| 20/20 [18:03<00:00]  ✓=20, ✖=0, error=0
Total instances: 20  |  Resolved: 20  |  Unresolved: 0  |  Errors: 0
```

### Score Progression During Development

| Run | Score | Fix Applied |
|-----|-------|-------------|
| Initial run | 16/20 = 80% | Baseline |
| After patch format fix | 17/20 = 85% | Missing trailing `\n` in patch |
| After root cause fixes | 19/20 = 95% | Wrong file target; missing Django built-in |
| **Final** | **20/20 = 100%** | Hunk header count correction |

---

## What We Learned — Key Failure Patterns

During our pilot run, 4 tasks initially failed. Root cause analysis revealed important lessons:

| Failure | Root Cause | Category |
|---------|-----------|----------|
| astropy-14182 | Patch string missing trailing `\n` | Patch format |
| django-11019 | Reimplemented `stable_topological_sort` instead of using Django's built-in | Unknown library function |
| django-11620 | Fixed `resolvers.py` — wrong file; bug was in `views/debug.py` | Wrong diagnosis |
| astropy-7746 | Hunk 1 declared 6 old lines, body had 5 — `except ValueError:` missing | Patch format |

These patterns informed the test-execution loop: running pytest before declaring done catches 3 of these 4 failures automatically.

---

## Patch Format Lessons (Reproducible Knowledge)

1. **Trailing newline is mandatory** — the `patch` binary fails with *"unexpectedly ends in middle of line"* if the patch string doesn't end with `\n`
2. **Hunk counts must be exact** — `@@ -start,old_count +start,new_count @@`: count all context lines + removed lines for `old_count`, all context + added for `new_count`
3. **Target the right file** — test class names hint at the correct module (`DebugViewTests` → `views/debug.py`, not `resolvers.py`)
4. **Check existing library functions first** — Django, astropy, sympy have extensive utilities; reimplementing diverges from expected behaviour verified by the test suite

---

## Technical Stack

| Component | Technology |
|-----------|-----------|
| Orchestration | Automotive multi-agent framework |
| Agent model | `claude-sonnet-4.6` (GitHub Copilot SDK) |
| Agent tools | grep, glob, view, edit, run_command, bash |
| Evaluation harness | SWE-bench Docker (official) |
| Infrastructure | WSL2 Ubuntu-24.04, Docker Desktop |
| Parallelism | 4 Docker workers (--max_workers 4) |

---

## Submission Info

- **System name:** Automotive Developer Agent
- **Model:** claude-sonnet-4.6
- **Open source model:** No (Anthropic Claude)
- **Open source system:** No (private)
- **Pass@1:** Yes — one attempt per task
- **Test knowledge used:** No — agent receives only the problem statement
- **Web browsing:** No — agent operates on local repository only

---

## Author

**Suhail Akhtar**
- GitHub: [@suhail-akhtar](https://github.com/suhail-akhtar)

---

## Citation

If you reference this work, please cite:

```bibtex
@misc{akhtar2026automotive,
  title  = {Automotive Agent: A Multi-Agent System for Automated Software Engineering},
  author = {Suhail Akhtar},
  year   = {2026},
  url    = {https://github.com/suhail-akhtar/automotive-swe-bench}
}
```
