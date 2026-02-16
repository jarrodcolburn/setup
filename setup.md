# Agent System Provisioning Protocol

**Objective:** Configure this machine using `apps.yaml` as the source of truth.
**Role:** You are an expert SysAdmin. You are safe, interactive, and observant.

## Phase 1: System Discovery
Before installing anything, run the following to understand the environment:
1. `cat /etc/os-release` (Identify Distro/Version)
2. `uname -a` (Identify Architecture: x86_64 vs aarch64)
3. `id` (Check sudo privileges)
4. `mkdir -p artifacts` (Ensure log directory exists)
5. `sudo apt update -qq && sudo apt install -y -qq curl wget lsb-release gpg` (Install common dependencies)

## Phase 2: Interactive Installation Loop
Load `apps.yaml`. Iterate through each **group** defined in the file.

**Pre-flight Check:**
- Proactively update the current shell's `PATH` to include common user-specific binary directories, such as `$HOME/.local/bin` and `$HOME/.cargo/bin`, to ensure installed tools are immediately found. This can be done with `export PATH="$HOME/.local/bin:$HOME/.cargo/bin:$PATH"`.
- **Recommended:** Run the **Update Strategy** commands below before proceeding with new installations to ensure the system and package managers are up-to-date.

**Update Strategy (Periodic/Maintenance):**
To keep the system and global tools updated with minimal output, run:
- **System (Apt):** `sudo DEBIAN_FRONTEND=noninteractive apt update -qq && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y -qq >> artifacts/apt.log 2>&1`
- **Python (UV):** `uv tool upgrade --all -q`
- **Node (NPM):** `npm update -g --silent`

**For each Group:**
1.  **Announce:** "Ready to setup [Group Name]: [Description]".
2.  **Ask:** "Do you want to process this group, skip it, or select specific apps?"
3.  **Action:**
    * If **Process**: Iterate through items.
    * If **Skip**: Move to next group.
    * If **Select**: Ask user which items to install.

**For each Item:**
1.  **Dependency Check:** Before installation, check for a `depends_on` field in `apps.yaml`. If present, ensure all listed applications are already installed. If not, install the dependencies first.
2.  **Check Existence:** Run the `verify_cmd`. **This is the most reliable method:** `exec bash -ic 'which <command>'`. This ensures that user-specific paths are checked.
    * *If command returns 0/Success:* Output "✅ [App] already installed." and **SKIP** installation.
    * *If command fails:* Proceed to install.

3.  **Installation Strategy (Priority Order):**
    * **Note on PATH:** If an installation script modifies the `PATH` (e.g., `rustup`, `uv`), it may not be available in the current session. If a command fails with "not found" after installation, use its full path, which can be found with `exec bash -ic 'which <command>'`.
    * **Rule #0:** **NO SNAP. NO FLATPAK.** (Unless explicitly authorized by user).
    * **Rule #1 (System):** Use `apt` with quiet flags and redirect output to `artifacts/apt.log` for debugging.
      `sudo DEBIAN_FRONTEND=noninteractive apt install -y -qq [package] >> artifacts/apt.log 2>&1`
    * **Rule #2 (.deb/3rd Party):** Use `deb-get` if available and the app is supported. Use the `--quiet` flag. If `deb-get` fails, attempt to install with `apt`.
      `sudo deb-get install --quiet [package]`
      **Note:** Be aware that some `deb-get` packages may still prompt for interactive input (e.g., EULA agreement).
    * **Rule #3 (Python):** Use `uv tool install -q [package]`. Never use system pip.
    * **Rule #4 (Rust):** Use `rustup` for toolchains and `cargo install -q [package]` to install binaries.
    * **Rule #5 (Node):** Use `npm install -g --quiet [package]`.
    * **Rule #6 (Script):** If a `script` method is specified, do not pipe it directly to a shell.
        1. Download the script using `curl` or `wget` to `/tmp`.
        2. Inspect the script's contents (e.g., with `head` or `read_file`) to understand its actions and identify any useful flags (e.g., `--quiet`, `--non-interactive`, `-y`).
        3. Execute the script with the identified flags.
        4. After execution, verify that the application's binary is available in the `PATH`. If not, and the script did not provide instructions, add it to the appropriate shell profile file (e.g. `~/.bashrc`).
    * **Rule #7 (Manual Binary):** If a `manual_binary` method is specified, download the binary from `install_ref` to `/tmp`, extract it if necessary, and move the executable to `/usr/local/bin`.
    * **Rule #8 (Manual):** If a URL is provided in `install_ref`, read the site (using your browser tool) or curl the page to find the `.deb` or install script.
    * **Rule #9 (Apt with Repo):** If an `apt_with_repo` method is specified, execute the commands in the `notes` field to add the repository, then run `apt update` and `apt install`.
    * **Rule #10 (AppImage):** When installing an AppImage manually:
        1. Download the latest version from the official repository (usually GitHub) using `curl -L` to `/tmp`.
        2. Make the file executable with `chmod +x`.
        3. Move the file to `/usr/local/bin/` and name it as the primary command (e.g., `/usr/local/bin/nvim`).
        4. If the AppImage requires FUSE (common on Debian), inform the user or ensure `libfuse2` is available.
    * **Rule #11 (Git Clone):** If a `git_clone` method is specified:
        1. Clone the repository from `install_ref` to the path specified in the `verify_cmd` or `notes`.
        2. Perform any necessary setup steps described in the `notes` (e.g., adding to shell profile).

## Phase 3: Error Handling & Self-Correction
- **If an install fails:** 1. Analyze the error. 
  2. Check the `install_notes` section in `apps.yaml` to see if we solved this before.
  3. Attempt ONE logical fix (e.g., missing dependency, wrong arch, trying `apt` if `deb-get` fails).
  4. If it still fails, ask the user for guidance.
- **On Success (with workaround):** - If you had to use a special workaround (e.g., "Installed libssl1.1 manually to make X work"), you MUST append a brief note to the `install_notes` section of `apps.yaml` so we remember it for next time.

## Phase 4: Final Report
At the end, output a summary table. Use the `version_cmd` field from `apps.yaml` to get the version of each application. If `version_cmd` is not available, try to get the version using `--version` or similar standard flags.
The final report will be saved as `artifacts/installation_report.md`.
| App | Status | Version | Notes |
|-----|--------|---------|-------|
| ... | ...    | ...     | ...   |

## Phase 5: Configuration Deployment
After software installation, offer to deploy configuration templates from the `configs/` directory.

1. **Scan**: List available configurations in `configs/`.
2. **Ask**: "Would you like to deploy configuration templates for these applications?"
3. **Action**:
   * For each selected app:
     1. Locate the target path in the home directory.
        - `configs/zsh/zshrc` -> `~/.zshrc`
        - `configs/zsh/zsh_plugins.txt` -> `~/.zsh_plugins.txt`
        - `configs/zsh/p10k.zsh` -> `~/.p10k.zsh`
        - `configs/gemini/settings.json` -> `~/.gemini/settings.json`
     2. If the target file exists, ask the user if they want to **Backup & Overwrite**, **Skip**, or **Diff**.
     3. Perform the action and log it in `artifacts/config_deployment.log`.
