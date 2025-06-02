---
layout: post
title: "Using the Bitwarden SSH Agent in WSL2 Ubuntu"
category: Bitwarden SSH
---

## Using the Bitwarden SSH Agent in WSL2 (Ubuntu)

Using SSH keys in **WSL2** (Ubuntu) when using **Bitwarden** as the SSH agent can be implemented with a few brief (scripted) steps. This post introduces the script designed to link the Bitwarden SSH agent through to your WSL2 environment (for Ubuntu only).

### Script Functionality

The `setup-bw-ssh-agent.sh` script automates the necessary configurations:

* **Installs `npiperelay`**: A utility that exposes Windows named pipes as standard input/output streams, enabling WSL processes to interact with services (like the OpenSSH agent) running on the Windows host via named pipes.
* **Installs `socat`**: A powerful command-line utility that establishes two bidirectional byte streams and transfers data between them, crucial for bridging the Unix socket to the npiperelay process for SSH agent forwarding.
* **Creates `agent-bridge.sh`**: This script establishes and forwards a Unix socket to your Bitwarden SSH agent on Windows.
* **Integrates with `.bashrc`**: The bridge script is automatically sourced in your `.bashrc`, ensuring the SSH agent is available upon shell startup.

### Usage

To set up the Bitwarden SSH agent in your WSL2 Ubuntu environment, execute the following in your terminal:

```bash
wget https://gist.githubusercontent.com/jkwmoore/ce9eab106fe378709f447c843b0090e4/raw/setup-bw-ssh-agent.sh && bash setup-bw-ssh-agent.sh
```

<script src="https://gist.github.com/jkwmoore/ce9eab106fe378709f447c843b0090e4.js"></script>

After script execution, restart your shell or run `source ~/scripts/agent-bridge.sh` to activate the changes.

---

Thanks go to Aaron and the original work discussed here: https://www.rebelpeon.com/bitwarden-ssh-agent-on-wsl2/