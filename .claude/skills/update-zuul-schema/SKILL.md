---
name: update-zuul-schema
description: 'Use when asked to update zuulcilint/zuul-schema.json for a new Zuul release, or to add support for a specific Zuul config attribute/feature mentioned in Zuul release notes.'
---

# Update zuulcilint's zuul-schema.json from Zuul release notes

`zuulcilint/zuul-schema.json` is a JSON Schema (draft 2019-09) that validates
Zuul's **in-repo YAML configuration** (`.zuul.yaml` / `zuul.d/*.yaml`): the
`job`, `pipeline`, `project`, `project-template`, `nodeset`, `queue`,
`secret`, `semaphore`, and `pragma` top-level items. It does **not** cover
`zuul.conf` (connections, schedulers, executors, web) or anything REST/API/CLI
related — that's out of scope for this schema no matter how interesting the
release note is.

## Workflow

1. **Find the release notes for the target version.** This repo does **not**
   vendor Zuul's source — `zuul/` is a plain clone of
   `https://opendev.org/zuul/zuul.git` that the user makes locally themselves,
   and it may not exist yet. Check with `test -d zuul/.git`; if it's missing,
   ask the user to clone it (don't clone third-party git repos yourself
   without asking) with something like
   `git clone https://opendev.org/zuul/zuul.git`. If it exists, check what
   it's checked out at with `git -C zuul describe --tags` or
   `git -C zuul log -1`. If it doesn't match the version you're updating to,
   say so and ask the user to check out the right tag — don't guess from
   memory, release note wording changes between drafts and merged versions.
   Individual reno fragments live in `zuul/releasenotes/notes/*.yaml`; grep
   them for the feature keywords (e.g. `grep -rl "replication_delay"
   zuul/releasenotes/notes/`).

2. **Classify each feature before touching the schema:**
   - **In scope**: new/changed attributes on `job`, `pipeline` (trigger,
     require, reject, reporters), `project`, `nodeset`, `queue`, `secret`,
     `semaphore`, `pragma`.
   - **Out of scope**: `zuul.conf` driver/connection settings (e.g. a new
     Gerrit connection option like `replication_delay`), scheduler/executor
     internals, web UI, REST API, CLI flags. Confirm scope by checking where
     the attribute is documented/implemented in `zuul/doc/source/` and
     `zuul/zuul/driver/*/*.py` — connection-level settings are read via
     `self.connection_config.get(...)` in `<driver>connection.py`, not via
     `getSchema()`/parser context in `<driver>trigger.py` /
     `<driver>reporter.py` / `zuul/configloader.py`.
   - Say explicitly which features you're skipping and why, rather than
     silently dropping them.

3. **Watch out for docs drifting ahead of the target release.** The `zuul/`
   checkout's docs describe upstream `master`, which can differ from an
   older tag: enum values or job types may exist in docs but not yet in the
   tag you're targeting. Cross-check against the actual tagged source (e.g.
   `grep` the relevant `.py`/`.rst` at that tag) rather than trusting the
   release note wording alone, and prefer failing narrow (adding less) over
   inventing values that don't exist yet at that version.

4. **Locate the right spot in `zuul-schema.json`.** High-level shape of the
   file:

   ```
   zuul-schema.json
   ├─ $schema / title / description / version   (schema's own metadata)
   ├─ type: array, items.oneOf                   (top-level YAML doc items)
   │   → nodeset | job | pragma | project | pipeline
   │     | project-template | queue | secret | semaphore
   │   (each just $refs into definitions.<name>)
   └─ definitions
       ├─ job              → anonymous-job (name required)
       │   └─ anonymous-job          all job.* attributes (the big one)
       │       ├─ nodeset            → anonymous-nodeset
       │       ├─ run / pre-run / post-run / cleanup-run
       │       │                     → run-type / post-run-type
       │       │                        → run-type-object / post-run-type-object
       │       ├─ secrets, semaphores, roles, dependencies, include-vars, ...
       │       └─ branches, files, irrelevant-files → regular-expression
       ├─ nodeset          → anonymous-nodeset
       │   └─ anonymous-nodeset      nodes → nodeset-nodes-object, groups, alternatives
       ├─ pipeline                   pipeline.* attributes
       │   ├─ trigger.patternProperties.oneOf
       │   │     → github-trigger | gerrit-trigger | zuul-trigger | timer-trigger
       │   ├─ require  → any-driver-require
       │   │     → github-require-entry | gerrit-require-entry
       │   ├─ reject   → any-driver-reject
       │   │     → github-reject-entry | gerrit-reject-entry
       │   └─ success/failure/merge-conflict/config-error/enqueue/start/
       │      no-jobs/disabled/dequeue → any-driver-reporter
       │            → github-reporter-entry | gerrit-reporter-entry
       │              | mqtt-reporter-entry
       ├─ project          project.* + patternProperties.<pipeline>.jobs/debug/fail-fast
       ├─ project-template → allOf [ ..., #/definitions/project ]
       ├─ queue, secret, semaphore, pragma   (small, self-contained)
       ├─ gerrit-review-category            (shared by gerrit trigger/require/reporter)
       └─ regular-expression                (shared string-or-{regex,negate} type)
   ```

   Driver-specific pieces are modeled per-driver under `definitions`, e.g.
   `gerrit-trigger`, `github-reporter-entry`, `github-require-entry`,
   `any-driver-reporter`, etc. Only gerrit, github, zuul (trigger), timer
   (trigger), and mqtt (reporter) are currently modeled — **gitlab has no
   schema support at all**. If a release note is about a driver/attribute
   that isn't modeled yet (e.g. a new GitLab trigger attribute), that's a
   bigger addition than a one-line fix (new `<driver>-trigger`/
   `-reporter-entry`/`-require-entry`/`-reject-entry` definitions, wiring
   into the `oneOf`/`patternProperties` lists) — flag it as a separate,
   larger follow-up rather than bolting one attribute onto a driver that
   isn't there.

5. **Make the edit.** Keep the existing style: `title`/`description` per
   property, `enum` lists for closed value sets, `default` where the docs
   state one, and reuse `$ref`s (`regular-expression`, `run-type`, etc.)
   instead of duplicating structure.

6. **Bump the schema's own metadata**: the top-level `title` (contains the
   Zuul version, e.g. `"... Zuul CI 13.1.0 configuration files"`) and
   `version` (the schema file's own semver — bump minor for new attributes).

7. **Validate JSON**: `jq empty zuulcilint/zuul-schema.json`.

8. **Add test fixtures** under `tests/zuul_data/`, matching the existing
   convention of one small, isolated named example per feature:
   - Job-level attributes → `tests/zuul_data/jobs.yaml` (e.g.
     `test-job-type-initializer`).
   - Pipeline/trigger/require/reject/reporter attributes →
     `tests/zuul_data/pipelines.yaml`.
   - Nodeset/queue/secret/semaphore/project attributes → their respective
     `tests/zuul_data/*.yaml` files.
   Before adding, read `tests/test_zuulcilint.py::test_warnings` and
   `test_playbook_errors` — they assert **exact hardcoded counts** (currently
   5 inexistent nodesets, 1 duplicate job, 1 bad-extension file, 9 playbook
   path errors). New fixtures must not perturb these unless you also update
   the assertions:
   - Omit `nodeset:` on new job examples unless you reuse a name that
     genuinely exists elsewhere in `tests/zuul_data/nodeset.yaml` (jobs
     without a `nodeset` key are skipped by the inexistent-nodeset check).
   - Don't add `run`/`pre-run`/`post-run` playbook paths unless you intend
     to add to the playbook-path-error count.
   - Don't reuse an existing job `name` (bumps the duplicate-job count).

9. **Run the tests.** Check for an existing virtualenv first — don't assume
   one is or isn't there:
   - Look for `.venv/`, `venv/`, an active `$VIRTUAL_ENV`, or a poetry env
     (`poetry env info` if `pyproject.toml` uses poetry) before creating
     anything.
   - If one exists, use it (activate it, or run e.g.
     `.venv/bin/python -m pytest tests/ -q` directly) rather than creating a
     second one.
   - If none exists and it's not obvious which tool the user prefers
     (venv vs poetry vs something else), ask rather than guessing — creating
     an environment is cheap to redo but worth confirming once.
   - If you do create one: `python3 -m venv .venv && source .venv/bin/activate
     && pip install -e . pytest pytest-cov`. `.venv/` is already gitignored.
   - Run `python -m pytest tests/ -q` and confirm all tests still pass (33 as
     of this writing) before reporting done.

10. **Summarize** what changed in the schema, what test fixtures were added
    and why, and call out anything explicitly skipped (out-of-scope
    `zuul.conf` settings, unmodeled drivers) so the user can decide whether
    to follow up.
