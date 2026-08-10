# New PC — Windows Development Environment Setup

This guide builds a clean Windows development environment that feels familiar to a macOS developer: Windows Terminal + PowerShell 7, Git/GitHub SSH, `uv` + Python, modern Unix-like CLI tools, Oh My Posh, and a version-controlled PowerShell profile.

## Who this guide is for

Two audiences, one document. Everything is written so the first group can follow it alone and the second group can support them.

- **A. Tech-savvy non-engineers** who are comfortable using Claude Code, Codex, Claude Cowork, ChatGPT Work, terminals, GitHub, and developer tools on macOS, but are new to Windows development.
- **B. Engineers** who already work across Windows/macOS and may need to help teammates in Group A inside a corporate Windows/security environment.

## Table of contents

- [Before you start: corporate Windows is not a personal Mac](#before-you-start-corporate-windows-is-not-a-personal-mac)
- [Phase 1 — Preflight](#phase-1--preflight)
  - [1. Confirm Windows and available disk space](#1-confirm-windows-and-available-disk-space)
  - [2. Confirm winget](#2-confirm-winget)
- [Phase 2 — Git and GitHub](#phase-2--git-and-github)
  - [3. Install Git](#3-install-git)
  - [4. Generate a GitHub SSH key](#4-generate-a-github-ssh-key)
  - [5. Enable the Windows SSH agent](#5-enable-the-windows-ssh-agent)
  - [6. Add the public key to GitHub](#6-add-the-public-key-to-github)
  - [7. Test GitHub SSH](#7-test-github-ssh)
  - [8. Configure Git identity](#8-configure-git-identity)
- [Phase 3 — Repository layout](#phase-3--repository-layout)
  - [9. Create the standard code hierarchy](#9-create-the-standard-code-hierarchy)
- [Phase 4 — Modern terminal and shell](#phase-4--modern-terminal-and-shell)
  - [10. Install PowerShell 7](#10-install-powershell-7)
  - [11. Install or update Windows Terminal](#11-install-or-update-windows-terminal)
- [Phase 5 — Python with uv](#phase-5--python-with-uv)
  - [12. Install uv](#12-install-uv)
  - [13. Install Python](#13-install-python)
  - [14. Install IPython as a user-level CLI tool](#14-install-ipython-as-a-user-level-cli-tool)
- [Phase 6 — Python virtual environments](#phase-6--python-virtual-environments)
  - [15. Use one virtual environment per repository](#15-use-one-virtual-environment-per-repository)
  - [16. Install project dependencies](#16-install-project-dependencies)
- [Phase 7 — Modern CLI tools](#phase-7--modern-cli-tools)
  - [17. Install the command-line toolkit](#17-install-the-command-line-toolkit)
- [Phase 8 — Prompt and font](#phase-8--prompt-and-font)
  - [18. Install Oh My Posh](#18-install-oh-my-posh)
  - [19. Install a Nerd Font](#19-install-a-nerd-font)
- [Phase 9 — Version-controlled PowerShell profile](#phase-9--version-controlled-powershell-profile)
  - [20. Create the version-controlled profile](#20-create-the-version-controlled-profile)
  - [21A. Recommended: bootstrap $PROFILE](#21a-recommended-bootstrap-profile)
  - [21B. Optional personal-machine alternative: symbolic link](#21b-optional-personal-machine-alternative-symbolic-link)
- [Phase 10 — Verification](#phase-10--verification)
  - [22. Restart Windows Terminal and run the smoke test](#22-restart-windows-terminal-and-run-the-smoke-test)
- [Troubleshooting quick reference](#troubleshooting-quick-reference)
- [Notes for engineers supporting corporate teammates](#notes-for-engineers-supporting-corporate-teammates)
- [Resulting baseline](#resulting-baseline)
- [Official references](#official-references)

---

## Before you start: corporate Windows is not a personal Mac

This section sets expectations before anything gets installed. Corporate machines are managed differently from personal ones, and knowing that up front saves hours of fighting policies that were never going to yield.

On a personal Mac, it is common to install Homebrew, run arbitrary shell installers, change shell settings, and add developer tools without thinking much about device-management policy. On a corporate Windows PC, some or all of those actions may be intentionally restricted.

**Do not work around corporate security controls.** In particular, do not disable antivirus/EDR, tamper protection, application control, TLS inspection, firewall rules, Group Policy, or organization-managed execution policies just to get a tool installed.

### When a step is blocked

Follow this short escalation path instead of forcing the install.

1. Record the exact command and error message.
2. Check whether the company has an approved software portal, package source, or documented developer workstation setup.
3. Ask the IT/security/dev-infrastructure team for the approved path.
4. Continue with the rest of the guide only where policy allows.

### Common corporate restrictions

These are the policies you are most likely to run into. Recognizing them helps you describe the problem accurately when you ask IT for help.

- no local Administrator rights;
- `winget` disabled or restricted to an internal source;
- Microsoft Store disabled;
- Developer Mode disabled;
- PowerShell execution policy enforced by Group Policy;
- AppLocker/WDAC blocking unsigned or unapproved executables;
- GitHub SSH on TCP/22 blocked;
- outbound HTTPS routed through a proxy or TLS-inspection appliance;
- GitHub, PyPI, npm, model registries, or binary release sites allow-listed selectively;
- company certificates required for package managers to trust intercepted TLS.

This guide gives a normal path and, where useful, a corporate-safe fallback. **A blocked security policy is an IT/infrastructure issue, not a prompt-engineering challenge.**

---

## Phase 1 — Preflight

Before installing anything, confirm the machine has what the rest of the guide depends on: a current Windows build, enough free disk space, and a working package manager.

### 1. Confirm Windows and available disk space

Check which Windows build you are on and make sure there is enough room for developer tooling before any of it gets installed.

Open **PowerShell** from the Start menu. It may initially be the built-in Windows PowerShell 5.1; that is fine for the first few commands.

Check Windows:

```powershell
winver
```

Check disk space:

```powershell
Get-Volume C | Select-Object DriveLetter, @{Name="FreeGB";Expression={[math]::Round($_.SizeRemaining/1GB,1)}}
```

For a normal development workstation, have **at least ~20 GB free** before starting. **50+ GB is more comfortable** if the machine will use Python scientific packages, Docling, PyTorch/model weights, Docker, build caches, or large repositories.

### 2. Confirm `winget`

Verify the package manager works, because nearly every install in this guide goes through it. `winget` is Windows Package Manager; for this setup it plays roughly the role Homebrew plays on macOS.

```powershell
winget --version
```

If that prints a version number, continue.

If `winget` is missing on a personal machine, install/update **App Installer** through the Microsoft-supported Windows path.

If `winget` is missing or blocked on a corporate machine, **stop trying to bootstrap alternate package managers** unless your organization explicitly supports them. Ask IT which package source or software portal to use.

Useful discovery command:

```powershell
winget search <name>
```

When this guide gives a package ID, prefer exact installs:

```powershell
winget install --id <PACKAGE_ID> --exact
```

---

## Phase 2 — Git and GitHub

Install Git and set up SSH so this machine authenticates to GitHub the way a macOS setup would: with keys, not passwords.

### 3. Install Git

Install Git itself and confirm the shell can see it.

```powershell
winget install --id Git.Git --exact
```

Close and reopen the shell if `git` is not immediately visible, then verify:

```powershell
git --version
```

Git for Windows also includes **Git Bash**. The default shell for this guide will be PowerShell 7, but Git Bash is useful when you explicitly need GNU/Bash behavior.

### 4. Generate a GitHub SSH key

Create the SSH keypair that will identify this machine to GitHub.

Check OpenSSH:

```powershell
ssh -V
```

Generate an Ed25519 key:

```powershell
ssh-keygen -t ed25519 -C "YOUR_GITHUB_EMAIL"
```

When prompted for the file location, pressing **Enter** accepts the normal location:

```text
C:\Users\<username>\.ssh\id_ed25519
```

Use a passphrase unless your organization's credential policy specifies another approach.

Verify:

```powershell
Get-ChildItem $HOME\.ssh
```

You should see:

```text
id_ed25519
id_ed25519.pub
```

**Never share or upload `id_ed25519`.** That is the private key. Only the `.pub` file is meant to be shared.

### 5. Enable the Windows SSH agent

Turn on the built-in Windows service that holds your key in memory, so you are not retyping the passphrase on every Git operation.

Check it:

```powershell
Get-Service ssh-agent
```

If the service is stopped, the normal setup is:

```powershell
Set-Service -Name ssh-agent -StartupType Automatic
Start-Service ssh-agent
```

This may require an Administrator PowerShell session.

#### If corporate policy denies this

Some organizations lock down Windows services; the answer is to ask which authentication path is supported, not to force this one. Do not weaken service policy yourself. Ask the Windows/IT team whether the approved setup is:

- Windows `ssh-agent`;
- a managed SSH agent;
- Git Credential Manager over HTTPS;
- hardware-backed authentication;
- another company-approved Git authentication path.

Once the agent is available, load the key:

```powershell
ssh-add $HOME\.ssh\id_ed25519
```

Verify:

```powershell
ssh-add -l
```

### 6. Add the public key to GitHub

Register the public half of the key with your GitHub account so GitHub accepts connections from this machine.

Copy it to the Windows clipboard:

```powershell
Get-Content $HOME\.ssh\id_ed25519.pub | Set-Clipboard
```

In GitHub:

**Settings → SSH and GPG keys → New SSH key**

Add it as an authentication key.

If your organization uses GitHub Enterprise, SSO authorization, SSH certificates, or centrally managed credentials, follow the organization-specific flow instead.

### 7. Test GitHub SSH

Confirm the whole chain — key, agent, GitHub — works before trying to clone anything.

```powershell
ssh -T git@github.com
```

On the first connection, SSH may ask whether to trust GitHub's host key. Verify the fingerprint against GitHub's published host-key fingerprints before accepting it.

A successful result is similar to:

```text
Hi YOUR_USERNAME! You've successfully authenticated, but GitHub does not provide shell access.
```

#### Corporate network: SSH port 22 blocked

Some corporate networks block outbound SSH on TCP/22; GitHub supports SSH over the HTTPS port instead. Before changing anything, confirm that using this route is allowed by company policy.

A typical `~/.ssh/config` entry is:

```text
Host github.com
    Hostname ssh.github.com
    Port 443
    User git
```

Test again:

```powershell
ssh -T git@github.com
```

A proxy can still interfere with this. If it fails, involve IT/network engineering rather than bypassing the proxy.

### 8. Configure Git identity

Tell Git who you are so commits are attributed correctly.

```powershell
git config --global user.name "YOUR NAME"
git config --global user.email "YOUR_GITHUB_EMAIL"
```

Verify:

```powershell
git config --global --list
```

Use an email associated with GitHub, or a GitHub `noreply` address if that is your preference.

---

## Phase 3 — Repository layout

Set up one predictable place where every repository lives, mirroring the layout you already use on macOS.

### 9. Create the standard code hierarchy

Create the `~/code/github/...` directory tree and clone your first repository into it.

Use the same logical structure on Windows and macOS:

```text
~/code/github/{GITHUB_USER_OR_ORG}/{REPO}
```

Examples:

```text
~/code/github/austinogilvie/jodie
~/code/github/good-tradecraft/good-return
```

Create the root:

```powershell
New-Item -ItemType Directory -Force $HOME\code\github
```

Create an owner/org directory:

```powershell
New-Item -ItemType Directory -Force $HOME\code\github\YOUR_GITHUB_USERNAME
```

Clone with SSH:

```powershell
cd $HOME\code\github\YOUR_GITHUB_USERNAME
git clone git@github.com:YOUR_GITHUB_USERNAME/YOUR_REPO.git
```

PowerShell understands `~` and `$HOME`, so paths can remain conceptually similar to macOS.

To open the current directory in Windows File Explorer:

```powershell
explorer .
```

That is the Windows equivalent of:

```bash
open .
```

on macOS.

---

## Phase 4 — Modern terminal and shell

Replace the legacy Windows PowerShell 5.1 experience with PowerShell 7 running inside Windows Terminal — the closest Windows equivalent of iTerm2 + zsh.

### 10. Install PowerShell 7

Install the modern, cross-platform shell that the rest of this guide assumes.

Windows PowerShell 5.1 and PowerShell 7 are separate products. Installing PowerShell 7 does **not** remove Windows PowerShell 5.1.

Install:

```powershell
winget install --id Microsoft.PowerShell --exact
```

Verify:

```powershell
pwsh -Version
```

From this point forward, the intended development shell is **PowerShell 7 (`pwsh`)**, not Windows PowerShell 5.1.

### 11. Install or update Windows Terminal

Install the terminal application and make PowerShell 7 its default shell.

Windows Terminal is the **terminal application**; PowerShell 7 is the **shell running inside it**.

macOS analogy:

```text
iTerm2 / Terminal.app  ≈  Windows Terminal
zsh                    ≈  PowerShell 7
```

Install or update:

```powershell
winget install --id Microsoft.WindowsTerminal --exact
```

If already installed, `winget` may upgrade it.

Launch **Terminal** from the Start menu.

Open:

**Terminal → Settings → Startup**

Set:

```text
Default profile: PowerShell
```

Make sure the opened shell reports:

```powershell
$PSVersionTable.PSVersion
```

with major version `7`.

Do not confuse:

- **Windows Terminal** — terminal emulator/application;
- **PowerShell 7** — modern shell;
- **Windows PowerShell 5.1** — legacy Windows shell;
- **Command Prompt** — `cmd.exe`.

---

## Phase 5 — Python with `uv`

Install `uv` and let it manage everything Python-related: interpreter versions, virtual environments, project dependencies, and standalone CLI tools.

### 12. Install `uv`

Install the `uv` binary itself; everything Python-related in this guide flows through it.

Install with Windows Package Manager:

```powershell
winget install --id astral-sh.uv --exact
```

Restart Terminal after installation if necessary.

Verify:

```powershell
uv --version
```

`uv` will manage Python versions, virtual environments, project dependencies, and globally exposed Python CLI tools.

Do not install project dependencies into a shared global Python environment.

### 13. Install Python

Use `uv` to install a real Python interpreter and make sure `python` on the command line resolves to it, not to the Microsoft Store stub.

For a compatibility-oriented baseline:

```powershell
uv python install 3.12 --default
```

See what `uv` knows about:

```powershell
uv python list
```

If:

```powershell
python --version
```

still opens the Microsoft Store alias or says Python cannot be found, run:

```powershell
uv python update-shell
```

Then **close Terminal completely and reopen it**.

Verify:

```powershell
python --version
```

A project is free to require a different Python version. Follow the repository's `pyproject.toml`, `.python-version`, lock file, or contributor documentation when present.

### 14. Install IPython as a user-level CLI tool

Install IPython once, globally, so a good interactive Python shell is always available regardless of which project you are in.

```powershell
uv tool install ipython
```

Verify:

```powershell
ipython --version
```

`uv tool install` is appropriate for command-line applications you want available across projects. Project libraries should stay project-scoped.

---

## Phase 6 — Python virtual environments

Keep every project's dependencies isolated in its own environment, exactly as you would on macOS.

### 15. Use one virtual environment per repository

Create a `.venv` inside each repository rather than sharing one environment across projects.

Do **not** put a general `.venv` in your home directory.

Inside a repository:

```powershell
cd $HOME\code\github\YOUR_ORG\YOUR_REPO
uv venv
```

PowerShell activation:

```powershell
. .\.venv\Scripts\Activate.ps1
```

#### If PowerShell says running scripts is disabled

Activation is itself a PowerShell script, so it is subject to the machine's execution policy.

Inspect policy first:

```powershell
Get-ExecutionPolicy -List
```

On a personal machine, a common user-scoped setting is:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

Then retry:

```powershell
. .\.venv\Scripts\Activate.ps1
```

#### Corporate machine warning

On managed machines the execution policy may not be yours to change. If `MachinePolicy` or `UserPolicy` is set by Group Policy, **do not bypass it** with `Bypass`, unsigned-script tricks, or machine-wide changes. Ask IT/dev-infrastructure for the supported Python environment workflow.

Also remember that many `uv` commands do **not require manual activation**. For example, depending on the repository:

```powershell
uv run python script.py
uv run pytest
uv sync
```

may be preferable.

### 16. Install project dependencies

Populate the environment from the project's lock file when one exists, or ad hoc when it does not.

If the repository already has a `pyproject.toml` / `uv.lock`, prefer the repository-defined workflow:

```powershell
uv sync
```

For an ad-hoc environment without a project dependency file:

```powershell
uv pip install pandas numpy scikit-learn requests docopt beautifulsoup4 docling
```

Common baseline packages for data/document work:

```text
pandas
numpy
scikit-learn
requests
docopt
beautifulsoup4
docling
```

Do not install these globally merely because they are commonly used.

---

## Phase 7 — Modern CLI tools

Install the modern Unix-style search and navigation tools — `rg`, `fd`, `fzf`, and friends — that make a command line pleasant to live in.

### 17. Install the command-line toolkit

Install the whole toolkit in one pass, then verify each command answers.

Install:

```powershell
winget install --id BurntSushi.ripgrep.MSVC --exact
winget install --id sharkdp.fd --exact
winget install --id jqlang.jq --exact
winget install --id junegunn.fzf --exact
winget install --id sharkdp.bat --exact
winget install --id eza-community.eza --exact
winget install --id ajeetdsouza.zoxide --exact
```

Restart Terminal if newly installed commands are not found.

Verify:

```powershell
rg --version
fd --version
jq --version
fzf --version
bat --version
eza --version
zoxide --version
```

Conceptually:

| Tool | Role |
|---|---|
| `rg` | fast recursive text search; modern `grep` |
| `fd` | friendly filesystem search; modern `find` |
| `jq` | JSON querying/transformation |
| `fzf` | interactive fuzzy finder |
| `bat` | enhanced file viewer |
| `eza` | enhanced directory listing |
| `zoxide` | directory jumping based on usage |

PowerShell is **not Bash**. Even where familiar names such as `ls`, `cat`, or `mv` exist, they may be PowerShell aliases backed by PowerShell cmdlets rather than GNU/BSD programs.

Examples:

```text
mv   -> Move-Item
cp   -> Copy-Item
rm   -> Remove-Item
ls   -> Get-ChildItem (until customized)
cat  -> Get-Content
```

Use PowerShell semantics when scripting PowerShell.

---

## Phase 8 — Prompt and font

Make the shell prompt informative and good-looking with Oh My Posh and a Nerd Font.

### 18. Install Oh My Posh

Install the prompt engine; the profile in Phase 9 will activate it at shell startup.

```powershell
winget install --id JanDeDobbeleer.OhMyPosh --exact
```

Restart Terminal if necessary.

Verify:

```powershell
oh-my-posh version
```

### 19. Install a Nerd Font

Install a patched font so the prompt's glyphs and icons render correctly, then select it in Windows Terminal.

```powershell
oh-my-posh font install meslo
```

Then in Windows Terminal:

**Settings → Profiles → Defaults → Appearance → Font face**

Choose an installed Meslo Nerd Font Mono variant, for example:

```text
MesloLGL Nerd Font Mono
```

Exact family names can vary with the current Meslo package.

If corporate policy prevents user-installed fonts, skip this customization and use a prompt/theme that does not depend on Nerd Font glyphs.

---

## Phase 9 — Version-controlled PowerShell profile

Store your shell configuration in your dotfiles repository so it is versioned, portable, and easy to restore on the next machine.

There are two reasonable ways to keep PowerShell configuration in a dotfiles repository:

1. **Bootstrap file:** Windows' `$PROFILE` dot-sources a differently named file in the repo. This is the most corporate-friendly option because it does not require symlink privileges.
2. **Symbolic link:** `$PROFILE` itself points into the dotfiles repo. This is elegant on a personal/dev-mode machine but may be blocked by corporate policy.

**Do not mix the two approaches with the same target filename.** A symlinked `$PROFILE` that is replaced with a bootstrap line pointing to itself creates an infinite profile-loading loop.

### 20. Create the version-controlled profile

Write the actual profile — prompt setup, aliases, and helper functions — as a file inside the dotfiles repository.

Recommended repository file:

```text
~/code/github/YOUR_GITHUB_USERNAME/dotfiles/powershell/profile.ps1
```

Use:

```powershell
New-Item -ItemType Directory -Force $HOME\code\github\YOUR_GITHUB_USERNAME\dotfiles\powershell
notepad $HOME\code\github\YOUR_GITHUB_USERNAME\dotfiles\powershell\profile.ps1
```

Suggested contents:

```powershell
# PowerShell development profile

# ---------------------------------------------------------------------------
# Prompt
# ---------------------------------------------------------------------------

if (Get-Command oh-my-posh -ErrorAction SilentlyContinue) {
    oh-my-posh init pwsh --config "jandedobbeleer" | Invoke-Expression
}

# ---------------------------------------------------------------------------
# Directory navigation
# ---------------------------------------------------------------------------

# Keep zoxide initialization after the prompt initialization.
if (Get-Command zoxide -ErrorAction SilentlyContinue) {
    zoxide init powershell | Out-String | Invoke-Expression
}

# ---------------------------------------------------------------------------
# Modern Unix-style command-line tools
# ---------------------------------------------------------------------------

if (Get-Command eza -ErrorAction SilentlyContinue) {
    Remove-Item Alias:ls -ErrorAction SilentlyContinue

    function ls {
        eza --icons @args
    }

    function ll {
        eza --long --icons --git @args
    }

    function la {
        eza --long --all --icons --git @args
    }

    function lt {
        eza --tree --level=2 --icons @args
    }
}

if (Get-Command rg -ErrorAction SilentlyContinue) {
    Set-Alias -Name grep -Value rg
}

# ---------------------------------------------------------------------------
# Environment
# ---------------------------------------------------------------------------

$env:EDITOR = "code"
$env:VISUAL = "code"
$env:PYTHONUTF8 = "1"
$env:PYTHONUNBUFFERED = "1"

# ---------------------------------------------------------------------------
# Git helpers
# ---------------------------------------------------------------------------

function gs {
    git status --short --branch
}

function gd {
    git diff @args
}

function gl {
    git log --graph --decorate --oneline --all @args
}

function root {
    $repositoryRoot = git rev-parse --show-toplevel 2>$null

    if (!$repositoryRoot) {
        Write-Error "The current directory is not inside a Git repository."
        return
    }

    Set-Location $repositoryRoot
}

# ---------------------------------------------------------------------------
# Convenience helpers
# ---------------------------------------------------------------------------

function which {
    param(
        [Parameter(Mandatory, Position = 0)]
        [string]$Command
    )

    Get-Command $Command |
        Select-Object -ExpandProperty Source
}

function touch {
    param(
        [Parameter(Mandatory, Position = 0)]
        [string[]]$Path
    )

    foreach ($item in $Path) {
        if (Test-Path $item) {
            (Get-Item $item).LastWriteTime = Get-Date
            continue
        }

        New-Item -ItemType File -Path $item | Out-Null
    }
}

function mkcd {
    param(
        [Parameter(Mandatory, Position = 0)]
        [string]$Path
    )

    New-Item -ItemType Directory -Force -Path $Path | Out-Null
    Set-Location $Path
}
```

Commit this file to the dotfiles repository.

### 21A. Recommended: bootstrap `$PROFILE`

Make the real `$PROFILE` a one-line loader that dot-sources the repo file. This works even without symlink privileges, which is why it is the recommended path.

Find the PowerShell 7 profile location:

```powershell
$PROFILE
```

Create it:

```powershell
New-Item -ItemType File -Path $PROFILE -Force
```

Open it:

```powershell
notepad $PROFILE
```

Its **entire contents** should be:

```powershell
. "$HOME\code\github\YOUR_GITHUB_USERNAME\dotfiles\powershell\profile.ps1"
```

Notice that the repository target is named `profile.ps1`, **not** `Microsoft.PowerShell_profile.ps1`. Keeping the filenames distinct makes accidental self-recursion obvious and unlikely.

Reload:

```powershell
. $PROFILE
```

If the prompt appears normally, the bootstrap is working.

### 21B. Optional personal-machine alternative: symbolic link

Point `$PROFILE` directly at the repo file with a symbolic link. Use this only if you deliberately prefer a symlink.

Windows usually requires Administrator privileges to create a symlink unless **Developer Mode** is enabled. Corporate policy may prevent Developer Mode.

Enable Developer Mode only if permitted:

**Settings → System → Advanced / For developers → Developer Mode**

Then, after ensuring `$PROFILE` is not needed as a separate real file:

```powershell
Remove-Item $PROFILE -ErrorAction SilentlyContinue
```

Create the link:

```powershell
New-Item `
    -ItemType SymbolicLink `
    -Path $PROFILE `
    -Target "$HOME\code\github\YOUR_GITHUB_USERNAME\dotfiles\powershell\profile.ps1"
```

Verify:

```powershell
Get-Item $PROFILE | Format-List FullName,LinkType,Target
```

Expected:

```text
LinkType : SymbolicLink
Target   : C:\Users\<username>\code\github\<username>\dotfiles\powershell\profile.ps1
```

Again: **do not put a line in the symlink target that dot-sources `$PROFILE`.** That would source the same file recursively.

---

## Phase 10 — Verification

Prove the whole setup works end to end with one final pass through every tool.

### 22. Restart Windows Terminal and run the smoke test

Restart the terminal so every PATH and profile change takes effect, then run each tool once.

Close **all** Terminal windows. Reopen **Terminal** normally.

Run:

```powershell
$PSVersionTable.PSVersion
git --version
ssh -T git@github.com
uv --version
python --version
ipython --version
rg --version
fd --version
jq --version
fzf --version
bat --version
eza --version
zoxide --version
oh-my-posh version
Get-Command ll
```

Check the profile:

```powershell
$PROFILE
Get-Content $PROFILE
```

If using a symlink:

```powershell
Get-Item $PROFILE | Select-Object LinkType,Target
```

Open a repository and check Git helpers:

```powershell
cd $HOME\code\github\YOUR_ORG\YOUR_REPO
gs
```

Try directory navigation:

```powershell
z YOUR_REPO
```

Open the current directory in Explorer:

```powershell
explorer .
```

At this point the baseline setup is complete.

---

## Troubleshooting quick reference

The symptoms you are most likely to hit, each with its usual cause and fix. Every entry is self-contained — jump straight to the one that matches.

### `winget` says no package found

The package ID probably differs in your configured `winget` source.

Search first:

```powershell
winget search ripgrep
```

Package IDs can change. Use the package ID returned by the currently configured `winget` source.

On corporate PCs, the organization may expose only an internal package catalog.

### A newly installed command is "not recognized"

The install changed `PATH`, but the running shell still has the old environment.

Close **all** Terminal windows and reopen Terminal.

Then:

```powershell
Get-Command <COMMAND>
where.exe <COMMAND>
```

### `python` opens the Microsoft Store or says Python is missing

The Microsoft Store alias is still winning over the `uv`-managed Python.

Check:

```powershell
uv python list
```

Then:

```powershell
uv python update-shell
```

Restart Terminal and retry:

```powershell
python --version
```

### Virtual environment activation says scripts are disabled

The PowerShell execution policy is blocking the activation script.

Inspect:

```powershell
Get-ExecutionPolicy -List
```

Personal machine:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

Corporate machine: if policy is organization-enforced, ask IT. Do not use `Bypass` as a permanent workaround.

### `ssh-agent` configuration says `Access is denied`

The service configuration may require an elevated shell.

```powershell
Start-Process pwsh -Verb RunAs
```

Then configure the service from that Administrator window.

On a corporate machine, if elevation is unavailable or policy blocks the service, ask IT for the supported Git authentication path.

### GitHub SSH works at home but fails on the corporate network

The corporate network is interfering somewhere between the machine and GitHub; verbose SSH output shows where.

Collect:

```powershell
ssh -vT git@github.com
```

Typical causes:

- TCP/22 blocked;
- DNS filtering;
- outbound proxy;
- GitHub not allow-listed;
- organization requires SSO or managed credentials.

GitHub supports SSH on port 443, but use it only when allowed by company policy.

### Terminal starts Windows PowerShell 5.1

The default Terminal profile is still pointing at the legacy shell.

Check:

```powershell
$PSVersionTable.PSVersion
```

Open:

**Terminal → Settings → Startup → Default profile**

Choose **PowerShell** / PowerShell 7.

PowerShell 7 installs side-by-side with Windows PowerShell 5.1.

### Oh My Posh says `CONFIG NOT FOUND`

The prompt initialization is pointing at a theme path that does not exist.

Prefer a built-in theme name:

```powershell
oh-my-posh init pwsh --config "jandedobbeleer" | Invoke-Expression
```

rather than relying on a theme-directory environment variable that may not exist.

### The PowerShell profile hangs forever on startup

This is almost always profile recursion — the profile is loading itself.

Start PowerShell without loading the profile:

```powershell
pwsh -NoProfile
```

Then inspect:

```powershell
$PROFILE
Get-Item $PROFILE | Format-List FullName,LinkType,Target
Get-Content $PROFILE
```

A common cause is profile recursion:

```text
$PROFILE -> repo profile -> $PROFILE -> repo profile -> ...
```

If the Windows profile is a symlink, remember that **editing `$PROFILE` edits its target**.

### Disk-related installers fail in strange ways

A nearly full disk shows up as unrelated-looking errors.

Check:

```powershell
Get-Volume C | Select-Object DriveLetter, @{Name="FreeGB";Expression={[math]::Round($_.SizeRemaining/1GB,1)}}
```

A nearly full system disk can produce misleading installer, VS Code, Python, and package-manager errors.

---

## Notes for engineers supporting corporate teammates

How to help a Group A teammate remotely without making their machine worse: gather facts first, then change as little as possible.

### Remote diagnostics

Ask the teammate to paste the output of these small, non-destructive commands before suggesting any changes.

General state:

```powershell
$PSVersionTable
winget --version
git --version
uv --version
python --version
Get-ExecutionPolicy -List
Get-Volume C | Select-Object DriveLetter, @{Name="FreeGB";Expression={[math]::Round($_.SizeRemaining/1GB,1)}}
```

For a missing command:

```powershell
Get-Command <COMMAND> -ErrorAction SilentlyContinue
where.exe <COMMAND>
winget list
```

For SSH:

```powershell
Get-Service ssh-agent
ssh-add -l
ssh -vT git@github.com
```

For the PowerShell profile:

```powershell
$PROFILE
Test-Path $PROFILE
Get-Item $PROFILE | Format-List FullName,LinkType,Target
Get-ExecutionPolicy -List
```

### Support principles

Ground rules that keep remote help safe, reversible, and inside company policy.

- Distinguish **"Windows is different"** from **"the company intentionally prohibits this."**
- Do not tell non-engineers to click through security warnings they do not understand.
- Prefer company-approved package sources over random downloadable installers.
- Prefer per-user changes over machine-wide changes when both are legitimate.
- Keep changes reversible and version-controlled.
- Avoid manual Windows Terminal JSON edits unless the Settings UI genuinely cannot express the required change.
- Do not disable EDR/AV/firewalls or certificate validation to make a package manager work.
- If a corporate proxy/certificate breaks package installation, fix the trust/proxy configuration through the organization rather than adding insecure flags.
- Capture exact errors. Windows security products often provide a useful policy name, rule ID, event log, or blocked executable path that the infrastructure team can act on.

---

## Resulting baseline

What the finished machine looks like, at a glance.

A completed machine should conceptually look like:

```text
Windows Terminal
└── PowerShell 7
    ├── Oh My Posh + Nerd Font
    ├── Git + GitHub SSH
    ├── uv
    │   ├── Python
    │   └── IPython
    ├── rg
    ├── fd
    ├── jq
    ├── fzf
    ├── bat
    ├── eza
    └── zoxide
```

Repositories:

```text
~/code/github/{user-or-org}/{repo}
```

Python projects:

```text
repo/
├── .venv/
├── pyproject.toml
├── uv.lock
└── ...
```

Shell configuration:

```text
dotfiles/
└── powershell/
    └── profile.ps1
```

The result is a Windows-native environment with a Unix-friendly command-line workflow, without pretending PowerShell is Bash and without bypassing corporate security controls.

---

## Official references

The upstream documentation for every tool this guide installs, for when you need more depth than a setup guide provides.

- Microsoft — Windows Package Manager / WinGet: https://learn.microsoft.com/windows/package-manager/winget/
- Microsoft — Install PowerShell on Windows: https://learn.microsoft.com/powershell/scripting/install/install-powershell-on-windows
- Microsoft — Windows Terminal installation/settings: https://learn.microsoft.com/windows/terminal/install
- Microsoft — Developer Mode: https://learn.microsoft.com/windows/advanced-settings/developer-mode
- Microsoft — PowerShell `New-Item` / symbolic links: https://learn.microsoft.com/powershell/module/microsoft.powershell.management/new-item
- GitHub — Connecting to GitHub with SSH: https://docs.github.com/authentication/connecting-to-github-with-ssh
- GitHub — SSH over the HTTPS port: https://docs.github.com/authentication/troubleshooting-ssh/using-ssh-over-the-https-port
- Astral — `uv` installation: https://docs.astral.sh/uv/getting-started/installation/
- Astral — Managing Python with `uv`: https://docs.astral.sh/uv/guides/install-python/
- Oh My Posh — Windows installation: https://ohmyposh.dev/docs/installation/windows
- Oh My Posh — Nerd Fonts: https://ohmyposh.dev/docs/installation/fonts
- Oh My Posh — PowerShell prompt setup: https://ohmyposh.dev/docs/installation/prompt
- zoxide — official repository/docs: https://github.com/ajeetdsouza/zoxide
