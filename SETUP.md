# Setup — installing these skills in Claude Code

This is a fork of [mattpocock/skills](https://github.com/mattpocock/skills) (MIT),
rebranded so it registers as its own Claude Code marketplace:

- **Marketplace:** `anabaei` → `github.com/anabaei/skills`
- **Plugin:** `anabaei-skills`
- Skills load with the prefix `anabaei-skills:*` (e.g. `anabaei-skills:tdd`).

Follow this on any laptop where you want to *use* the skills.

---

## Step 1 — Authenticate to GitHub as `anabaei`

Nothing can fetch, pull, or push until this machine can auth to the `anabaei`
account. Pick one:

### SSH (recommended)
1. Add an SSH public key to <https://github.com/settings/keys> while logged in as
   **anabaei**. Either:
   - copy an existing key that's already on `anabaei` (`~/.ssh/id_ed25519{,.pub}`)
     to this machine — the **same** key works on any number of machines; or
   - generate a fresh one here and add it:
     ```bash
     ssh-keygen -t ed25519 -C "anabaei" -f ~/.ssh/id_ed25519
     ```
   > A given SSH key can only belong to **one** GitHub account. If GitHub says
   > "key is already in use", it's on another account — generate a new key for
   > this machine instead and add that.
2. Verify:
   ```bash
   ssh -T git@github.com          # expect: "Hi anabaei! You've successfully authenticated..."
   ```

### HTTPS token (alternative)
1. Create a token at <https://github.com/settings/tokens> with the **`repo`** scope.
2. Store it once (macOS):
   ```bash
   git config --global credential.helper osxkeychain
   ```
   Username = `anabaei`, password = the token (prompted on first pull/push).

---

## Step 2 — Register the marketplace and install the plugin

No git clone needed — the marketplace is fetched straight from GitHub.

```bash
claude plugin marketplace add anabaei/skills
claude plugin install anabaei-skills@anabaei

# If Matt Pocock's original is installed, remove it to avoid duplicate skills:
claude plugin uninstall mattpocock-skills
claude plugin marketplace remove mattpocock
```

Then **restart Claude Code** so the skill list refreshes.

---

## Editing skills

Only clone the repo on a machine where you want to *change* the skills:

```bash
git clone git@github.com:anabaei/skills.git
cd skills
git remote add upstream https://github.com/mattpocock/skills.git   # to pull Matt's updates later
```

Make your edits under `skills/`, then:

```bash
git add -A
git commit -m "..."
git push origin main
```

On each laptop, pick up the new version with:

```bash
claude plugin update anabaei-skills        # or: claude plugin marketplace update anabaei
```

Restart Claude Code to apply.

### Pulling in upstream (Matt's) updates

```bash
git fetch upstream
git merge upstream/main
git push origin main
```

> Keep the `LICENSE` file and attribution to Matt Pocock — that's all the MIT
> license requires.

---

## Where Claude Code stores this (per machine)

- `~/.claude/plugins/known_marketplaces.json` — registered marketplaces
- `~/.claude/plugins/installed_plugins.json` — installed plugins + versions
- `~/.claude/plugins/cache/anabaei/…` — the fetched skill files Claude loads

These are per-machine state, which is why Step 1–2 run on every laptop rather than
being synced through the repo.
