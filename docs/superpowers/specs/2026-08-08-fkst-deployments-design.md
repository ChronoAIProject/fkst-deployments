# fkst-deployments Design

**Status:** Implemented for packages and substrate; website pending
**Date:** 2026-08-08

## Purpose

`fkst-deployments` is the version-controlled, machine-independent configuration
repository for FKST deployments. It owns declarations, source pins, deployment
policy, package composition, and logical references to machine values. All
operational behavior remains in pinned `fkst-ops`.

There are two live declarations: `deployments/packages.toml` and
`deployments/substrate.toml`. A third deployment, website, is pending because
its target workspace does not yet declare a non-empty platform package set.

## Repository Layout

```text
fkst-deployments/
|-- .gitignore
|-- README.md
|-- fkst.lock
|-- deployments/
|   |-- packages.toml
|   `-- substrate.toml
|-- .fkst/
|   |-- machine-profile.example.toml
|   `-- machine-profile.toml          # untracked
`-- docs/superpowers/specs/
    `-- 2026-08-08-fkst-deployments-design.md
```

The lock retains the `fkst-website` source pin so the pending declaration can
land without changing repository-wide source identity.

## Ownership Boundary

Committed deployment truth includes target identity, source lock references,
package composition, integration policy, the GitHub devloop profile, and
provider bindings. Machine truth includes absolute checkout, durable, runtime,
log, rate-pool, and binary paths; bot identity; managed bot membership; and the
machine-default integration branch. Machine truth stays in the ignored machine
profile and is referenced only by logical name.

`BOT` and `MANAGED_BOT_LOGINS` are machine values and must never appear in
`github_devloop_profile.data`.

## GitHub Devloop Profile

Every live declaration selects the authoritative fleet-wide GitHub devloop
policy:

```toml
[deployment.github_devloop_profile]
version = "1"
id = "fleet-github-devloop-policy"
data = { upstream_branch = "dev", integration_branch = "integration", rollup_merge = "auto", poll_label_prefix = "fkst-dev:", organization = "ChronoAIProject", authorize_org_members = 0 }
producer_binding = "github-board"
```

Version `1` is the first concrete semantic revision. The ID names the shared
policy rather than either target repository, so both declarations can truthfully
refer to the same single source of fleet-wide policy. These values are measured from
`fkst-packages/.claude/skills/dogfood-github-devloop/dogfood.sh:53-61`.

## Platform Composition

Platform composition is derived by
`fkst-packages/.claude/skills/dogfood-github-devloop/workspace_manifest.py
platform-packages`; it is not copied from a host fixture. Both live declarations
use this exact ordered set:

```text
github-devloop github-devloop-pr github-devloop-integration github-devloop-intake github-devloop-workflow github-devloop-decompose github-devloop-ops github-proxy github-external-pr-intake github-ratchet-migration-slicer fkst-substrate-ref-maintainer integration-coverage-producer idle-detector git-branch-detector github-devloop-worktree-gc
```

The TOML representation is:

```toml
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
```

## Pending Website Deployment

There is no live `deployments/website.toml`. Running the same deriver against
`fkst-website/fkst.workspace.toml` fails exactly:

```text
error: website: external_sources(id=fkst-packages-platform).packages must not be empty
```

The website declaration lands unchanged the day
`fkst-website/fkst.workspace.toml` declares its platform packages. Until then,
website is excluded from declaration inventory, validation commands, live
deployment counts, and operational examples.

## Provider and Mechanism Contract

Each declaration binds the three published `fkst-ops` providers: `engine`,
`board.engine-durable`, and `board.github-control`. The engine build command is
the shell-free argv `["cargo", "build", "-p", "fkst-framework"]`.

This repository owns no bootstrap or executable code. The `bin/fkst-ops`
entrypoint in an available mechanism checkout verifies its checkout against the
deployment-owned pin, hydrates the pinned `fkst-ops` mechanism when necessary,
and re-executes it with the original arguments.

## Validation

From the repository root, validate both live declarations with the real
validator and a machine profile whose paths exist:

```sh
python3 <fkst-ops-checkout>/schema/validator.py deployments/packages.toml <machine-profile> fkst.lock
python3 <fkst-ops-checkout>/schema/validator.py deployments/substrate.toml <machine-profile> fkst.lock
```

Successful validation emits resolved JSON and exits zero. Validation is not
complete if a live declaration contains placeholders, uses nonexistent machine
paths, or binds an unpublished provider. The repository gate also requires
`ASSUMED-UNVERIFIED` to occur zero times in every declaration and requires no
committed file outside `.git` to have an executable bit.

## Operation

The mechanism exposes `board`, `status`, `logs`, `restart`, `sync`, and
`doctor`. Operate it using the configuration-only owner form:

```sh
<fkst-ops-checkout>/bin/fkst-ops --deployment-dir <deployment-repository> --declaration <deployment-repository>/deployments/packages.toml --machine-profile <deployment-repository>/.fkst/machine-profile.toml --lock <deployment-repository>/fkst.lock status
```

`packages` remains the first adopter. Adding another deployment requires one
validated declaration and any new source pin it references, with no source
change in `fkst-ops`.
