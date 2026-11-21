# orbis-private-env-demo

This repo is public, but its `.env` isn't. Open [.env](.env) on GitHub and all you'll see is a pointer:

```text
orbis-git v1
object: <id>
```

The real values live in [Orbis](https://github.com/sourcenetwork/orbis-rs), a threshold secret engine. Only the people listed in [.orbis/readers](.orbis/readers) can get them back. Everyone else can clone fine, they just keep the pointer.

The tooling (`git-orbis`, the local Orbis stack, tests) is in the parent repo: **[reecepbcups/orbis-git-poc](https://github.com/reecepbcups/orbis-git-poc)**. Check its README for how it works under the hood.

> Heads up: right now the Orbis stack for this repo runs on my laptop (localhost). So the guide below shows the real flow, but you'll only get plaintext once the stack is hosted somewhere you can reach. Until then you can run the whole thing yourself with your own stack (see the bottom).

## Getting the .env

### 1. Get the tool

```bash
git clone https://github.com/reecepbcups/orbis-git-poc ~/orbis-git-poc
cd ~/orbis-git-poc && make tools        # builds the patched orbis cli-tool, first time is slow
export PATH=~/orbis-git-poc/bin:$PATH   # put this in your shell rc
```

### 2. Pick an identity

Your identity is just a secret seed. Treat it like a password and keep it in your shell rc or a secret manager.

```bash
export ORBIS_READER_DID_PK="$(openssl rand -hex 32)"   # once, then save it
git orbis whoami                                         # prints your did:key:z6Mk...
```

### 3. Ask for access

Send your `did:key:...` to someone who already has access. They run this in their clone and push:

```bash
git orbis grant did:key:z6Mk...
git commit -am "give <you> access to .env" && git push
```

### 4. Clone it

Run `git orbis init` *before* checkout, so the filter is there when `.env` gets written:

```bash
git clone --no-checkout https://github.com/reecepbcups/orbis-private-env-demo
cd orbis-private-env-demo
git orbis init
git checkout main
cat .env   # real values
```

If you don't have access you'll see `git-orbis: no access to .env ... leaving pointer` and `.env` stays as the pointer. Nothing breaks.

Want every clone to just work, like git-lfs? Run `git orbis install` once. It sets the filter in your global git config, so a plain `git clone` decrypts on its own. (This one's new and not tested yet, the `init` route above is.)

## Changing the .env

Just edit it and commit like normal:

```bash
echo "NEW_KEY=value" >> .env
git commit -am "add NEW_KEY"
git push
```

On commit the new contents go to Orbis and everyone in `.orbis/readers` gets access again. That takes about 20s, mostly waiting on chain blocks. Teammates `git pull` and get the new values.

## Run it yourself

Don't want to wait for a hosted stack? Spin up your own and make your own private env repo:

```bash
cd ~/orbis-git-poc
make up && make setup    # local vera + 3 orbis nodes, writes .orbis-ring.env
make e2e                 # optional, runs the full alice/bob/mallory test
```

Then in any repo:

```bash
git orbis init
mkdir -p .orbis && cp ~/orbis-git-poc/.orbis-ring.env .orbis/config
git orbis whoami > .orbis/readers
echo '.env filter=orbis' >> .gitattributes
git add -A && git commit -m "encrypted env"
```

## Things to know

- **No revoke.** Pulling someone off `.orbis/readers` doesn't un-share what they've seen. Rotate the actual secret.
- **History counts.** Old `.env` versions stay readable to readers, same as any file in git.
- **It's a POC.** Commits share a devnet signing key and the secret briefly shows up in `ps`. The [parent repo's README](https://github.com/reecepbcups/orbis-git-poc#known-gaps) has the full list.
