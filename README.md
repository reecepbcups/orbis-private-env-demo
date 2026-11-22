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

> `install` hooks the filter into your global git config (like `git lfs install`) and offers to create your identity:
>
> ```text
> No git-orbis identity found. Create one at ~/.ssh/git-orbis? [y/N] y
> saved. back it up, losing it means losing access.
> did:key:z6Mk...
> ```
>
> Send that `did:key:...` to someone who already has access.

## Start the Orbis stack

Needs docker. This is vera + 3 orbis nodes on localhost. It runs locally just for the PoC, but it can be hosted in any cloud docker environment.

```bash
make -C ~/git-orbis up
```

Don't run `make down` between demos, it wipes the chain and the `.env` pointer here dies with it.

## Get the .env

```bash
[ -d orbis-private-env-demo ] || git clone https://github.com/reecepbcups/orbis-private-env-demo
cd orbis-private-env-demo
```

If you're on the readers list, `.env` has the real values and you're done.

Fresh identity? You'll see `git-orbis: no access to .env ... leaving pointer`. Nothing breaks. Add yourself as a reader and push it:

```bash
# add a new entity
git orbis grant "$(git orbis whoami)"
git commit -am "add $(whoami) as reader" && git pull --rebase && git push

# still the pointer, your checkout is from before the grant
cat .env
# updates every file you can read now (directly or through a team)
git orbis refresh
# you now have access!
cat .env
```

This only works because the local stack lets anyone grant (it signs with the public devnet key). In a real setup someone who already has access runs `grant` for you, then you `git pull && git orbis refresh`.

## Private branches

`private/*` branches are encrypted, only `@ops` can read them ([.orbis/access](.orbis/access)). GitHub just sees the name and an encrypted bundle.

Push a branch if you are in `@ops`:

```bash
git orbis init  # once per clone, makes git push encrypt private/*
git checkout -b private/my-thing
git push origin private/my-thing
```

Grab the example one:

```bash
git orbis fetch-private private/my-thing
git checkout private/my-thing
```

Make it public (renames it, pushes real commits, deletes the encrypted copy, no undo):

```bash
git orbis private-publish private/my-thing my-new-public-branch
```


## Change the .env

Edit it and commit like any file. On commit the new contents go to Orbis and everyone in `.orbis/readers` gets access again (~20s, mostly waiting on blocks). Teammates `git pull` and get the new values.

## Things to know

- **No revoke.** Removing someone from `.orbis/readers` doesn't un-share what they've seen. Rotate the real secret.
- **History counts.** Old `.env` versions stay readable to readers.
- **Your identity is `~/.ssh/git-orbis`.** Lose it and you lose access. `ORBIS_READER_DID_PK` overrides it if you'd rather use a secret manager.
- **It's a POC.** See the [parent repo's known gaps](https://github.com/reecepbcups/git-orbis#known-gaps).
