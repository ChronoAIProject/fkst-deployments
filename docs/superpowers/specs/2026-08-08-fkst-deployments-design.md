# fkst-deployments Design

**Status:** Approved specification; GitHub repository creation gated
**Date:** 2026-08-08

## 1. Purpose and Success Contract

`fkst-deployments` is the version-controlled, machine-independent configuration repository for the `packages`, `substrate`, and `website` deployments. It pins sources, declares deployment policy and composition, references machine values by logical name, and delegates all operational behavior to pinned `fkst-ops`. The dependency direction is only `fkst-deployments -> pinned fkst-ops`. (`fkst-ops/docs/superpowers/specs/2026-08-08-fkst-ops-extraction-design.md:8-14`; `/private/tmp/claude-501/-Users-auric-fkst-packages/26dbcd24-03da-4fc6-b2aa-53a31f541b3a/scratchpad/ctx2.md:11-16`)

The repository succeeds when the same commit can operate all three deployments on multiple machines, every committed declaration passes the real validator from the pin-verified `fkst-ops` checkout, and `board`, `status`, `logs`, `restart`, `sync`, and `doctor` are invocable through that pinned mechanism. (`/private/tmp/claude-501/-Users-auric-fkst-packages/26dbcd24-03da-4fc6-b2aa-53a31f541b3a/scratchpad/ctx2.md:33-40`; `fkst-ops/bin/fkst-ops:31-36`)

Adding a fourth deployment requires one declaration and any new source pin it references, with zero source changes in `fkst-ops`. (`fkst-ops/docs/superpowers/specs/2026-08-08-fkst-ops-extraction-design.md:10-14`)

## 2. Repository Name

The repository name is `fkst-deployments`. The plural states that the repository holds three declarations. (`fkst-packages/.claude/skills/dogfood-github-devloop/dogfood.sh:82-99`)

`fkst-official-deployment` is rejected. `official` is an unverifiable governance claim, the name becomes false when another legitimate deployment repository exists, and `fkst-testhost` is already recorded on disk as evidence that uniqueness cannot be guaranteed; singular `deployment` also contradicts the three declared deployments. (`/private/tmp/claude-501/-Users-auric-fkst-packages/26dbcd24-03da-4fc6-b2aa-53a31f541b3a/scratchpad/ctx2.md:112-127`; `fkst-packages/.claude/skills/dogfood-github-devloop/dogfood.sh:85-96`)

`fkst-fleet` and `fkst-operations` are rejected because this repository neither owns a fleet control plane nor operational mechanism. `fkst-deployment-config` is rejected because `config` repeats the configuration-only ownership rule rather than identifying another scope. (`fkst-ops/docs/superpowers/specs/2026-08-08-fkst-ops-extraction-design.md:38-55`; `/private/tmp/claude-501/-Users-auric-fkst-packages/26dbcd24-03da-4fc6-b2aa-53a31f541b3a/scratchpad/ctx2.md:18-27`)

## 3. One Repository, Three Declarations

One repository owns all three declarations. Three deployment repositories would split one policy and lock domain without separating the mechanism, while embedding declarations in target repositories would assign cross-repository configuration to the wrong owner. The decisive topology is `substrate`: its target is an `fkst-substrate` checkout and its platform is a separate `fkst-packages` checkout, so its declaration is not the property of either checkout alone. (`fkst-packages/.claude/skills/dogfood-github-devloop/dogfood.sh:89-92`; `fkst-ops/docs/superpowers/specs/2026-08-08-fkst-ops-extraction-design.md:38-55`)

The accepted cost is fleet-wide review and blast radius for a repository-level lock or shared policy change. The declarations remain separate files so target-specific diffs stay explicit. Repository-level lock entries may be shared rather than repeated in each declaration. (`fkst-ops/docs/superpowers/specs/2026-08-08-fkst-ops-extraction-design.md:52-55`; `/private/tmp/claude-501/-Users-auric-fkst-packages/26dbcd24-03da-4fc6-b2aa-53a31f541b3a/scratchpad/ctx2.md:114-123`)

## 4. Configuration-Only Boundary

This repository contains declarations, source locks, a machine-profile template, documentation, ignore rules, and the two deployment-edge bootstrap files instantiated byte-for-byte from the pinned `fkst-ops` template. (`fkst-ops/deployments/packages/INSTALL.md:3-16`; `fkst-ops/bootstrap/run.sh:1-8`)

It must never contain lifecycle execution, action implementations, provider implementations, declaration validation, schema code, target discovery, package semantics, board semantics, host supervision, doctor behavior, schedulers, credentials, absolute machine paths, or a second bootstrap framework. Those responsibilities remain in pinned `fkst-ops`, producing packages, or the untracked machine profile according to their ownership. (`fkst-ops/docs/superpowers/specs/2026-08-08-fkst-ops-extraction-design.md:38-73,167-179,323-338`; `fkst-ops/schema/validator.py:83-87`)

The copied bootstrap is the sole exception to ordinary configuration bytes and has only the trust-edge duties in Section 8. It does not become runtime authority. (`fkst-ops/docs/superpowers/specs/2026-08-08-fkst-ops-extraction-design.md:223-240`)

## 5. Exact Repository Layout

The intended tracked and untracked layout is:

```text
fkst-deployments/
|-- .gitignore
|-- README.md
|-- fkst.lock
|-- bootstrap/
|   |-- canonical_tree.py
|   `-- run.sh
|-- deployments/
|   |-- packages.toml
|   |-- substrate.toml
|   `-- website.toml
|-- .fkst/
|   |-- machine-profile.example.toml
|   `-- machine-profile.toml          # untracked, one per machine
`-- docs/superpowers/specs/
    `-- 2026-08-08-fkst-deployments-design.md
```

The bootstrap template ships as `bootstrap/run.sh` and `bootstrap/canonical_tree.py`; the machine-profile template ships from `schema/examples/machine-profile.example.toml`; machine values are not committed. (`/private/tmp/claude-501/-Users-auric-fkst-packages/26dbcd24-03da-4fc6-b2aa-53a31f541b3a/scratchpad/ctx2.md:94-98`; `fkst-ops/schema/examples/machine-profile.example.toml:1-23`)

`fkst.lock` contains one entry for each distinct source ID used below: `fkst-ops`, `fkst-packages`, `fkst-substrate`, and `fkst-website`. Every entry carries `id`, `git`, full lowercase 40-character `resolved.rev`, and canonical `resolved.tree_sha256`; the validator rejects missing pins or malformed values. (`fkst-ops/schema/validator.py:90-113`; `fkst-ops/schema/validator.py:274-287`)

The exact revisions and canonical tree hashes for the first repository cut are **ASSUMED-UNVERIFIED** because no accepted cutover pin set is recorded in the inspected topology. They must be resolved and recorded before validation. (`fkst-packages/.claude/skills/dogfood-github-devloop/dogfood.sh:78-99`; `fkst-ops/deployments/packages/INSTALL.md:18-23`)

## 6. Machine and Deployment Partition

| Classification | Exact contents | Source |
|---|---|---|
| Machine truth, in untracked `.fkst/machine-profile.toml` and referenced by logical name only | Every `DOGFOOD_ROOT`-derived target, platform, and engine checkout path; durable roots including `DUR_*` overrides; runtime root; log root; rate-pool root; engine binary path; bot login; managed bot set; credentials; and a per-device integration branch when it is the machine default. The closed profile has only `roots`, `binaries`, `credentials`, `sets`, and `defaults`; it has no `[commands]` table. | `fkst-packages/.claude/skills/dogfood-github-devloop/dogfood.sh:37-62,78-96`; `fkst-packages/.claude/skills/dogfood-github-devloop/dogfood.config.example.sh:14-31,72-77`; `fkst-ops/schema/validator.py:119-138` |
| Deployment truth, committed | Target identity; source lock refs and pins; package composition; integration policy when the deployment chooses it; provider bindings including the engine binding's required shell-free `configuration.build_command` argv; and the topology fact that `packages` uses the same checkout for target and platform while `substrate` and `website` use separate checkouts. | `fkst-ops/schema/validator.py:231-269`; `fkst-ops/docs/superpowers/specs/2026-08-08-fkst-ops-extraction-design.md:169`; `fkst-packages/.claude/skills/dogfood-github-devloop/dogfood.sh:85-96` |
| Not needed | `GH_ORG`. Each `target_identity` contains the full owner/repository identity and each lock entry contains its Git URL, so no organization variable exists in either layer. | `fkst-ops/schema/validator.py:100-109,265-286`; `fkst-packages/.claude/skills/dogfood-github-devloop/dogfood.sh:61,85-95` |

No declaration contains an absolute path, login, secret, or path-like machine reference. The validator requires logical names and resolves them through the closed machine-profile tables. (`fkst-ops/schema/validator.py:83-87,116-150`)

The build argv must remain committed deployment truth in `provider.configuration.build_command`. It must not be moved to machine truth: `deployment.machine` has no `engine_build_command` field, and the machine-profile schema has no command carrier. (`fkst-ops/schema/validator.py:33-45,119-154,231-269`; `fkst-ops/docs/superpowers/specs/2026-08-08-fkst-ops-extraction-design.md:169`)

The machine-profile template must declare every logical name used by the three declarations:

```toml
schema = "fkst.ops.machine-profile.v1"

[roots]
packages-checkout = "<absolute DOGFOOD_ROOT-derived packages target/platform checkout>"
packages-durable = "<absolute durable root, preserving DUR_PACKAGES override>"
packages-runtime = "<absolute packages runtime root>"
packages-logs = "<absolute packages log root>"
substrate-target-checkout = "<absolute DOGFOOD_ROOT-derived substrate target checkout>"
substrate-platform-checkout = "<absolute DOGFOOD_ROOT-derived separate packages checkout>"
substrate-durable = "<absolute durable root, preserving DUR_SUBSTRATE override>"
substrate-runtime = "<absolute substrate runtime root>"
substrate-logs = "<absolute substrate log root>"
website-target-checkout = "<absolute DOGFOOD_ROOT-derived website target checkout>"
website-platform-checkout = "<absolute DOGFOOD_ROOT-derived separate packages checkout>"
website-durable = "<absolute durable root, preserving DUR_WEBSITE override>"
website-runtime = "<absolute website runtime root>"
website-logs = "<absolute website log root>"
engine-checkout = "<absolute fkst-substrate engine checkout>"
shared-rate-pool = "<absolute rate-pool root>"

[binaries]
engine-binary = "<absolute fkst-framework executable path>"

[credentials]
github-bot-login = "<bot login for this machine>"

[sets]
managed-bot-logins = ["<managed bot login>"]

[defaults]
integration-branch = "<per-device integration branch>"
```

Roots and binaries must be absolute; credentials are strings; sets are string lists; defaults are strings. (`fkst-ops/schema/validator.py:116-135`)

## 7. Provider Binding and Publication Gate

`provider.implementation` has the form `<pinned-source-id>:<safe-relative-entry>`. The validator rejects a missing source pin, an absolute entry, traversal, checkout escape, a missing executable, or a provider kind/contract mismatch. (`fkst-ops/schema/validator.py:20-29,201-215,241-253`)

The hydrated and pin-verified `fkst-ops` checkout is the source root for mechanism-hosted providers. The declarations bind all three published entries:

```toml
implementation = "fkst-ops:providers/engine.py"
implementation = "fkst-ops:providers/board_engine_durable.py"
implementation = "fkst-ops:providers/board_github_control.sh"
```

The explicit publication map binds those paths to `engine`, `board.engine-durable`, and `board.github-control`. File presence and the executable bit publish nothing: a mechanism path absent from the map, or mapped to another kind, fails closed. (`fkst-ops/schema/provider_surface.py:6-13`; `fkst-ops/schema/validator.py:202-218`)

Every provider declaration closes over exactly `id`, `kind`, `implementation`, `contract`, and `configuration`. The engine configuration requires a non-empty string-list `build_command`; both board configurations must be empty. The engine provider executes the resolved argv directly in the engine checkout without a shell. (`fkst-ops/schema/validator.py:231-269`; `fkst-ops/providers/engine.py:54-62,79-90`)

## 8. Bootstrap Trust Edge

There is one bootstrap per deployment repository, not one per declaration. It is instantiated from the `fkst-ops` template and performs exactly these ordered duties:

1. Locate this repository's `fkst.lock`.
2. Hydrate pinned `fkst-ops` and verify full `resolved.rev` plus canonical `tree_sha256`.
3. Hand over the selected declaration path and machine-reference resolution inputs.

The template locates the lock at `bootstrap/run.sh:5-8`, validates both pin forms at `:27-43`, verifies checkout revision and tree at `:48-54,71-81`, and passes the declaration and remaining arguments to pinned `fkst-ops` at `:60-74,81-90`. (`fkst-ops/bootstrap/run.sh:5-90`)

The bootstrap is schema-agnostic and never parses a declaration. It duplicates template bytes but not runtime authority: before hydration there is no pinned mechanism available to authenticate itself, so the deployment edge is the irreducible trust root. (`fkst-ops/docs/superpowers/specs/2026-08-08-fkst-ops-extraction-design.md:223-240`)

The bootstrap may be removed only when a pre-existing launcher is itself version-pinned and authenticated. A globally installed unpinned launcher does not meet that condition. (`fkst-ops/docs/superpowers/specs/2026-08-08-fkst-ops-extraction-design.md:258-259`)

## 9. Sequencing Gate

This specification lands now. The GitHub repository is created only when all four conditions are true:

1. `fkst-ops` is green.
2. The pinned `fkst-ops` revision contains the three-entry published surface and provider-binding configuration contract in Section 7.
3. All three declarations validate through the real validator in the hydrated, pin-verified `fkst-ops` checkout.
4. The `packages` first cutover has a named owner.

This is a gate, not a preference. The engine entry point and its configuration carrier are settled in the mechanism; condition 3 remains unmet until accepted pins, materialized declarations, and a real cutover-host machine profile are present and all three validator invocations succeed. (`fkst-ops/schema/provider_surface.py:6-13`; `fkst-ops/schema/validator.py:93-138,222-269,364-389`; `fkst-ops/deployments/packages/INSTALL.md:18-27`)

`packages` remains the named first adopter. (`fkst-ops/docs/superpowers/specs/2026-08-08-fkst-ops-extraction-design.md:17-32`)

## 10. Declarations

The following are the complete intended files. They use only fields admitted by the closed declaration schema. (`fkst-ops/schema/validator.py:219-253,255-343`)

All three bind the same published `fkst-ops` provider set. Their engine binding carries the existing checkout-local build as the shell-free argv `["cargo", "build", "-p", "fkst-framework"]`; each board binding carries the required empty configuration. (`fkst-ops/schema/provider_surface.py:6-13`; `fkst-ops/schema/validator.py:231-269`; `fkst-packages/scripts/run.sh:792-815`)

The common `github_devloop_profile.version`, `github_devloop_profile.id`, and producer-defined `data` are **ASSUMED-UNVERIFIED** because the current operator exports defaults but the inspected topology does not record a contract-versioned profile value set. The declaration must not replace these markers until the pinned `board.github-control` provider publishes the accepted values. (`fkst-packages/.claude/skills/dogfood-github-devloop/dogfood.sh:53-60`; `fkst-ops/schema/validator.py:324-338`)

The platform package lists come from each target workspace selection. `packages` selects its workspace `[[package]]` entries; `substrate` names its external platform packages; `website`'s current checked-in manifest has no non-empty platform package list even though the schema requires one, so the website list below is **ASSUMED-UNVERIFIED** and copied from the host equivalence fixture pending reconciliation. (`fkst-packages/fkst.workspace.toml:11-84`; `fkst-substrate/fkst.workspace.toml:7-26`; `fkst-website/fkst.workspace.toml:17-21`; `fkst-ops/tests/host/host_run_equivalence_test.py:31-46`; `fkst-ops/schema/validator.py:289-295`)

### 10.1 `deployments/packages.toml`

```toml
schema = "fkst.ops.deployment.v1"

[[provider]]
id = "engine"
kind = "engine"
implementation = "fkst-ops:providers/engine.py"
contract = "fkst.ops.engine.v1"
configuration = { build_command = ["cargo", "build", "-p", "fkst-framework"] }

[[provider]]
id = "engine-board"
kind = "board.engine-durable"
implementation = "fkst-ops:providers/board_engine_durable.py"
contract = "fkst.ops.board.engine-durable.v1"
configuration = {}

[[provider]]
id = "github-board"
kind = "board.github-control"
implementation = "fkst-ops:providers/board_github_control.sh"
contract = "fkst.ops.board.github-control.v1"
configuration = {}

[[deployment]]
id = "packages"
target_identity = "ChronoAIProject/fkst-packages"

[deployment.github_devloop_profile]
version = "ASSUMED-UNVERIFIED"
id = "ASSUMED-UNVERIFIED"
data = {}
producer_binding = "github-board"

[deployment.sources.target]
lock_ref = "fkst-packages"
[deployment.sources.platform]
lock_ref = "fkst-packages"
[deployment.sources.engine]
lock_ref = "fkst-substrate"

[deployment.packages]
platform = [
  "github-devloop", "github-devloop-pr", "github-devloop-integration",
  "github-devloop-intake", "github-devloop-workflow", "github-devloop-decompose",
  "github-devloop-ops", "github-proxy", "github-external-pr-intake",
  "github-ratchet-migration-slicer", "fkst-substrate-ref-maintainer",
  "integration-coverage-producer", "idle-detector", "git-branch-detector",
  "github-devloop-worktree-gc",
]
host = []

[deployment.integration]
upstream_branch = "dev"
integration_branch = "machine:integration-branch"
rollup_merge = "auto"

[deployment.machine]
target_checkout = "packages-checkout"
platform_checkout = "packages-checkout"
engine_checkout = "engine-checkout"
engine_binary = "engine-binary"
durable = "packages-durable"
runtime = "packages-runtime"
logs = "packages-logs"
rate_pool = "shared-rate-pool"
bot_login = "github-bot-login"
managed_bot_set = "managed-bot-logins"

[deployment.providers]
engine = "engine"
board_engine_durable = "engine-board"
board_github_control = "github-board"
```

The identical target/platform lock reference and logical checkout encode the existing `HOST == PKGSRC` topology. (`fkst-packages/.claude/skills/dogfood-github-devloop/dogfood.sh:85-88`)

### 10.2 `deployments/substrate.toml`

```toml
schema = "fkst.ops.deployment.v1"

[[provider]]
id = "engine"
kind = "engine"
implementation = "fkst-ops:providers/engine.py"
contract = "fkst.ops.engine.v1"
configuration = { build_command = ["cargo", "build", "-p", "fkst-framework"] }

[[provider]]
id = "engine-board"
kind = "board.engine-durable"
implementation = "fkst-ops:providers/board_engine_durable.py"
contract = "fkst.ops.board.engine-durable.v1"
configuration = {}

[[provider]]
id = "github-board"
kind = "board.github-control"
implementation = "fkst-ops:providers/board_github_control.sh"
contract = "fkst.ops.board.github-control.v1"
configuration = {}

[[deployment]]
id = "substrate"
target_identity = "ChronoAIProject/fkst-substrate"

[deployment.github_devloop_profile]
version = "ASSUMED-UNVERIFIED"
id = "ASSUMED-UNVERIFIED"
data = {}
producer_binding = "github-board"

[deployment.sources.target]
lock_ref = "fkst-substrate"
[deployment.sources.platform]
lock_ref = "fkst-packages"
[deployment.sources.engine]
lock_ref = "fkst-substrate"

[deployment.packages]
platform = [
  "github-devloop", "github-devloop-pr", "github-devloop-integration",
  "github-devloop-intake", "github-devloop-workflow", "github-devloop-decompose",
  "github-devloop-ops", "github-proxy", "consensus", "github-external-pr-intake",
  "github-ratchet-migration-slicer", "fkst-substrate-ref-maintainer",
  "integration-coverage-producer", "idle-detector",
]
host = []

[deployment.integration]
upstream_branch = "dev"
integration_branch = "machine:integration-branch"
rollup_merge = "auto"

[deployment.machine]
target_checkout = "substrate-target-checkout"
platform_checkout = "substrate-platform-checkout"
engine_checkout = "engine-checkout"
engine_binary = "engine-binary"
durable = "substrate-durable"
runtime = "substrate-runtime"
logs = "substrate-logs"
rate_pool = "shared-rate-pool"
bot_login = "github-bot-login"
managed_bot_set = "managed-bot-logins"

[deployment.providers]
engine = "engine"
board_engine_durable = "engine-board"
board_github_control = "github-board"
```

The distinct target and platform source references and logical checkout names encode the existing separate `sub` and `pkgs` checkouts. (`fkst-packages/.claude/skills/dogfood-github-devloop/dogfood.sh:89-92`)

### 10.3 `deployments/website.toml`

```toml
schema = "fkst.ops.deployment.v1"

[[provider]]
id = "engine"
kind = "engine"
implementation = "fkst-ops:providers/engine.py"
contract = "fkst.ops.engine.v1"
configuration = { build_command = ["cargo", "build", "-p", "fkst-framework"] }

[[provider]]
id = "engine-board"
kind = "board.engine-durable"
implementation = "fkst-ops:providers/board_engine_durable.py"
contract = "fkst.ops.board.engine-durable.v1"
configuration = {}

[[provider]]
id = "github-board"
kind = "board.github-control"
implementation = "fkst-ops:providers/board_github_control.sh"
contract = "fkst.ops.board.github-control.v1"
configuration = {}

[[deployment]]
id = "website"
target_identity = "ChronoAIProject/fkst-website"

[deployment.github_devloop_profile]
version = "ASSUMED-UNVERIFIED"
id = "ASSUMED-UNVERIFIED"
data = {}
producer_binding = "github-board"

[deployment.sources.target]
lock_ref = "fkst-website"
[deployment.sources.platform]
lock_ref = "fkst-packages"
[deployment.sources.engine]
lock_ref = "fkst-substrate"

[deployment.packages]
platform = [
  "github-devloop", "github-devloop-pr", "github-devloop-integration",
  "github-devloop-intake", "github-devloop-workflow", "github-devloop-decompose",
  "github-devloop-ops", "github-proxy", "github-external-pr-intake",
  "github-ratchet-migration-slicer", "idle-detector",
]
host = ["site-board"]

[deployment.integration]
upstream_branch = "dev"
integration_branch = "machine:integration-branch"
rollup_merge = "auto"

[deployment.machine]
target_checkout = "website-target-checkout"
platform_checkout = "website-platform-checkout"
engine_checkout = "engine-checkout"
engine_binary = "engine-binary"
durable = "website-durable"
runtime = "website-runtime"
logs = "website-logs"
rate_pool = "shared-rate-pool"
bot_login = "github-bot-login"
managed_bot_set = "managed-bot-logins"

[deployment.providers]
engine = "engine"
board_engine_durable = "engine-board"
board_github_control = "github-board"
```

The distinct target and platform checkout names and `host = ["site-board"]` encode the existing website topology. (`fkst-packages/.claude/skills/dogfood-github-devloop/dogfood.sh:93-96`)

## 11. Validation

Each declaration is validated directly by the real validator from the hydrated, pin-verified checkout:

```sh
python3 <verified-fkst-ops-checkout>/schema/validator.py \
  deployments/packages.toml .fkst/machine-profile.toml fkst.lock
python3 <verified-fkst-ops-checkout>/schema/validator.py \
  deployments/substrate.toml .fkst/machine-profile.toml fkst.lock
python3 <verified-fkst-ops-checkout>/schema/validator.py \
  deployments/website.toml .fkst/machine-profile.toml fkst.lock
```

Direct absolute-path invocation works from any working directory because the script establishes its checkout import root before importing `schema.provider_surface`; it is equivalent to `python3 -m schema.validator`. The CLI accepts exactly `declaration`, `machine_profile`, and `lock`, emits resolved JSON on success, and exits `2` with a fail-closed diagnostic on validation failure. (`fkst-ops/schema/validator.py:15-18,364-393`; `fkst-ops/tests/schema/test_validator.py:241-257`)

The mechanism suite was executed in one process as `python3 -m pytest -q`: `132 passed, 21 skipped`, exit `0`. This is dynamic verification of the inspected checkout, not validation of any declaration in this repository.

The operational path uses the bootstrap so pin and tree authentication precede validation:

```sh
bootstrap/run.sh deployments/packages.toml \
  --machine-profile .fkst/machine-profile.toml --lock fkst.lock status
```

The bootstrap forwards the declaration and machine arguments to `fkst-ops`; `bin/fkst-ops` invokes `python3 -m schema.validator` before dispatching any action. (`fkst-ops/bootstrap/run.sh:60-90`; `fkst-ops/bin/fkst-ops:9-35`)

Validation is not considered complete with placeholders, synthetic fixtures, a copied validator, or a locally unpinned checkout. All three commands must use real first-cut pins, real provider executables, and the machine profile for the cutover host. (`fkst-ops/schema/validator.py:154-215`; `fkst-ops/deployments/packages/INSTALL.md:18-30`)

None of the three intended declarations passes the real validator today because this repository contains only this specification, not materialized declaration, lock, or machine-profile inputs. With the provider bindings above, `packages` and `substrate` remain blocked by accepted source pins, accepted profile values, and real machine paths; `website` has those same blockers plus unresolved platform composition. No placeholder declaration validation was run. (`fkst-ops/schema/validator.py:93-138,179-218,271-359`; `fkst-website/fkst.workspace.toml:17-21`)

## 12. Non-Goals

- Owning or duplicating any operational mechanism. (`fkst-ops/docs/superpowers/specs/2026-08-08-fkst-ops-extraction-design.md:38-45`)
- Embedding declarations in `fkst-ops` or in individual target repositories. (`fkst-ops/docs/superpowers/specs/2026-08-08-fkst-ops-extraction-design.md:10-14,47-55`)
- Inventing a schema, validator, provider framework, bootstrap framework, scheduler, global launcher, compatibility mode, or dual-write path. (`fkst-ops/docs/superpowers/specs/2026-08-08-fkst-ops-extraction-design.md:145-157,223-259,323-338`)
- Committing paths, logins, credentials, secrets, managed-bot membership, or device defaults. (`fkst-ops/schema/validator.py:30-42,83-87,116-150`)
- Discovering targets from a machine, organization, directory, or repository naming convention. (`fkst-ops/docs/superpowers/specs/2026-08-08-fkst-ops-extraction-design.md:38-45`)
- Changing package-owned fact semantics or `fkst-packages` check/test behavior. (`fkst-ops/docs/superpowers/specs/2026-08-08-fkst-ops-extraction-design.md:58-73,171-179`)
- Creating the GitHub repository before every condition in Section 9 passes. (`fkst-ops/schema/validator.py:176-215`; `fkst-ops/deployments/packages/INSTALL.md:18-27`)

## 13. Open Items

1. **Accepted source pins — ASSUMED-UNVERIFIED.** Record full revisions and canonical tree hashes for `fkst-ops`, `fkst-packages`, `fkst-substrate`, and `fkst-website`. (`fkst-ops/schema/validator.py:90-113`)
2. **GitHub devloop profile — ASSUMED-UNVERIFIED.** Publish and record the accepted profile `version`, `id`, and producer-owned `data` for all three declarations. (`fkst-ops/schema/validator.py:324-338`; `fkst-packages/.claude/skills/dogfood-github-devloop/dogfood.sh:53-60`)
3. **Website platform composition — ASSUMED-UNVERIFIED.** Reconcile the non-empty package set in the host equivalence fixture with the checked-in website workspace, which currently declares the platform source but no `packages` member. (`fkst-ops/tests/host/host_run_equivalence_test.py:31-46`; `fkst-website/fkst.workspace.toml:17-21`; `fkst-ops/schema/validator.py:289-295`)
4. **Host fixture coverage — ASSUMED-UNVERIFIED.** `fkst-ops` currently has 21 tests recorded as skipping unless `FKST_HOST_FIXTURE_ROOT` points at a host fixture checkout, so the covered host-layer behavior is unverified on a machine without that checkout. The named environment-variable guard is defined in the host entry, host run, equivalence, and local-iteration suites. (`fkst-ops/tests/host/host_entry_test.py:16-27`; `fkst-ops/tests/host/host_run_test.py:21-31`; `fkst-ops/tests/host/host_run_equivalence_test.py:22-28`; `fkst-ops/tests/host/host_run_local_iteration_test.py:15-26`)
5. **Five-action equivalence and first cutover — ASSUMED-UNVERIFIED.** No real deployment cutover or real-entry equivalence result is recorded; the `packages` owner and finite soak duration must be recorded before cutover. (`/private/tmp/claude-501/-Users-auric-fkst-packages/26dbcd24-03da-4fc6-b2aa-53a31f541b3a/scratchpad/ctx2.md:108-110`; `fkst-ops/docs/superpowers/specs/2026-08-08-fkst-ops-extraction-design.md:17-32`)

Repository ownership, naming, repository shape, bootstrap responsibility, the three provider entry points, provider publication semantics, and the build-command machine/deployment partition are settled. Accepted pins, profile values, website composition, host-fixture coverage, actual three-declaration validation, cutover ownership, and soak duration remain open, so sequencing is not complete. (`fkst-ops/schema/provider_surface.py:6-13`; `fkst-ops/schema/validator.py:93-138,202-269`; `fkst-ops/docs/superpowers/specs/2026-08-08-fkst-ops-extraction-design.md:349-356`)

⟦AI:FKST⟧
