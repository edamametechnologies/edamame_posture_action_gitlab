# EDAMAME Posture Setup for GitLab

## Overview

The GitLab CI configuration ensures:
- **Dependency Installation**: Installs any missing packages such as Git, `libpcap`, `wget`, `curl`, etc., depending on your Linux distribution or OS.  
- **Binary Download/Detection**: Checks if the EDAMAME Posture binary is already available; if not, downloads the appropriate build for your runner’s OS and CPU architecture (Linux x86_64, Alpine ARM, macOS universal, Windows x86_64, etc.).  
- **Posture Setup**: Optionally starts the EDAMAME Posture service if the required credentials (`EDAMAME_USER`, `EDAMAME_DOMAIN`, `EDAMAME_PIN`, `EDAMAME_ID`) are provided.  
- **Remediations**: Performs posture remediations automatically if configured.  
- **Checkout**: Allows fetching your code via Git (with retries), which can be handy if your environments or networks require special whitelisting.  

## Requirements

- A functioning GitLab runner (self-hosted or GitLab-hosted).  
- Basic knowledge of `.gitlab-ci.yml` structure.  
- Optional: Credentials for EDAMAME Posture (user, domain, pin, ID) to start the posture service.

## Usage

1. **Include the Script in Your `.gitlab-ci.yml`:**

   Add a reference to the `setup_edamame_posture.yaml` from this repository. To do this, you can either:
   - Copy the contents directly into your `.gitlab-ci.yml`.
   - Or store `setup_edamame_posture.yaml` in your project and use Git submodule or a snippet to include it.

   For instance:

   ```yaml
   include:
     - local: edamame_posture_action_gitlab/setup_edamame_posture.yaml
   ```

   Or if you prefer to reference it remotely:

   ```yaml
   include:
     - project: 'your-org/edamame-posture-templates'
       file:
         - 'setup_edamame_posture.yaml'
   ```

2. **Use the Reusable Job or Extends Mechanism:**

   The `setup_edamame_posture.yaml` defines a reusable block (`.setup_edamame_posture`) for Linux/macOS and a separate block (`.setup_edamame_posture_windows`) for Windows. You can extend or call these from your pipeline stages:

   ```yaml
   stages:
     - setup
     - build
     - test

   setup_edamame_posture:
     stage: setup
     # "extends" the reusable config:
     extends: .setup_edamame_posture
     variables:
       EDAMAME_USER: "myuser"
       EDAMAME_DOMAIN: "mydomain"
       EDAMAME_PIN: "1234"
       EDAMAME_ID: "my-runner"  # Typically, an identifier for your runner
       AUTO_REMEDIATE: "true"
       CHECKOUT: "true"

   # Example usage on Windows (if needed):
   setup_edamame_posture_windows:
     stage: setup
     extends: .setup_edamame_posture_windows
     variables:
       EDAMAME_USER: "myuser"
       EDAMAME_DOMAIN: "mydomain"
       EDAMAME_PIN: "1234"
       EDAMAME_ID: "my-runner-windows"
       AUTO_REMEDIATE: "false"
   ```

3. **Environment Variables**

   Below is a partial list of environment variables recognized by `setup_edamame_posture.yaml`:

   | Variable               | Description                                                                    | Default   |
   |------------------------|--------------------------------------------------------------------------------|-----------|
   | `EDAMAME_USER`        | EDAMAME Posture user (required for starting the posture process).              | *(none)*  |
   | `EDAMAME_DOMAIN`      | EDAMAME Posture domain (required for starting the posture process).            | *(none)*  |
   | `EDAMAME_PIN`         | EDAMAME Posture PIN (required for starting the posture process).               | *(none)*  |
   | `EDAMAME_ID`          | Identifier for the EDAMAME Posture session.                                    | *(none)*  |
   | `AUTO_REMEDIATE`      | Automatically fix posture issues if `true`.                                     | `false`   |
   | `SKIP_REMEDIATIONS`   | Comma-separated list of remediations to skip.                                   | *(none)*  |
   | `NETWORK_SCAN`        | If `true`, runs a network scan for critical endpoints.                          | `false`   |
   | `DUMP_SESSIONS_LOG`   | If `true`, attempts to dump session logs (limited support on Windows).         | `false`   |
   | `CHECKOUT`            | If `true`, attempts to fetch/checkout the repo.                                | `false`   |
   | `CHECKOUT_SUBMODULES` | If `true`, also checks out submodules.                                         | `false`   |
   | `DISPLAY_LOGS`        | If `true`, shows posture logs after setup.                                     | `true`    |
   | `WHITELIST`           | (Optional) Whitelist name for network scanning context.                         | `github`  |
   | `WAIT`                | If `true`, waits for 180 seconds at a certain stage.                            | `false`   |
   | `WHITELIST_CONFORMANCE` | If `true`, pipeline fails if posture logs show non-compliant endpoints.       | `false`   |
   | `CI_JOB_TOKEN`        | Provided by GitLab for repository checkout if `CHECKOUT` is enabled.            | *(auto)*  |

   **Important**: For the posture process to connect, you **must** set `EDAMAME_USER`, `EDAMAME_DOMAIN`, `EDAMAME_PIN`, and `EDAMAME_ID`. If you omit these, the script will skip or fail the posture connection setup.

4. **Behavior in Self-Hosted vs. GitLab-Hosted Runners**

   - **Self-Hosted Runners**  
     If EDAMAME Posture is already installed on the runner, the script detects the existing binary and checks whether the runner is already connected. If so, the connection step is skipped unless you specifically override with new credentials. If the runner’s posture service is not connected or missing fields, it prompts to configure them properly.

   - **GitLab-Hosted or Other Hosted Runners**  
     If EDAMAME Posture is not installed, the script will:
     1. Determine your runner’s OS (Linux, macOS, or Windows).
     2. Download the matching binary (x86_64, ARM, etc.).
     3. Optionally start EDAMAME Posture with the provided credentials and wait for successful connection.

5. **Steps Outline**

   The `setup_edamame_posture.yaml` script runs through several logical steps:

   1. **Detect OS and Install Dependencies (Linux/macOS)**  
      - Checks `/etc/os-release` to see if it’s Ubuntu, Alpine, CentOS/RHEL, etc.  
      - Installs needed packages (`git`, `libpcap`, `wget`, etc.).  
      - Creates a pseudo `sudo` if necessary (e.g., on Alpine Docker images).  

   2. **Windows Setup (if needed)**  
      - Installs Git with Git Bash if not present.  
      - Uses PowerShell to prepare a Bash script that downloads the EDAMAME Posture CLI.

   3. **Download EDAMAME Posture**  
      - Checks if the binary already exists and, if not, downloads the appropriate build for the OS/architecture.  
      - Makes it executable.

   4. **Score and Remediate**  
      - Displays the current posture score (`./edamame_posture score`).  
      - If `AUTO_REMEDIATE` is `true`, runs `./edamame_posture remediate` (with optional skip list).

   5. **Wait**  
      - If `WAIT` is `true`, sleeps 180 seconds (useful if your environment requires extra steps or whitelisting to become active).

   6. **Start Posture Connection**  
      - If the four required environment variables (`EDAMAME_USER`, `EDAMAME_DOMAIN`, `EDAMAME_PIN`, `EDAMAME_ID`) are set, attempts to start the posture service in the background.  
      - Waits for a successful connection with `./edamame_posture wait-for-connection`.

   7. **Repo Checkout**  
      - If `CHECKOUT` is `true`, tries multiple attempts to clone the repo with `git` using `$CI_JOB_TOKEN`, in case your runner is behind a firewall or requires extra permissions.  
      - If `CHECKOUT_SUBMODULES` is also `true`, initializes submodules.

   8. **Logs and Session Information**  
      - If `DISPLAY_LOGS` is `true`, prints posture logs.  
      - If `DUMP_SESSIONS_LOG` is `true` (and on a supported OS), attempts to retrieve session details.

