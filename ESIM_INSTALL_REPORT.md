# eSim 2.5 Installation on Ubuntu 25.04: Issue Report

## Environment
- Host: Windows 11 laptop (Dell), Oracle VirtualBox
- Guest: Ubuntu 25.04 Desktop, 4 GB RAM, 2 CPUs, 30 GB disk
- Repo: fork of FOSSEE/eSim, branch `installers`
- Script: `Ubuntu/install-eSim.sh` (wrapper) and `Ubuntu/install-eSim-scripts/install-eSim-25.04.sh`
- Command: `./install-eSim.sh --install 2>&1 | tee install.log`

## Summary
| # | Stage | Issue | Status |
|---|-------|-------|--------|
| 1 | Setup | Ubuntu 25.04 ISO not on main download page (end of life) | Worked around |
| 2 | Git | `installer` branch not found; it is `installers` and exists only on upstream | Fixed |
| 3 | Git | GitHub rejects account password on push | Fixed |
| 4 | Installer | Syntax error at line 70 (missing `fi`) | Fixed |
| 5 | Installer | `library/kicadLibrary.tar.xz` missing, install aborts after KiCad | Root cause identified, fix not completed |
| 6 | Installer | Stray config `echo` lines write to `/` | Fixed |
| 7 | Installer | `Detected Ubuntu Version:` prints blank | Fixed |
| 8 | Installer | Dead duplicate functions in the wrapper script | Reported, not changed |

## Issue details

### 1: Ubuntu 25.04 ISO unavailable
- Cause: 25.04 is past end of life, so it was moved to old-releases.ubuntu.com.
- Fix: downloaded `ubuntu-25.04-desktop-amd64.iso` from https://old-releases.ubuntu.com/releases/plucky/

### 2: Branch not found
- Error: `pathspec 'installer' did not match any file(s) known to git`
- Cause: the fork only has master, and the branch is named `installers`.
- Fix:
  git remote add upstream https://github.com/FOSSEE/eSim.git
  git fetch upstream
  git checkout -b installers upstream/installers
  git push -u origin installers

### 3: GitHub authentication
- Error: password rejected on `git push`
- Cause: GitHub no longer accepts account passwords for Git operations.
- Fix: created a personal access token (repo scope) and used it as the password.

### 4: Syntax error in install-eSim.sh (FIXED)
- Error: `install-eSim.sh: line 70: syntax error near unexpected token '}'`
- Found with: `bash -n install-eSim.sh`
- Cause: in `run_version_script()`, `if [[ -f "$SCRIPT" ]]; then` was never closed. `git diff upstream/installers` was empty, so the bug is in the original script.
- Fix: added the missing `fi` before the closing `}`.
- Result: `bash -n` printed nothing, and the installer then started, picked `install-eSim-25.04.sh`, and installed KiCad 8.0.8.

### 5: kicadLibrary.tar.xz not found (NOT FIXED, root cause found)
- Error: `tar (child): library/kicadLibrary.tar.xz: Cannot open: No such file or directory`, then `Aborting Installation...`
- Stage: immediately after "KiCad installation completed successfully!"
- Cause: `install-eSim-25.04.sh` line 232 runs `tar -xJf library/kicadLibrary.tar.xz`, but `git ls-tree -r --full-tree upstream/master` shows no such file. The library is stored as an extracted folder `library/kicadLibrary/`. Only `sky130_fd_pr.tar.xz` and the NGHDL source are tarballs.
- Suggested fix: remove the `tar -xJf` step (and the `rm -rf kicadLibrary` cleanup) and copy directly from the existing `library/kicadLibrary/` folder. Not applied or tested because I ran out of time.
- Impact: blocks the main installation.

### 6: Stray config lines write to / (FIXED)
- Error: `install-eSim.sh: line 62 ... 68: /: Is a directory`
- Cause: the wrapper wrote the `[eSim]` config lines using `$config_dir/$config_file`, which are never set there, so the path became `/`. The version script already writes these lines.
- Fix: removed these 7 lines from the wrapper.

### 7: Blank Ubuntu version (FIXED)
- Symptom: `Detected Ubuntu Version:` printed nothing.
- Cause: `lsb_release -d` prints `Ubuntu 25.04`, but the regex `\d+\.\d+\.\d+` needs three numbers.
- Fix: changed to `\d+\.\d+(\.\d+)?`. The 22.04 comparison with "22.04.4" still works.

### 8: Dead code in the wrapper (reported)
- Functions after `run_version_script()` (`installNghdl`, `installKicad`, `copyKicadLibrary`, etc.) duplicate the version script and are never called. They also reference `library/kicadLibrary.tar.xz`. Left unchanged.

## Changes made to Ubuntu/install-eSim.sh
1. Added the missing `fi` and an `else` branch that prints "Script not found" and exits (Issue 4).
2. Deleted the 7 stray `echo` lines (Issue 6).
3. Fixed the version regex (Issue 7).

## Testing notes
- The `fi` fix was tested locally: `bash -n` was clean and the installer ran past the old syntax error.
- The line removal and regex change were made afterwards in the GitHub editor and have not been re-run end to end.
- The full installation did not complete because of Issue 5.

## Evidence

### Issue 4: syntax error at line 70 and the fix
![Issue 4 fix](issue%204%20fix.jpg)
The syntax error at line 70, and `git diff` showing the one-line fix (`+fi`).

### Issues 5 and 6: kicadLibrary tarball and stray lines
![Issues 5 and 6](issue%205,6.jpg)
KiCad installs successfully, then `tar` cannot open `library/kicadLibrary.tar.xz` and the installer aborts. The `/: Is a directory` lines are Issue 6.

### Issue 5: proof the tarball does not exist
![Issue 5 proof](issue%205%20proof.jpg)
`git ls-tree` on upstream/master lists only `sky130_fd_pr.tar.xz` and the NGHDL tarball, so `kicadLibrary.tar.xz` does not exist.
