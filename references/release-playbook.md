# Release playbook (0.2 experimental prerelease)

Only run this after the full verification matrix passes and the user explicitly asks for publication. Publication is authorized for an experimental prerelease only; it does not change production acceptance criteria.

## 1. Rebuild the asset from the verified working tree

Module root: `D:\Laragon\www\FOSSBill\modules\Whmcstofossbilling\`. Output: `.local/WHMCStoFossbilling-0.2.zip`.

Exclude `.local/`, `.git/`, `dist/`, `*.zip`, `*.log`. Every entry must be rooted at `Whmcstofossbilling/`. The manifest must still read id `whmcstofossbilling`, name `WHMCStoFossbilling`, version `0.2`, author `Monlam Makhampa`, description containing `vibe coded`.

A reliable PowerShell approach is `robocopy <module> <staging>\Whmcstofossbilling /E /XD .local .git dist /XF *.zip *.log` (exit codes 0–7 are success), then write entries with `System.IO.Compression.ZipFile` using forward-slash relative paths. Create explicit directory entries for the root and every subdirectory in addition to the 84 file entries; file-only archives have only 84 entries and are not the verified layout. Verify entry count, zero excluded-path leaks, and SHA-256.

Current verified asset: 103 entries, 84 files, 415373 bytes, SHA-256 `7A776BB47A31015C81D8FE82DAE0ED77FC11A91096449FE25C98E7EA984B933D`. It carries the full AGPL-3.0 `LICENSE` text and `manifest.json` declares `AGPL-3.0`. The earlier assets (`C952A86A6CFC7C5C60F4E24D03C111CBCF94BADD5EDC1DE4E7B5317FBDF63A3B`, `BC14A81C9F2C557890467798EB0658ADF361406A22B71FF30F6B82E941C35869`, `34205A8A45EDE3EA93D72A2AA0B4DB4FFBE243CAE2C6044C70E1F5A9336FD731`, `A5EC0440E51B0D7A81524DC66C2474234E28513F8AA15B8FF8902E47B7E75C01`) are stale.

## 2. Commit and push

Stage only module files (`.gitignore` already excludes `.local/`, `dist/`, `*.zip`, `*.log`). Confirm the staged list before committing. Then:

```powershell
git -C <module> push origin main
git -C <module> tag -f 0.2
git -C <module> push -f origin 0.2
```

Force retag is required because `0.2` already exists. Branch is `main`, remote `origin` = `https://github.com/thai916/WHMCStoFossbilling.git`. Confirm `git status --short --branch` is clean and the remote tag points at the new commit.

## 3. Replace the GitHub release asset and notes

```powershell
gh release upload 0.2 <module>\.local\WHMCStoFossbilling-0.2.zip --clobber --repo thai916/WHMCStoFossbilling
gh release edit 0.2 --notes-file <notes-file> --repo thai916/WHMCStoFossbilling
```

Build the notes from `docs/RELEASE-0.2.md`, converting relative doc links to absolute `https://github.com/thai916/WHMCStoFossbilling/blob/0.2/docs/...` URLs.

**Encoding trap:** Windows PowerShell 5.1 `Get-Content -Raw` without `-Encoding UTF8` reads a UTF-8 file as ANSI and produces mojibake (`â€”`, `â†’`) in the published notes. Always read with `-Encoding UTF8` and write with `[System.IO.File]::WriteAllText($path, $doc, (New-Object System.Text.UTF8Encoding($false)))`. Verify the remote body contains the new section headings and no `â€` sequence.

Do not use a complex `--jq` expression with `gh release view` in PowerShell; simpler `gh api repos/.../releases/tags/0.2 --jq '.body'` avoids argument-splitting problems.

## 4. Verify publication

- `gh api repos/thai916/WHMCStoFossbilling/releases/tags/0.2 --jq '.assets[] | {name, size, digest}'` must show `WHMCStoFossbilling-0.2.zip` with digest `sha256:<local SHA-256>`.
- Release stays a **prerelease**, non-draft, with the incomplete-scope warning and `vibe coded` disclosure.
- Release URL: https://github.com/thai916/WHMCStoFossbilling/releases/tag/0.2

## 5. Never bundle

`.local/` (including `core-baseline.json`, browser artifacts, backups, Playwright install/cache), disposable test databases, generated reports, runtime logs, and any credentials. Do not commit `config.php` or `config.old.php` changes.

## Git state history

Before the final batching/visibility/recovery work: branch `main` and tag `0.2` pointed at `038494b` (`Avoid API rate limits during run processing`). After publication: commit `ed9de4d` (`Add adaptive batching, visibility repair, and session recovery`) with 21 module files changed, and tag `0.2` force-moved to it. The overwritten digest `C952…63A3B` corresponded to commit `29a7df6`; the current digest `7A77…B933D` corresponds to commit `3782099` (`Fix nullable hosting references and invoice currency disclosure`). Force-moving the same `0.2` tag is intentional because the version and asset filename stay unchanged.
