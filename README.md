# fkst-deployments

This public repository is the machine-independent configuration for operating
FKST's `packages`, `substrate`, and `website` deployments. It owns deployment declarations,
source identities, the exact mechanism pin, and documentation. It contains no
operational code.

It must never contain scripts, programs, executable files, secrets, credentials,
or machine-specific paths. Those boundaries keep three repository roles separate:

1. [`fkst-ops`](https://github.com/ChronoAIProject/fkst-ops/tree/3a85e4c68cd39d36ba3c8e7b27f924e388a66350)
   is the public mechanism. Its entrypoint validates configuration, obtains and
   verifies the pinned mechanism, then dispatches operations.
2. `fkst-deployments` is public configuration only. [`fkst.lock`](fkst.lock)
   pins the mechanism by full revision and canonical tree hash and identifies
   deployment-operated source URLs. The declarations bind those sources to
   machine-local names and declare where a platform commit carries its engine
   revision.
3. `fkst-packages`, `fkst-substrate`, and `fkst-website` are operated targets.
   They do not depend on or invoke `fkst-ops`.

The dependency direction is targets <- declarations <- pinned mechanism. Target
repositories remain usable without this repository or `fkst-ops`.

## What ships

Three declarations ship:

- [`deployments/packages.toml`](deployments/packages.toml) operates
  `fkst-packages`. Its target and platform roles use one checkout; its engine
  comes from the revision in that platform commit's `.fkst/substrate-ref`.
- [`deployments/substrate.toml`](deployments/substrate.toml) operates
  `fkst-substrate`. It uses a separate `fkst-packages` platform checkout. The
  target follows its integration branch for engine development, while the
  separate shared engine checkout is detached at the revision in that platform
  commit's `.fkst/substrate-ref`.
- [`deployments/website.toml`](deployments/website.toml) operates
  `fkst-website`. It uses a separate `fkst-packages` platform checkout and loads
  the platform packages declared by the website workspace alongside the
  website-owned `site-board` package. GitHub writes are disabled in the
  declaration.

All three deployments name the same `engine-binary` stem, but each selected revision
is published once as the regular file `engine-binary-<E>`. Different platform
commits may therefore select different engine revisions without a shared pointer
or agreement check. Reuse recomputes the artifact's receipt-bound SHA-256 digest,
and host-run receives `E` and rejects a path naming another revision. The set of
writable records that selects the engine remains the revision file in the
platform commit.

Only an engine built from the declared source checkout is supported today.
Released-engine deployment is not supported. A later released-engine case can
be added as a second `engine_revision` arm while retaining the current `path`
arm unchanged.

## Machine setup

The machine profile is generated, never copied from this repository or edited by
hand. From a clean `fkst-ops` checkout whose revision and canonical tree match
[`fkst.lock`](fkst.lock), run:

```sh
DEPLOYMENT_REPO=$(pwd -P)
MACHINE_PROFILE="$HOME/.fkst/machine/profile.toml"
<pinned-fkst-ops-checkout>/bin/fkst-regenerate "$DEPLOYMENT_REPO" \
  --bot-login <machine-actor-login> \
  --github-credential-source <github-app-or-github-cli-user>
```

The generator derives every logical root named by the declarations, including
`engine-checkout`, and writes the profile to
`$HOME/.fkst/machine/profile.toml`. It hydrates each checkout from its declared
source before validation and atomically publishes the profile, declaration
manifest, and cadence LaunchAgent as one control generation.

The `packages` and `substrate` declarations take their integration branch from
that profile: each names it as `machine:integration-branch`, which the mechanism
resolves against `[defaults]` in the profile. The branch is named for the actor
driving the machine, because the actor is the integrating party. Its value is
therefore machine truth and is not committed here. The website declaration uses
its repository's established `integration` branch. `dev` is the declared upstream
branch for all three targets.

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
schedule policy. A declared deployment writes to GitHub unconditionally: the operator
sets that posture for every deployment child, and a declaration exposes no field to
select it. Starts and automatic restarts do not inherit an operator shell. Generation
reconciles the user LaunchAgent on every run and prints whether it is live.

The cadence does not update this repository. This keeps declaration and mechanism-pin
adoption deliberate: update the local `fkst-deployments` checkout, inspect the pin
change, then rerun generation. An operator who wants automatic adoption must arrange
a separate verified updater for this repository before generation; the cadence itself
will not cross that control boundary.

For this engine-derivation adoption, the declaration changes and the `fkst-ops`
`rev`/`tree_sha256` bump in `fkst.lock` must be one deployment commit. The old
mechanism rejects the new declaration and the new mechanism rejects the old
declaration, so either crossed state is deliberately inoperable. The executable
four-cell test is
`fkst-ops/tests/bootstrap/test_bootstrap.py::BootstrapTest::test_engine_revision_adoption_requires_one_declaration_and_pin_operation`.

Before selecting that deployment commit, quarantine a pre-existing shared engine
root if it has the known undeclared local origin. The old declarations do not use
this root, so this recoverable move does not disturb their running pair:

```sh
ENGINE_ROOT="$HOME/.fkst/machine/roots/engine-checkout"
DECLARED_ENGINE_ORIGIN="https://github.com/ChronoAIProject/fkst-substrate.git"
QUARANTINE="$ENGINE_ROOT.pre-derivation-adoption"
if [ -e "$ENGINE_ROOT" ] || [ -L "$ENGINE_ROOT" ]; then
  OBSERVED_ENGINE_ORIGIN=$(git -C "$ENGINE_ROOT" remote get-url origin)
  if [ "$OBSERVED_ENGINE_ORIGIN" != "$DECLARED_ENGINE_ORIGIN" ]; then
    test ! -e "$QUARANTINE" && test ! -L "$QUARANTINE"
    mv "$ENGINE_ROOT" "$QUARANTINE"
  fi
fi
```

Then select the one deployment commit and run `fkst-regenerate` once. Hydration
clones the missing root from the declared origin, and atomic control-generation
publication cannot expose a new profile with an old declaration manifest, or the
reverse. Keep the quarantine until post-adoption verification succeeds.

Set `DEPLOYMENT_REPO` to this repository's absolute path and choose `packages`,
`substrate`, or `website` as `NAME`, and set `MACHINE_PROFILE` to the generated profile.
Run the complete entrypoint from any `fkst-ops` checkout; it self-pins through
[`fkst.lock`](fkst.lock):

```sh
~/fkst-ops/bin/fkst-ops preflight \
  --deployment-dir "$DEPLOYMENT_REPO" \
  --declaration "$DEPLOYMENT_REPO/deployments/$NAME.toml" \
  --machine-profile "$MACHINE_PROFILE" \
  --lock "$DEPLOYMENT_REPO/fkst.lock"
```

Preflight validates the full configuration and exits without dispatching an
operation. To operate, replace the leading `preflight` with nothing and append
one action after the four options:

```sh
~/fkst-ops/bin/fkst-ops \
  --deployment-dir "$DEPLOYMENT_REPO" \
  --declaration "$DEPLOYMENT_REPO/deployments/$NAME.toml" \
  --machine-profile "$MACHINE_PROFILE" \
  --lock "$DEPLOYMENT_REPO/fkst.lock" status
```

The actions are `board`, `status`, `logs`, `restart`, and `sync`; `status` does
not mutate the deployment. `doctor` is a separately invocable action on the
same entrypoint. The exact dispatch surface is defined in
[`bin/fkst-ops`](https://github.com/ChronoAIProject/fkst-ops/blob/3a85e4c68cd39d36ba3c8e7b27f924e388a66350/bin/fkst-ops#L1-L184).

Known rough edge: `--deployment-dir` does not derive the declaration, machine
profile, and lock paths yet, so pass all four paths.

⟦AI:FKST⟧
