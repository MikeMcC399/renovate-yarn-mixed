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

So for a robust experience, Renovate also needs to remove Yarn v1 before installing Corepack.

I have also seen Renovate log files where the Yarn v4 updates are listed before the ones for Yarn v1, and in this case both succeed.

I initially set up the minimal reproduction with `yarn-classic` & `yarn-modern` only, and there was no error. I then added the `yarn-modern-pnp` project sub-directory and then Renovate failed. It looks like this just affected the order of processing.

## Logs
