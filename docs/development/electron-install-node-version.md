---
title: Electron Installation Fails
description: "Fix 'Electron failed to install correctly' when running npm install / npm run start"
---

# Electron Installation Fails

## Symptom

After cloning the repository and running `npm install`, `npm run start` fails with:

```text
Error: Electron failed to install correctly, please delete node_modules/electron and try installing again
    at getElectronPath (.../node_modules/electron/index.js:17:11)
```

Deleting `node_modules/electron` and reinstalling does not fix the problem.

## Cause

This is not caused by outdated npm dependencies. It is a Node.js version
incompatibility in Electron's install script.

Electron 33 uses `extract-zip@2.0.1`, which depends on `yauzl@2.10.0`. On
Node.js **24.16+** and **26+**, `yauzl` extracts only the first file from the
Electron archive and then stops. The install script still exits with code `0`,
so npm reports no error, but the Electron binary is missing.

Typical signs:

- `node_modules/electron/dist/` contains only `locales/es.pak`.
- `node_modules/electron/path.txt` does not exist.
- The downloaded archive in `~/.cache/electron/` is complete (around 106 MB).

If you are using a Conda environment, make sure it actually provides Node.js.
An empty environment falls back to the system Node.js, which may be a version
that is too new:

```sh
conda list
which node
node -v
```

## Solution: Use Node.js 22 LTS

The project pins Node.js 18.20.5 in `package.json` (Volta). Node.js 22 LTS is
also compatible and receives security updates. Node.js 18, 20, or 22 all avoid
the `extract-zip` bug.

### Conda

```sh
conda activate upscayl
conda install -c conda-forge nodejs=22 -y

# Verify that the Conda Node.js is used
which node
node -v
```

Then reinstall Electron:

```sh
rm -rf node_modules/electron
npm install
```

If the rest of `node_modules` is already installed, rerunning only Electron's
install script is enough:

```sh
node node_modules/electron/install.js
```

### Verify

```sh
cat node_modules/electron/path.txt
# expected: electron

cat node_modules/electron/dist/version
# expected: 33.2.1

node_modules/electron/dist/electron --version
# expected: v33.2.1

npm run start
```

A complete `node_modules/electron/dist/` contains roughly 74 files and about
263 MB on Linux x64.

## Manual Workaround

If you cannot change the Node.js version, you can extract the already
downloaded archive manually:

```sh
mkdir -p node_modules/electron/dist
unzip -q ~/.cache/electron/*/electron-v33.2.1-linux-x64.zip \
  -d node_modules/electron/dist
printf 'electron' > node_modules/electron/path.txt
chmod +x node_modules/electron/dist/electron
```

`path.txt` must contain `electron` **without a trailing newline**, otherwise
Electron tries to spawn a path that includes the newline.

This is only a workaround. The extraction bug will return on the next
reinstall with a Node.js version that is too new.

## Do the Dependencies Need to Be Upgraded?

No. Upgrading the project's dependencies is not required for this error.

Electron 45 and newer use `@electron-internal/extract-zip` instead of
`extract-zip`, so they are not affected, but upgrading Electron is a much
larger change and may require code migration. Using a supported Node.js LTS
version is the smaller and safer fix.

## References

- `extract-zip` issue #154: https://github.com/max-mapper/extract-zip/issues/154
- Electron 33 package: https://www.npmjs.com/package/electron/v/33.2.1

## Verified Environment

- Linux x64 (CachyOS)
- Node.js 22.23.4 (Conda Forge)
- npm 10.9.9
- Electron 33.2.1
