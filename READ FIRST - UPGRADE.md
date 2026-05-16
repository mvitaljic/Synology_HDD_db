# Upgrade Runbook: syno_hdd_db.sh

How to safely upgrade the pinned `syno_hdd_db.sh` script on the Synology DS923+
from one verified version to a newer one.

**Principle:** Never run upstream code blindly. Review every change, pin to an
exact commit, verify integrity locally, then run.

---

## Current pinned version

Keep this section updated each time you upgrade.

| Field          | Value                              |
| -------------- | ---------------------------------- |
| Version        | `v3.6.125`                         |
| Commit SHA     | `2629d1e`                          |
| Date pinned    | `2026-05-16`                       |
| DSM at time of | `DSM 7.3.2-86009-3`                |
| NAS model      | `DS923+`                           |

---

## When to upgrade

Upgrade when **any** of these is true:

- A new DSM major/minor version is released and the current script fails after the update
- Upstream has a security fix you've reviewed and want
- You add new drive hardware that the current pinned version doesn't recognize
- You want a new feature documented in upstream release notes

**Do NOT upgrade just because there's a newer version available.** "If it ain't broke" applies here.

---

## Step 1 — Review upstream changes (on a regular computer, NOT the NAS)

1. Open the upstream repo:
   `https://github.com/007revad/Synology_HDD_db`

2. Click **Releases** in the right sidebar. Read the release notes for every
   version newer than the one you have pinned.

3. Open the commit history for `syno_hdd_db.sh`:
   `https://github.com/007revad/Synology_HDD_db/commits/main/syno_hdd_db.sh`

4. Compare your current pinned commit to the latest. Replace `OLD_SHA` and
   `NEW_SHA` in this URL and open it in your browser:
   `https://github.com/007revad/Synology_HDD_db/compare/OLD_SHA...NEW_SHA`

5. **Read every diff.** Look for:
   - New `curl` / `wget` calls (unexpected network requests)
   - Changes to file paths outside `/etc.defaults/` and the drive DB
   - New `sudo` invocations or privilege escalation
   - Obfuscated code (base64, eval, here-docs piping to shell)
   - New external URLs being contacted

   If anything looks suspicious, **stop**. Open an issue upstream or wait for the next release.

6. If the changes look clean, note the new commit SHA. Confirm it matches the
   version you want by looking at the diff — find the line:
   ```
   scriptver="vX.Y.Z"
   ```
   That confirms the commit actually contains the version you think it does.

---

## Step 2 — Sync your fork

1. Go to your fork on GitHub: `https://github.com/mvitaljic/Synology_HDD_db`

2. If GitHub shows "This branch is N commits behind 007revad:main", click
   **Sync fork** → **Update branch**.

3. Verify your fork's `main` now points to the same commit as upstream.

---

## Step 3 — Update the local copy on the NAS

SSH into the NAS as your admin user.

```bash
# Variables — UPDATE THESE for the new version
GITHUB_USER="mvitaljic"
NEW_COMMIT_SHA="abc1234"          # full or short SHA from Step 1
NEW_VERSION="v3.6.XXX"            # for documentation only

# Go to the script directory
cd /volume1/homes/marin/syno-scripts

# Back up the current working version before replacing it
cp syno_hdd_db.sh "syno_hdd_db.sh.bak-$(date +%Y%m%d-%H%M%S)"
cp syno_hdd_db.sh.sha256 "syno_hdd_db.sh.sha256.bak-$(date +%Y%m%d-%H%M%S)"

# Verify the existing script still matches its recorded hash
# (sanity check — make sure nothing on the NAS tampered with it)
sha256sum -c syno_hdd_db.sh.sha256
# Expected: syno_hdd_db.sh: OK
# If this FAILS, stop and investigate before continuing.

# Download new version from your fork at the new pinned commit
curl -fsSL "https://raw.githubusercontent.com/${GITHUB_USER}/Synology_HDD_db/${NEW_COMMIT_SHA}/syno_hdd_db.sh" -o syno_hdd_db.sh.new

# Confirm the downloaded file contains the version string you expect
grep '^scriptver=' syno_hdd_db.sh.new
# Expected: scriptver="v3.6.XXX"  (matching NEW_VERSION)

# If version doesn't match, abort:
#   rm syno_hdd_db.sh.new
# and investigate the SHA.

# Replace the live file
mv syno_hdd_db.sh.new syno_hdd_db.sh
chmod 700 syno_hdd_db.sh

# Record new integrity hash
sha256sum syno_hdd_db.sh > syno_hdd_db.sh.sha256
echo "Pinned to commit: ${NEW_COMMIT_SHA} (${NEW_VERSION})" >> syno_hdd_db.sh.sha256
echo "Upgraded from previous version on: $(date)" >> syno_hdd_db.sh.sha256

# Final review — read the script before executing
less syno_hdd_db.sh
```

---

## Step 4 — Dry run (if supported)

The script supports `--help` and may support `--dry-run` or similar. Always read help first on a new version:

```bash
sudo ./syno_hdd_db.sh --help
```

Look for new flags, removed flags, or changed defaults compared to the previous version.

---

## Step 5 — Run the script

```bash
sudo ./syno_hdd_db.sh
```

**Save the full output.** Copy it into a text file or paste it into your notes.
You'll want this if anything misbehaves later.

Expected indicators of success:
- `Edited unverified drives in ds923+_host_v7.db` or `already exists`
- `Support disk compatibility already enabled` (or `enabled`)
- `M.2 volume support already enabled` (or `enabled`)
- `DSM successfully checked disk compatibility`

If you see errors, **do not reboot**. Restore the previous version (Step 7) and report the issue upstream.

---

## Step 6 — Reboot and verify

```bash
sudo reboot
```

After the NAS is back online (~3 minutes), SSH in again and verify:

```bash
cd /volume1/homes/marin/syno-scripts

# Re-run the script — it should report "already exists" for everything
sudo ./syno_hdd_db.sh
```

Then in DSM web UI:
1. Open **Storage Manager** → check the M.2 storage pool shows **Healthy**
2. Check drives are not flagged as unsupported
3. SSH back in and confirm Docker stack is running: `sudo docker ps`

---

## Step 7 — Rollback (if something breaks)

If the new version causes problems:

```bash
cd /volume1/homes/marin/syno-scripts

# List backups
ls -lt syno_hdd_db.sh.bak-*

# Pick the most recent backup (replace TIMESTAMP)
TIMESTAMP="20260516-143000"

# Restore script
cp "syno_hdd_db.sh.bak-${TIMESTAMP}" syno_hdd_db.sh
cp "syno_hdd_db.sh.sha256.bak-${TIMESTAMP}" syno_hdd_db.sh.sha256

# Verify restore is intact
sha256sum -c syno_hdd_db.sh.sha256
# Expected: syno_hdd_db.sh: OK

chmod 700 syno_hdd_db.sh

# Run the old version
sudo ./syno_hdd_db.sh
sudo reboot
```

The script has a `--restore` flag that reverts DSM changes entirely. If even the rollback version misbehaves:

```bash
sudo ./syno_hdd_db.sh --restore
```

Then contact upstream or fall back to Synology-branded drives.

---

## Step 8 — Update this document

After a successful upgrade, edit the **Current pinned version** table at the top of this file and commit to your fork:

```bash
# On your dev machine, in your fork's local clone
git pull
# edit READ FIRST - UPGRADE.md
git add READ FIRST - UPGRADE.md
git commit -m "Bump pinned version to vX.Y.Z (commit SHA)"
git push
```

This keeps your runbook synchronized with reality.

---

## Quick reference: commands without explanation

For when you've done this before and just want the commands:

```bash
GITHUB_USER="mvitaljic"
NEW_COMMIT_SHA="abc1234"
NEW_VERSION="v3.6.XXX"

cd /volume1/homes/marin/syno-scripts
cp syno_hdd_db.sh "syno_hdd_db.sh.bak-$(date +%Y%m%d-%H%M%S)"
cp syno_hdd_db.sh.sha256 "syno_hdd_db.sh.sha256.bak-$(date +%Y%m%d-%H%M%S)"
sha256sum -c syno_hdd_db.sh.sha256
curl -fsSL "https://raw.githubusercontent.com/${GITHUB_USER}/Synology_HDD_db/${NEW_COMMIT_SHA}/syno_hdd_db.sh" -o syno_hdd_db.sh.new
grep '^scriptver=' syno_hdd_db.sh.new
mv syno_hdd_db.sh.new syno_hdd_db.sh
chmod 700 syno_hdd_db.sh
sha256sum syno_hdd_db.sh > syno_hdd_db.sh.sha256
echo "Pinned to commit: ${NEW_COMMIT_SHA} (${NEW_VERSION})" >> syno_hdd_db.sh.sha256
echo "Upgraded on: $(date)" >> syno_hdd_db.sh.sha256
less syno_hdd_db.sh
sudo ./syno_hdd_db.sh
sudo reboot
```

---

## Security checklist (every upgrade)

- [ ] Read all commit diffs between old and new SHA
- [ ] Confirmed version string in downloaded file matches expected version
- [ ] Backed up previous script and its hash file
- [ ] Verified previous hash before replacing
- [ ] Recorded new hash and commit SHA
- [ ] Reviewed script with `less` before executing
- [ ] Saved script output after running
- [ ] Verified Storage Pool healthy in DSM after reboot
- [ ] Updated this UPGRADE.md with new pinned version
