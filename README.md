# fkst-deployments

This repository is the machine-independent configuration for the live
`packages` and `substrate` FKST deployments. It contains deployment
declarations, source pins, shared policy, and documentation. It is configuration
only: scripts, programs, executable files, credentials, and machine-specific
paths do not belong here. All operational behavior lives in the pinned
`fkst-ops` mechanism.

## Machine setup

Create `.fkst/machine-profile.toml` from
`.fkst/machine-profile.example.toml`. Replace every angle-bracket placeholder
with the corresponding value for the operator's machine:

- absolute target, platform, engine, durable, runtime, log, and shared
  rate-pool paths;
- the absolute path to the executable `fkst-framework` engine binary;
- the GitHub bot login and complete managed-bot login set; and
- the machine's integration branch.

All referenced checkout, package, durable, runtime, log, and rate-pool
directories must exist, and the engine binary must exist and be executable.
The target and platform paths for `packages` must name the same checkout.

The real machine profile is ignored and must never be committed. It contains
absolute paths, credentials, managed identities, and machine defaults.

## Operate a deployment

Use the exact configuration-only owner form specified by `fkst-ops`:

```sh
<fkst-ops-checkout>/bin/fkst-ops --deployment-dir <deployment-repository> --declaration <deployment-repository>/deployment.toml --machine-profile <deployment-repository>/.fkst/machine-profile.toml --lock <deployment-repository>/fkst.lock status
```

For this repository, replace `<deployment-repository>/deployment.toml` with
either `<deployment-repository>/deployments/packages.toml` or
`<deployment-repository>/deployments/substrate.toml`. Replace both repository
placeholders with their absolute checkout paths. The entrypoint verifies its
checkout against the `fkst-ops` pin, hydrates that pinned mechanism when
necessary, and re-executes it with the original arguments.

`website` remains pending. Its declaration cannot ship because the deriver
currently reports exactly:

```text
error: website: external_sources(id=fkst-packages-platform).packages must not be empty
```
