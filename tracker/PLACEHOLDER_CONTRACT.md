# Benchmark Placeholder Contract

Runtime placeholders that may appear in benchmark prompt texts (`prompts.json`).
The dispatching environment / Benchmark Supervisor MUST resolve them before the
prompt reaches a model. An unresolved placeholder must never silently reach a
model as literal text.

## Current 50-prompt source contract

The V2 pack preserves all 50 supplied numbered task texts exactly, including
newlines and trailing whitespace. IDs, titles and difficulties come from the
source filenames; `source_ids: [id]` preserves the original numbered lineage,
not the superseded 43-prompt consolidation. `prompts-data.js` contains only
`id`, `title` and `difficulty`, matching `prompts.json` in the same order.
Historical tracker results are not migrated by this replacement and must not
be assumed comparable across pack revisions merely because IDs match.

The supervisor must resolve these named placeholders before dispatch:

| Placeholder | Meaning |
|---|---|
| `{{BENCHMARK_SESSION_ID}}` | Stable benchmark sweep/session ID; survives pause/resume and backend restart. |
| `{{PROMPT_ID}}` | Stable ID from the selected prompt source. |
| `{{PROJECT_NAME}}` | Canonical filesystem-safe project name derived from metadata. |
| `{{MODEL_ID}}` | Exact selected model slug/ID for the chosen harness. |
| `{{MODEL_DISPLAY_NAME}}` | Human-readable selected model name; delegated agents must use the same model or work stays in the primary session. |
| `{{HARNESS_KEY}}` | Short lowercase harness/provider key. |
| `{{RUN_NUMBER}}` | Supervisor-allocated two-digit run number, e.g. `01`. |
| `{{BENCHMARK_BASE_DIR}}` | User-selected benchmark root. |
| `{{RUN_PROJECT_DIR}}` | Exact atomically reserved project directory; the agent must use it unchanged. |
| `{{IMAGE_GEN_SKILL_PATH}}` | Configured image-generation script path, or `UNAVAILABLE`; no substitute image-generation provider. |
| `{{FISH_AUDIO_SKILL_PATH}}` | Harness-local `fish-audio-universal/SKILL.md` absolute path, or `UNAVAILABLE`; optional synthesized speech. |
| `{{MULTIPLAYER_SKILL_PATH}}` | Harness-local `ox-multiplayer-games/SKILL.md` absolute path, or `UNAVAILABLE` if not installed/resolvable; only prompts 41–46. |

The first eleven placeholders occur in every task. Multiplayer adds the twelfth
only for IDs 41–46. The literal `{{...}}` in the execution rules is explanatory
notation, not a named runtime placeholder: preserve it, do not invent a resolver
for it, and reject unresolved named tokens using `{{[A-Z][A-Z0-9_]*}}` matching.
Required values must not be invented; dispatch fails if they remain unresolved.
The agent-side contract likewise reports injection failure before project work.

### Directory ownership and Windows naming

The source contract's required name format remains verbatim:

`{{PROJECT_NAME}} [model:{{MODEL_ID}}] [{{HARNESS_KEY}}] [run:{{RUN_NUMBER}}]`

The supervisor owns atomic reservation and run-number allocation, including
resume/crash recovery. The agent must not scan for, increment, claim or invent a
sibling run. On Windows, preserve the supervisor's dash naming deviation
(`[model-<filesystem-safe-model-id>] [<harness-key>] [run-<NN>]`): colons and model
slug path separators cannot be literal filename components. Keep the exact model
ID in injected metadata; the final injected `RUN_PROJECT_DIR` is authoritative.
This is a filesystem adaptation, not permission to rewrite the supplied text or
change a reserved directory. The arcade's separately documented legacy `=` tags
remain unchanged; this pack replacement does not change folder scanners.

Runs use unused localhost ports, isolated browser profiles/contexts, run-scoped
artifacts, and cleanup limited to their own resources. Existing valid work from
the exact same run is resumed. Real project verification, screenshot inspection
and a concise `RESULT.md` remain part of each supplied task's execution contract.

### Capability setup and local configuration

Install and verify capabilities before benchmark timing. Machine-specific roots,
API URLs, model defaults and capability paths belong in editable supervisor/local
configuration, not reusable task text or scheduler constants.

- Image generation uses only the injected capability when it materially helps;
  inject `UNAVAILABLE` when unconfigured and preserve task-specific requirements.
- Fish Audio is optional for all 50 prompts. Resolve the harness-appropriate
  `fish-audio-universal/SKILL.md`, or `UNAVAILABLE`; missing speech capability
  does not block unrelated work. Standard locations include
  `~/.agents/skills/`, `~/.codex/skills/`, `~/.cursor/skills/`, or an intentional
  `<project>/.agents/skills/` installation. The skill owns credential lookup;
  agents must not expose or request keys. The source contract names
  `s2.1-pro-free` as the default backend and prohibits rate-limit/probe spam.
- Prompts 41–46 require `ox-multiplayer-games`; validate its identity before
  multiplayer work. Resolve its absolute `SKILL.md` path or `UNAVAILABLE` when
  unavailable, without pretending the required skill is installed. Benchmark
  no-deployment rules override skill deployment/restart instructions. Use it for
  format, manifests, netcode, runtime targets and badge readiness only.

### Current API boundary

The unchanged API resolves only the two legacy skill placeholders documented
below. It does **not** resolve current run identity, image, Fish Audio or
multiplayer placeholders. Current pack responses remain portable templates;
the dispatching supervisor must supply execution-machine values and validate
injection before dispatch. `/api/placeholders` describes legacy server-side
resolvers, not readiness of all twelve current placeholders.

## Legacy `{{E2E_ISOLATION_SKILL_PATH}}`

| Field | Value |
|---|---|
| Capability | isolated-e2e-testing (contested-machine browser/E2E protocol) |
| Type | file-backed skill |
| Required/optional | **Optional** — degrades gracefully |
| Injected value | Absolute path to the installed skill's `SKILL.md` file on the executing machine |
| Fallback | Literal string `UNAVAILABLE` when the skill is not installed for this machine/harness |
| Behavior if UNAVAILABLE | Agent proceeds using the isolation rules already inline in every prompt ("Browser-only + parallel testing" section) |
| Setup timing | Skill installation happens BEFORE benchmark timing; never during it |
| Safety notes | The skill governs browser/process hygiene only; it grants no deployment or production-access permissions. Artifacts produced under its guidance belong in the run's project folder |

### Resolution (server-side)

`api_server.py` resolves placeholders at **serve time** — raw `prompts.json`
stays portable and path-free:

- Resolution order for `{{E2E_ISOLATION_SKILL_PATH}}`: env override
  `E2E_ISOLATION_SKILL_PATH`, then `<home>/.agents/skills/isolated-e2e-testing/SKILL.md`,
  then `<home>/.claude/skills/...`; first existing file wins, else `UNAVAILABLE`.
- Applies to `GET /api/prompts`, `GET /api/prompts/{id}`, `GET /api/prompts/{id}/text`.
- `GET /api/placeholders` shows current resolution status (supervisor/debug aid).
- Note: resolution reflects the **API server's** machine. A local tracker
  resolves real paths for local agents; the kvm2 production instance serves
  `UNAVAILABLE` unless the skill is installed there — correct by contract,
  since the server cannot know the client machine's skills.

Legacy note: this E2E placeholder is absent from the current 50-prompt pack;
the resolver remains for older or externally supplied packs. The older
46-source pack inserted "OPTIONAL CAPABILITY — ISOLATED E2E TESTING" directly
after the "Optional capabilities" section. Its fallback description above
refers to that older inline isolation section.

## Legacy hardcoded capability references

An older pack hardcoded an image-generation script under the author's home
directory. The superseded 43-prompt Astra-native pack omitted capability blocks.
The current 50-prompt source uses the portable image, Fish Audio and multiplayer
placeholders documented above; no hardcoded legacy path is reintroduced.

## Legacy {{SYNTH_SONG_ENGINE_SKILL_PATH}}

| Field | Value |
|---|---|
| Capability | synth-song-engine (offline numpy song synthesis: 84 genre playbooks, 29 voices, deterministic WAV/MP3 render, multi-agent lease) |
| Type | file-backed skill |
| Required/optional | **Optional** — degrades gracefully |
| Injected value | Absolute path to the installed skill's SKILL.md on the executing machine |
| Fallback | Literal string UNAVAILABLE when the skill is not installed for this machine/harness |
| Behavior if UNAVAILABLE | Agent proceeds without it (WebAudio synthesis or other original methods remain fine) |
| Setup timing | Skill installation happens BEFORE benchmark timing; never during it |
| Safety notes | Renders audio offline into the run's project folder; grants no deployment or production-access permissions. FL/DAW contact requires the skill's fl_lease protocol |

### Resolution (server-side)

Same mechanism as {{E2E_ISOLATION_SKILL_PATH}}: env override
SYNTH_SONG_ENGINE_SKILL_PATH, then repo-relative
<repo-root>/skills/synth-song-engine/SKILL.md (ships inside Ephix, so
the kvm2 production instance resolves it), then
<home>/.agents/skills/..., then <home>/.claude/skills/...; first existing
file wins, else UNAVAILABLE. Visible via GET /api/placeholders.

Legacy note: the resolver remains available for older prompt packs; the
current 50-prompt pack does not carry this capability block.
