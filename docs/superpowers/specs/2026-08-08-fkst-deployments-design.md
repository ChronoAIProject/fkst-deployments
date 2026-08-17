# fkst-deployments Design

**Status:** Implemented for packages and substrate; website pending
**Date:** 2026-08-08

## Purpose

`fkst-deployments` is the version-controlled, machine-independent configuration
repository for FKST deployments. It owns declarations, source identities, deployment
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
`-- docs/superpowers/specs/
    `-- 2026-08-08-fkst-deployments-design.md
```

The lock retains the `fkst-website` source identity so the pending declaration can
land without changing repository-wide source identity.

## Ownership Boundary

Committed deployment truth includes target identity, source lock references,
the path that derives the engine revision from a file in the platform commit,
package composition, integration policy, the GitHub devloop profile, and
provider bindings. The set of writable records that can select which engine
executes goes from three to one: that revision file in the platform commit.
This is the narrow authority reduction established by the mechanism. Machine
truth includes absolute checkout, durable, runtime,
log, rate-pool, and binary paths; bot identity; managed bot membership; and the
machine-default integration branch. Machine truth stays in the generated
`$HOME/.fkst/machine/profile.toml` and is referenced only by logical name.
`[deployment.machine]` fields
name their logical values directly; `integration_branch` reaches the profile's
`[defaults]` through a `machine:` prefix, so no declaration carries a branch
value that differs per machine.

## Operating Identity

A machine's operating actor may be a GitHub App installation or a person's own
account; the mechanism treats them identically. Both `[credentials]
github-bot-login` and the entries of `deployment.managed_bot_logins` are logins,
not account types. An App login carries the `[bot]` suffix and a personal login
does not; the platform strips a trailing `[bot]` before comparing and is otherwise
case-sensitive, so a personal login must match GitHub's own spelling exactly.

The roster's meaning does not depend on account type: it enumerates the actors
whose activity this fleet reads as its own automation rather than as an outside
contribution. The field name `managed_bot_logins` predates personal-account
operation and is therefore narrower than the values it now carries; renaming it
would be a mechanism change and is not made here.

Membership is verifiable. An entry that does not resolve to an existing GitHub
account is not a pending note but a false entry, and the repository gate requiring
`ASSUMED-UNVERIFIED` to occur zero times in every declaration applies to it.

`managed_bot_logins` is the standing exception. The mechanism requires it as a
literal one-element list and compares the profile's resolved `managed-bot-set`
against it, so this machine-named value stays in committed configuration until
the mechanism accepts a `machine:` reference there too.

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

Deployment-operated source entries identify Git repositories and forbid a
`resolved` table. Only the `fkst-ops` mechanism entry retains an exact revision
and canonical tree hash. Target and platform checkouts follow the integration
branch; `engine-checkout` is separate and detached at the revision derived from
the platform commit. Both deployments use one binary stem, but publication and
launch use separate regular files named `engine-binary-<E>`, so differing
revisions do not contend for a mutable pointer.

A declaration can select `engine_revision.path`, and host-run can accept a
captured platform tree distinct from the project tree when their Git object
histories prove the repository relationship. The engine provider publishes a
revision-addressed file create-if-absent and records its content digest.

Only the build-from-source case is addressed today: `engine_revision.path` is
read from the platform commit and the selected source revision is built in the
engine checkout. A deployment that consumes a released engine is not supported.
Such support can later add a second arm to `engine_revision` while preserving
the existing `path` arm, so today's declarations do not need a breaking change.

This repository owns no bootstrap or executable code. The `bin/fkst-ops`
entrypoint in an available mechanism checkout verifies its checkout against the
deployment-owned pin, hydrates the pinned `fkst-ops` mechanism when necessary,
and re-executes it with the original arguments.

## Engine-derivation adoption

Publish the new mechanism commit first without changing a deployment lock. Then
land one deployment commit containing both declarations' `engine_revision` and
separate `engine_checkout` fields together with the new mechanism `rev` and
`tree_sha256`. These changes are one operation because the old mechanism rejects
the new declaration shape and the new mechanism rejects the old shape. The real
pin/re-exec four-cell matrix is executable as:

```sh
python3 -m pytest -q tests/bootstrap/test_bootstrap.py::BootstrapTest::test_engine_revision_adoption_requires_one_declaration_and_pin_operation
```

After checking out that deployment commit, use the pinned mechanism's generator:

```sh
<pinned-fkst-ops-checkout>/bin/fkst-regenerate <deployment-repository> \
  --bot-login <machine-actor-login> \
  --github-credential-source <github-app-or-github-cli-user>
```

That invocation derives the new `engine-checkout` root and atomically publishes
the profile, declaration manifest, and cadence LaunchAgent under
`$HOME/.fkst/machine`. Those three control files are one publication operation so
cadence cannot observe files from different generations. No generated file is
hand-edited during adoption.

An existing `engine-checkout` whose `origin` is not the declared GitHub URL must
be moved to a non-conflicting sibling quarantine before the deployment commit is
selected. The old declarations do not refer to this root. The generator then
clones the declared source into the missing path; changing the old checkout's
`origin` in place is not used as provenance remediation. This quarantine step is
recoverable and does not need to be atomic with control publication.

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
<fkst-ops-checkout>/bin/fkst-ops --deployment-dir <deployment-repository> --declaration <deployment-repository>/deployments/packages.toml --machine-profile "$HOME/.fkst/machine/profile.toml" --lock <deployment-repository>/fkst.lock status
```

`packages` remains the first adopter. Adding another deployment requires one
validated declaration and any new source binding it references, with no source
change in `fkst-ops`.
