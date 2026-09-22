# Yarn v4 lock file maintenance fails in monorepo with Yarn v1

## Current behavior

If both Yarn v1 Classic and Yarn v4 Modern (aka berry) projects are mixed in a GitHub monorepo and both are enabled for lock file maintenance, then the Yarn v4 projects may fail to update lock files with an error message similar to:

```console
File name: **/yarn.lock
error This project's package.json defines "packageManager": "yarn@4.18.0". However the current global version of Yarn is 1.22.22.

Presence of the "packageManager" field indicates that the project is meant to be used with Corepack, a tool included by default with all official Node.js distributions starting from 16.9 and 14.19.
Corepack must currently be enabled by running corepack enable in your terminal. For more information, check out https://yarnpkg.com/corepack.
```

If only Yarn Classic or only Yarn Modern is enabled for lock file maintenance, then each of these individually succeed.

This issue was previously observed in https://github.com/cypress-io/github-action where currently Yarn Modern sub-directories are disabled to avoid the issue:

```json
    {
      "matchUpdateTypes": ["lockFileMaintenance"],
      "matchFileNames": ["/^examples/yarn-modern/"],
      "enabled": false
    },
```

## Expected behavior

`lockFileMaintenance` should allow updating a mixture of Yarn v1 and Yarn v2 in the same repository.

## Assessment

In the failure situation, the logs show that Yarn 1.22.22 is installed when trying to run "yarn install --mode=update-lockfile" for a Yarn v4 project.

This can happen if the steps to update the lock file for a Yarn v1 project are executed before the ones for Yarn v4, and Yarn v1 remains globally installed.

Corepack https://github.com/nodejs/corepack#how-to-install advises:

> First uninstall your global Yarn and pnpm binaries (just leave npm). In general, you'd do this by running the following command:
>
> `npm uninstall -g yarn pnpm`

The logs below do not establish whether Corepack's shims were enabled or whether
the global Yarn v1 installation replaced or bypassed them. Installing Yarn v1
does not itself require running `corepack enable`; the relevant question is
which `yarn` executable Renovate leaves active for the subsequent Yarn Modern
command.

I have also seen Renovate log files where the Yarn v4 updates are listed before the ones for Yarn v1, and in this case both succeed.

I initially set up the minimal reproduction with `yarn-classic` & `yarn-modern` only, and there was no error. I then added the `yarn-modern-pnp` project sub-directory and then Renovate failed. It looks like this just affected the order of processing.

## Logs

This part of the README used AI assistance to extract and summarize log file content:

The following extract is from Renovate `44.103.0`, job log context
`798064d4-ffdf-40ce-bc0d-835afec1a5e8`.

Renovate correctly identifies the three lockfiles as Yarn Classic and Yarn
Modern, and detects the Yarn Modern package manager constraint:

```text
yarn.lock yarn-classic/yarn.lock is has no __metadata so is yarn 1
yarn.lock yarn-modern-pnp/yarn.lock is has __metadata so is yarn 2+
yarn.lock yarn-modern/yarn.lock is has __metadata so is yarn 2+

Found yarn constraint in package.json packageManager: 4.18.0
```

An earlier dependency update processes the Yarn Modern projects successfully,
then installs Yarn Classic globally:

```text
Generating yarn.lock for yarn-modern
Executing command: yarn install --mode=skip-build
stdout: Yarn 4.18.0

Generating yarn.lock for yarn-modern-pnp
Executing command: yarn install --mode=skip-build
stdout: Yarn 4.18.0

Resolved stable matching version: yarn 1.22.22
Executing command: install-tool yarn 1.22.22
stdout: Installing tool yarn@1.22.22...
stdout: 1.22.22
Executing command: yarn install --ignore-engines --ignore-platform --network-timeout 100000 --ignore-scripts
stdout: yarn install v1.22.22
stdout: success Saved lockfile.
```

The later `lockFileMaintenance` branch then processes Yarn Modern first. It
resolves Corepack, but still invokes the global `yarn` executable, which is now
Yarn `1.22.22`:

```text
branch: renovate/lock-file-maintenance

Generating yarn.lock for yarn-modern
Removing yarn-modern/yarn.lock first due to lock file maintenance upgrade
Resolved stable matching version: corepack 0.36.0
Executing command: yarn install --mode=skip-build
cwd: .../renovate-yarn-mixed/yarn-modern

error This project's package.json defines "packageManager": "yarn@4.18.0".
However the current global version of Yarn is 1.22.22.

Presence of the "packageManager" field indicates that the project is meant to
be used with Corepack.
Corepack must currently be enabled by running corepack enable in your terminal.
```

The same failure occurs for the Yarn PnP project:

```text
Generating yarn.lock for yarn-modern-pnp
Removing yarn-modern-pnp/yarn.lock first due to lock file maintenance upgrade
Executing command: yarn install --mode=skip-build
cwd: .../renovate-yarn-mixed/yarn-modern-pnp
error This project's package.json defines "packageManager": "yarn@4.18.0".
However the current global version of Yarn is 1.22.22.
```

Yarn Classic then succeeds, confirming that the failure is specific to the
Yarn Modern commands running after Yarn `1.22.22` has been installed:

```text
Generating yarn.lock for yarn-classic
Resolved stable matching version: yarn 1.22.22
Executing command: yarn install --ignore-engines --ignore-platform --network-timeout 100000 --ignore-scripts
stdout: yarn install v1.22.22
stdout: success Saved lockfile.
```

The resulting artifact errors are reported for exactly the two Yarn Modern
lockfiles, while `yarn-classic/yarn.lock` is updated:

```text
artifactErrors:
  yarn-modern/yarn.lock: global Yarn 1.22.22 is incompatible with Yarn 4.18.0
  yarn-modern-pnp/yarn.lock: global Yarn 1.22.22 is incompatible with Yarn 4.18.0

updatedArtifacts: ["yarn-classic/yarn.lock"]
context: "renovate/artifacts"
state: "red"
```

This supports the hypothesis that the Renovate worker's global tool state is
shared between branches in one run. Installing Yarn `1.22.22` for the Yarn
Classic project leaves a Yarn v1 executable active when the later
lock-file-maintenance branch tries to run Yarn `4.18.0`. Corepack `0.36.0` is
resolved successfully, but the logs do not show whether its installation
completed, whether its shims were enabled, or which executable is ultimately
found for `yarn`. The error could therefore be caused by an unsuccessful or
inactive Corepack installation, or by the Yarn v1 installation overriding or
bypassing the Corepack shim. Renovate should ensure that the executable used
for each project is selected independently and that Yarn Modern commands use
the requested Yarn version.
