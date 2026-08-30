# dep-sandbox

`ds` is a small bash wrapper that runs the risky half of your package manager commands inside a sandbox, so a malicious install script can't rifle through your home directory, your keychain, or your environment variables.

## Why

Supply chain attacks on npm and friends mostly do the same thing: a compromised package ships an install-time script that grabs `~/.ssh`, browser data, shell history and any API keys sitting in your environment, then posts them somewhere unpleasant. Package managers are adding good protections (cooldown windows, script approval, registry malware scanning), but the blunt problem remains that `npm install` runs other people's code with your full user privileges.

`ds` narrows that. Install-time code still runs, but inside a [nono](https://nono.sh) sandbox.

A note on platforms: this was built and tested on macOS, but nothing about the design is Mac-only. nono itself supports Linux, and the only macOS-specific pieces here are the keychain lookups, which nono's [credential injection](https://nono.sh/docs/cli/features/credential-injection#linux) can source from other secret stores. We just don't have a Linux box to test on, so a tested PR would be very welcome.

## What it does

- Dependency-mutating commands (`install`, `add`, `require`, `update`, `uninstall` and so on) for npm, bun, uv, pip and composer run inside the sandbox, using the `dep-sandbox.json` profile in this repo.
- Everything else (`npm run dev`, `composer dump-autoload`, `uv run`, `pip list`) is passed straight through to the real tool, untouched.
- `npx` and `bunx` are always sandboxed.
- In a Lando project (`.lando.yml` present), composer commands defer to `lando composer`.

Inside the sandbox, the profile:

- strips the environment down to a short allow-list (`PATH`, `HOME`, `TERM` and a few friends)
- denies reads of credentials, keychains, browser data, shell history and shell config files
- allows writes only to the project directory and the package managers' own cache directories
- routes network traffic through nono's filtering proxy

Installs run natively (no Docker, no VM), so native node modules, wheels and virtualenvs all work on the host afterwards.

## Prerequisites

- [nono](https://nono.sh) (`brew install nono`), built and tested against version 0.74
- whichever package managers you actually use (npm, bun, uv, pip, composer)
- [Lando](https://lando.dev/), only if you use it for local development

## Getting started

```bash
git clone https://github.com/ohnotnow/dep-sandbox.git
cd dep-sandbox
ln -s "$PWD/ds" /usr/local/bin/ds
```

Then use your package managers as normal, with `ds` in front:

```bash
ds npm install left-pad
ds composer require monolog/monolog
ds uv add requests
ds npm run dev        # passthrough, no sandbox
```

To see what `ds` would do without running anything:

```bash
DS_DRY_RUN=1 ds npm install left-pad
```

And to watch nono's own banner and session summary:

```bash
DS_VERBOSE=1 ds npm install left-pad
```

## Private composer registries

Private registries normally mean an `auth.json` on disk or credentials in your environment. `ds` avoids that: nono's credential proxy injects the auth at the network layer, host-side, so the sandboxed composer process never sees the credential at all. There is no `auth.json`, and nothing for a malicious postinstall to steal.

The convention is one keychain entry per registry, service `ds-registry`, account set to the registry hostname, value in `username:password` format:

```bash
security add-generic-password -U -s ds-registry -a composer.fluxui.dev -w
```

Run without a value, it prompts with hidden input. If you already have the credentials in environment variables from an older setup, you can pass them directly, and this is safe for shell history because history stores the unexpanded variable names:

```bash
security add-generic-password -U -s ds-registry -a composer.fluxui.dev -w "${FLUX_USERNAME}:${FLUX_LICENSE_KEY}"
```

Never paste the literal values inline, those do land in history. Once the keychain entry exists, delete the old `export` lines from your shell config.

On Linux there is no `security` command; swap the lookup in the profile's `credential_capture` block for your secret store of choice (`secret-tool`, `pass`, or any of the backends in nono's [credential injection docs](https://nono.sh/docs/cli/features/credential-injection#linux)).

Each registry also needs a route in `dep-sandbox.json`. The repo ships with `composer.fluxui.dev` (the [FluxUI](https://fluxui.dev/) private registry) as a worked example; copy the `custom_credentials` block and the matching `credential_capture` entry, change the hostname, and add the hostname to `allow_domain`. The capture command reads the hostname from `$NONO_REQUEST_HOST`, so it works unchanged for any registry that follows the keychain convention.

## Lando projects

Lando's container runs outside the sandbox, so private registries there still need a traditional per-project `auth.json`. Rather than guessing up front, `ds` lets composer fail, reads the hostname out of the 401 error, and prints the exact commands to fix it. The fix is:

```bash
ds auth composer.fluxui.dev
```

which writes the project's `auth.json` from the same keychain entry, and adds it to `.gitignore` if it isn't covered already.

One caveat: pick one install path per project. A `composer.lock` resolved by your host PHP can be refused by a Lando container running an older PHP, so don't mix sandboxed installs and Lando installs in the same project.

## Limitations worth knowing

- The project directory is the accepted blast radius. A malicious script can still tamper with the mounted tree, poison your lockfile, or read anything in the project, including a project-local `.npmrc` or `auth.json`.
- The domain allowlist is a tripwire, not a wall. Registries and GitHub have to be reachable for installs to work, and a determined attacker can exfiltrate to an allowlisted host they control an account on.
- Wrapper discipline is on you. `ds npm install` is sandboxed; a bare `npm install` typed on autopilot is not.
- nono is pre-release. Flags and profile fields change between versions, so expect the occasional breakage until it settles.

## Contributing

Clone or fork the repo, make your change, and run the routing matrix to check nothing regressed:

```bash
DS_DRY_RUN=1 ./ds npm install x
DS_DRY_RUN=1 ./ds npm run dev
DS_DRY_RUN=1 ./ds composer require foo/bar
```

Pull requests and issues welcome, especially profile fixes for package manager paths I haven't tripped over yet.

## Licence

MIT. See [LICENSE](LICENSE).
