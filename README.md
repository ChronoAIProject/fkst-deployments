# fkst-deployments

This public repository is the machine-independent configuration for operating
FKST's `packages` and `substrate` deployments. It owns deployment declarations,
exact source pins, and documentation. It contains no operational code.

It must never contain scripts, programs, executable files, secrets, credentials,
or machine-specific paths. Those boundaries keep three repository roles separate:

1. [`fkst-ops`](https://github.com/ChronoAIProject/fkst-ops/tree/18cfb18d74d74ec2927d6b2531d07985c69bb9ff)
   is the public mechanism. Its entrypoint validates configuration, obtains and
   verifies the pinned mechanism, then dispatches operations.
2. `fkst-deployments` is public configuration only. [`fkst.lock`](fkst.lock)
   pins the mechanism and all operated sources by full revision and canonical
   tree hash; the declarations bind those sources to machine-local names.
3. `fkst-packages`, `fkst-substrate`, and `fkst-website` are operated targets.
   They do not depend on or invoke `fkst-ops`.

The dependency direction is targets <- declarations <- pinned mechanism. Target
repositories remain usable without this repository or `fkst-ops`.

## What ships

Two declarations ship:

- [`deployments/packages.toml`](deployments/packages.toml) operates
  `fkst-packages`. Its target and platform roles use one checkout; its engine
  comes from the pinned `fkst-substrate` source.
- [`deployments/substrate.toml`](deployments/substrate.toml) operates
  `fkst-substrate`. It uses a separate `fkst-packages` platform checkout. The
  target is also the engine source, so target and engine use the same pinned
  source and checkout.

There is no website declaration. The
[`fkst-website` workspace manifest](https://github.com/ChronoAIProject/fkst-website/blob/fd3cc37505071d0e47c749069953754de0b596e3/fkst.workspace.toml)
declares its external platform source without a package composition. The
[authoritative deriver](https://github.com/ChronoAIProject/fkst-ops/blob/18cfb18d74d74ec2927d6b2531d07985c69bb9ff/ops/workspace_manifest.py#L179-L211)
therefore reports exactly:

```text
error: website: external_sources(id=fkst-packages-platform).packages must not be empty
```

## Machine setup

From this repository, create the local profile:

```sh
cp .fkst/machine-profile.example.toml .fkst/machine-profile.toml
```

Replace every angle-bracket placeholder in the copy. The
[`example profile`](.fkst/machine-profile.example.toml) names the required
checkout, durable, runtime, log, shared rate-pool, engine-binary, discovered-tool,
bot-login, managed-bot-set, and integration-branch values. Paths under `roots`,
`binaries`, and `tools` must be absolute. Target, platform, engine, durable, and declared
package directories must exist; the engine binary must exist and be executable,
as enforced by the
[validator](https://github.com/ChronoAIProject/fkst-ops/blob/18cfb18d74d74ec2927d6b2531d07985c69bb9ff/schema/validator.py#L151-L204).

`.fkst/machine-profile.toml` is ignored and must never be committed. It contains
machine paths, managed identities, and machine defaults.

Both declarations take their integration branch from that profile: each names it
as `machine:integration-branch`, which the mechanism resolves against `[defaults]`
in the profile. The branch is named for the actor driving the machine, because the
actor is the integrating party. Its value is therefore machine truth and is not
committed here; `dev` is the declared upstream branch for both targets.

## Operating identity: an App or a person

A machine may be driven by a GitHub App installation or by a person's own account.
The mechanism does not distinguish them. `[credentials] github-bot-login` in the
machine profile and every entry of `deployment.managed_bot_logins` hold a login,
not an account type: an App identity carries the `[bot]` suffix, a personal
account does not. The platform strips a trailing `[bot]` before comparing and is
otherwise case-sensitive, so a personal login must be written exactly as GitHub
spells it.

The field is named `managed_bot_logins` for historical reasons and now also holds
personal logins; the name is narrower than the values it carries. Renaming it is a
mechanism change and is deliberately not done here.

What the roster means is unchanged either way: these are the logins whose activity
this fleet treats as its own automation rather than as an outside contribution.

`managed_bot_logins` is fleet policy: it lists every actor in the fleet, so one
committed declaration serves every machine. Each machine's own actor is machine
truth and lives in its profile; the mechanism binds the two by requiring the
profile's actor to be a member of the declared roster. Earlier text here described
this field as a single machine-named value; the mechanism requires the profile's
actor to be a member of the declared roster rather than equal to it.

How a machine authenticates is machine truth as well, and it now reaches the
declarations through the same `machine:` form the integration branch uses. The
credential provider's `source` is `machine:credential-source`, and each profile
resolves it to one of two closed values: `github-app` for an appliance driven by a
GitHub App installation, or `github-cli-user` for one driven by a person's own
account, which takes the GitHub CLI's stored credential for the declared login.

The two prove deliberately different things, and neither is strictly safer. An
installation token cannot resolve `/user`, so the App path proves target access but
not the principal login. The user path compares the login exactly and checks the
target's reported push permission on every refresh, but the credential carries the
whole account's authority rather than an installation's repository-scoped authority.
An appliance that ingests untrusted issue and pull-request text should weigh that
blast radius deliberately. The mechanism's `SPEC.md` states both limits.

## Preflight and operate

`cadence_enabled` and `cadence_interval_seconds` are required repository-level
schedule policy. Each deployment also requires `github_write_enabled`; starts and
automatic restarts use that declared value and do not inherit an operator shell.
Generation reconciles the user LaunchAgent on every run and prints whether it is
live.

The cadence does not update this repository. This keeps declaration and mechanism-pin
adoption deliberate: update the local `fkst-deployments` checkout, inspect the pin
change, then rerun generation. An operator who wants automatic adoption must arrange
a separate verified updater for this repository before generation; the cadence itself
will not cross that control boundary.

Set `DEPLOYMENT_REPO` to this repository's absolute path and choose `packages`
or `substrate` as `NAME`. Run the complete entrypoint from any `fkst-ops`
checkout; it self-pins through [`fkst.lock`](fkst.lock):

```sh
~/fkst-ops/bin/fkst-ops preflight \
  --deployment-dir "$DEPLOYMENT_REPO" \
  --declaration "$DEPLOYMENT_REPO/deployments/$NAME.toml" \
  --machine-profile "$DEPLOYMENT_REPO/.fkst/machine-profile.toml" \
  --lock "$DEPLOYMENT_REPO/fkst.lock"
```

Preflight validates the full configuration and exits without dispatching an
operation. To operate, replace the leading `preflight` with nothing and append
one action after the four options:

```sh
~/fkst-ops/bin/fkst-ops \
  --deployment-dir "$DEPLOYMENT_REPO" \
  --declaration "$DEPLOYMENT_REPO/deployments/$NAME.toml" \
  --machine-profile "$DEPLOYMENT_REPO/.fkst/machine-profile.toml" \
  --lock "$DEPLOYMENT_REPO/fkst.lock" status
```

The actions are `board`, `status`, `logs`, `restart`, and `sync`; `status` does
not mutate the deployment. `doctor` is a separately invocable action on the
same entrypoint. The exact dispatch surface is defined in
[`bin/fkst-ops`](https://github.com/ChronoAIProject/fkst-ops/blob/18cfb18d74d74ec2927d6b2531d07985c69bb9ff/bin/fkst-ops#L1-L184).

Known rough edge: `--deployment-dir` does not derive the declaration, machine
profile, and lock paths yet, so pass all four paths.

## Open rollout work

- No deployment has cut over from the previous operator.
- The five-action equivalence matrix has not run against both real entries.
- The website deployment is pending its platform composition.

⟦AI:FKST⟧
