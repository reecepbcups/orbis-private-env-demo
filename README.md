# orbis-private-env-demo

This repo is public, but its `.env` isn't. Open [.env](.env) on GitHub and all you'll see is a pointer:

```text
orbis-git v1
object: <id>
```

The real values live in [Orbis](https://github.com/sourcenetwork/orbis-rs), a threshold secret engine. Only people listed in [.orbis/readers](.orbis/readers) get them. Everyone else clones fine, they just keep the pointer.

The tool lives in **[reecepbcups/git-orbis](https://github.com/reecepbcups/git-orbis)**, check its README for how it works under the hood.

> Heads up: the Orbis stack for this repo runs on my laptop right now, so you won't get plaintext until it's hosted somewhere you can reach. The flow below is the real one though.

## Setup (once per machine)

Safe to re-run, it skips whatever you already have.

```bash
[ -d ~/git-orbis ] || git clone https://github.com/reecepbcups/git-orbis ~/git-orbis
make -C ~/git-orbis tools                        # first build is slow
RC=~/.$(basename "$SHELL")rc                         # ~/.zshrc or ~/.bashrc
grep -qs 'git-orbis/bin' "$RC" || echo 'export PATH="$HOME/git-orbis/bin:$PATH"' >> "$RC"
export PATH="$HOME/git-orbis/bin:$PATH"
git orbis install
```

## Start the Orbis stack

Needs docker. This is vera + 3 orbis nodes on localhost. It runs locally just for the PoC, but it can be hosted in any cloud docker environment.

```bash
make -C ~/git-orbis up
```

Don't run `make down` between demos, it wipes the chain and the `.env` pointer here dies with it.

`install` hooks the filter into your global git config (like `git lfs install`) and offers to create your identity:

```text
No git-orbis identity found. Create one at ~/.ssh/git-orbis? [y/N] y
saved. back it up, losing it means losing access.
did:key:z6Mk...
```

Send that `did:key:...` to someone who already has access.

## Get the .env

```bash
[ -d orbis-private-env-demo ] || git clone https://github.com/reecepbcups/orbis-private-env-demo
cd orbis-private-env-demo
```

That's it. If you're on the readers list, `.env` has the real values.

Not on it yet? You'll see `git-orbis: no access to .env ... leaving pointer`. Nothing breaks. Once you've been granted, run:

```bash
git pull && git orbis refresh
```

## Give someone access

```bash
git orbis grant did:key:z6Mk...
git commit -am "give <name> access" && git push
```

## Change the .env

Edit it and commit like any file. On commit the new contents go to Orbis and everyone in `.orbis/readers` gets access again (~20s, mostly waiting on blocks). Teammates `git pull` and get the new values.

## Things to know

- **No revoke.** Removing someone from `.orbis/readers` doesn't un-share what they've seen. Rotate the real secret.
- **History counts.** Old `.env` versions stay readable to readers.
- **Your identity is `~/.ssh/git-orbis`.** Lose it and you lose access. `ORBIS_READER_DID_PK` overrides it if you'd rather use a secret manager.
- **It's a POC.** See the [parent repo's known gaps](https://github.com/reecepbcups/git-orbis#known-gaps).
