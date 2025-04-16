# Ansible-Proxmox-inventory

[![License: GPL v3](https://img.shields.io/badge/license-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)

This script, `proxmox.py`, is a dynamic inventory script for Ansible that automatically discovers virtual machines (VMs) and containers within a Proxmox VE cluster. It generates inventory data that Ansible can use to manage these resources efficiently.

## Ansible Control Node Prerequisites

- **Python 3:** Ensure Python 3 is installed.
- **Ansible:** Ansible must be installed.
- **`six` Python Library:** Install the `six` library:

```bash
pip3 install six
```

## Proxmox VE Server Prerequisites

### PVE User Permissions

Before proceeding, ensure that the Proxmox VE user account has been granted the necessary permissions within the Proxmox interface. This means:
- The user must have access rights set under **Datacenter > Permissions** for the nodes or pools that include the VMs and containers you plan to manage.
-	Without proper permissions, even if the user account is valid, the API will not be able to retrieve resource details.

Inadequate permissions will result in incomplete inventory data, such as an empty hosts list.

### PVE Role Privileges

For the dynamic inventory script to successfully fetch detailed VM and container information including network IP addresses the Proxmox VE user’s role must include specific privileges:

- **VM.Audit:**

  Provides access to general information about virtual machines. This includes configuration data and system status. However, it does not cover real-time network details.

- **VM.Monitor:**

  This additional privilege is required to access dynamic data such as network statistics, including the IP addresses of your VMs. Without VM.Monitor, endpoints like `/api2/json/nodes/<node>/qemu` might return VM information that lacks the required network details.

## Create API Token

When creating the API token, adhere to the following guidelines:

- **Generate the API Token with a Valid User:**

  The token must be created by a Proxmox user who already has the appropriate permissions as described above.
  
- **Verify Proper Role Assignment:**

  Ensure that the user's role (for example, Administrator or a custom role with equivalent privileges) is correctly set under **Datacenter > Permissions** for the target nodes or pools.
  
- **Consequences of Insufficient Permissions:**

  If the API token is associated with a user lacking the necessary rights, API calls (e.g., to `/api2/json/nodes/<node>/qemu`) will fail to retrieve any host information. In such cases, the script will output an inventory structure similar to:

  ```json
  {
    "all": { "hosts": [] },
    "_meta": { "hostvars": {} }
  }
  ```

## Installation

1. **Download the Script:** Download `proxmox.py` and place it in a suitable location (e.g., a directory in Ansible project).
2. **Make it Executable:**

```bash
chmod +x proxmox.py
```

## Configuration

### `proxmox.json` (Recommended)

Create a `proxmox.json` file in the same directory as `proxmox.py` to store your Proxmox VE connection details. This is the recommended approach for security.

```json
{
  "url": "https://<your_proxmox_ip_or_hostname>:8006/",
  "username": "<your_proxmox_username>",
  "validateCert": false,
  "include_iface_patterns": ["eth.*"],
  "exclude_iface_patterns": [],
  "token": "<your_api_token_name>",
  "secret": "<your_api_token_secret>"
}
```

> **Note:** If you use `proxmox.json`, you can omit the corresponding command-line arguments.

### Environment Variables (Alternative)

You can also use environment variables to configure the script:

```bash
export PROXMOX_URL=https://<your_proxmox_ip_or_hostname>:8006/
export PROXMOX_USERNAME=<your_proxmox_username>
export PROXMOX_TOKEN=<your_api_token_name>
export PROXMOX_SECRET=<your_api_token_secret>
export PROXMOX_INVALID_CERT=False  # Set to False to disable certificate validation
```

### Command-Line Arguments (Not Recommended for Credentials)

You can also pass connection details as command-line arguments, but this is not recommended for sensitive information like passwords or API secrets.

## Usage

### Listing Hosts and Groups

```bash
./proxmox.py --list
```

This command will output a JSON representation of your Proxmox VE inventory, including hosts, groups, and host variables.

### Getting Host Information

```bash
./proxmox.py --host <hostname>
```

Replace `<hostname>` with the name of a specific VM or container to get its detailed information.

### Using Command-Line Arguments

```bash
./proxmox.py --list --url https://<your_proxmox_ip_or_hostname>:8006/ --username <your_proxmox_username> --token <your_api_token_name> --secret <your_api_token_secret> --validate false
```

### Filtering Network Interfaces

```bash
./proxmox.py --list --exclude-iface "docker.*"
```

Or configure it in `proxmox.json`:

```json
{
  "url": "https://<your_proxmox_ip_or_hostname>:8006/",
  "username": "<your_proxmox_username>",
  "validateCert": false,
  "include_iface_patterns": ["eth.*"],
  "exclude_iface_patterns": ["eth0"],
  "token": "<your_api_token_name>",
  "secret": "<your_api_token_secret>"
}
```

### Using with Ansible

#### Option 1: In your `ansible.cfg` file

```ini
[defaults]
inventory = ./proxmox.py
```

#### Option 2: Use the `-i` flag when running Ansible commands

```bash
ansible-playbook -i ./proxmox.py your_playbook.yml
```

### List all VMs and containers

```bash
python ./proxmox.py --list --pretty
```

### Get details for a specific host

```bash
python ./proxmox.py --host <hostname>
```

### Limit by group (e.g., running only)

```bash
ansible-playbook -i ./proxmox.py --limit='running' playbook.yml
```

## Key Improvements

1. **Improved Naming for Interface Filters**: `include_iface_patterns` and `exclude_iface_patterns` offer better clarity and configuration flexibility.
2. **Secure Authentication**: Only supports authentication via Proxmox API token. Password-based login has been removed for better security.