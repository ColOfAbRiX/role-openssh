# openssh

Ansible role to configure SSH clients and servers with comprehensive customization options.

## Description

This role provides flexible and feature-rich configuration for OpenSSH servers and clients across
your infrastructure. It takes into consideration configurations for jump hosts, allowing transparent
use of SSH gateways. The role also offers special compatibility for WSL (Windows Subsystem for
Linux) environments.

Key capabilities include:

- Comprehensive SSH server (sshd) configuration
- Client-side SSH configuration with advanced options
- SSH agent management and configuration
- Support for jump hosts and SSH gateways
- Special handling for WSL environments
- Modular configuration approach for easy customization

## Requirements

- **Supported distributions**:
  - CentOS/RedHat
  - Ubuntu 16.0+
  - Debian 6.0+
- **OpenSSH** packages (automatically installed by the role)

## Role Variables

The role provides extensive configuration options for both SSH server and client settings. Below
are the primary variables:

### General Configuration Variables

| Variable                   | Default                       | Description                                                    |
| :---                       | :---                          | :---                                                           |
| `ssh_version`              | "" (empty string)             | OpenSSH version to install (empty for default, "latest" for latest) |
| `ssh_server_enabled`       | `yes`                         | Whether to enable the SSH daemon service                       |
| `ssh_config_path`          | `/etc/ssh`                    | Path to SSH configuration directory                           |
| `ssh_generate_server_keys` | `false`                       | Whether to generate new SSH server keys                        |
| `ssh_generate_user_config` | `false`                       | Whether to recreate user SSH configurations                   |
| `ssh_host_keys`            | List of standard key types    | SSH server keypairs to manage                                 |
| `ssh_authkeys_files`       | `~/.ssh/authorized_keys`      | Path to authorized keys files                                 |
| `ssh_identity_files`       | `~/.ssh/id_rsa`               | Path to user identity files                                   |
| `ssh_knownhosts_files`     | `~/.ssh/known_hosts`          | Path to known hosts files                                     |
| `ssh_agent_enabled`        | `false`                       | Whether to enable the SSH agent                               |
| `ssh_agent_keys_timeout`   | `1800` (30 minutes)           | Timeout for SSH agent keys                                    |

### SSH Server Configuration

The role supports all standard sshd_config options, which can be set through variables with the
`sshd_` prefix. Some common examples include:

| Variable                   | Default                       | Description                                                    |
| :---                       | :---                          | :---                                                           |
| `sshd_Protocol`            | `2`                           | SSH protocol version to use                                    |
| `sshd_Port`                | `22`                          | Port for SSH server to listen on                               |
| `sshd_ListenAddresses`     | `[0.0.0.0]`                   | Addresses to listen on                                         |
| `sshd_PermitRootLogin`     | `no`                          | Whether to allow root login                                    |
| `sshd_PasswordAuthentication` | (not set)                  | Whether to allow password authentication                       |
| `sshd_PubkeyAuthentication` | (not set)                    | Whether to allow public key authentication                     |
| `sshd_AuthenticationMethods` | `['password keyboard-interactive publickey']` | Authentication methods to use             |
| `sshd_UsePAM`              | `yes`                         | Whether to use PAM authentication                              |
| `sshd_X11Forwarding`       | (not set)                     | Whether to allow X11 forwarding                                |
| `sshd_AllowAgentForwarding` | (not set)                    | Whether to allow SSH agent forwarding                          |
| `sshd_AllowTcpForwarding`  | (not set)                     | Whether to allow TCP port forwarding                           |

### SSH Client Configuration

The role supports all standard ssh_config options, which can be set through variables with the
`ssh_` prefix. Some common examples include:

| Variable                   | Default                       | Description                                                    |
| :---                       | :---                          | :---                                                           |
| `ssh_Protocol`             | `2`                           | SSH protocol version to use                                    |
| `ssh_Port`                 | `22`                          | Default port for SSH connections                               |
| `ssh_AddressFamily`        | `inet`                        | Address family to use (inet for IPv4)                          |
| `ssh_ControlMaster`        | `auto`                        | Connection multiplexing setting                                |
| `ssh_IdentityFile`         | `ssh_identity_files` value    | Path to identity (private key) files                           |
| `ssh_UserKnownHostsFile`   | `ssh_knownhosts_files` value  | Path to user known hosts file                                  |
| `ssh_GlobalKnownHostsFile` | `{{ ssh_config_path }}/ssh_known_hosts` | Path to system-wide known hosts file               |
| `ssh_ForwardAgent`         | (not set)                     | Whether to forward SSH agent                                   |
| `ssh_ForwardX11`           | (not set)                     | Whether to forward X11 connections                             |

The role supports many more configuration options. See the [defaults file](defaults/main.yml) for a
complete list with detailed descriptions.

## Host and Match Configuration

### SSH Server Host-Specific Configuration

The `sshd_Matches` variable allows for conditional blocks in the sshd_config file:

```yaml
sshd_Matches:
  - user: deploy
    address: "10.0.0.0/8"
    keywords:
      PasswordAuthentication: "no"
      PubkeyAuthentication: "yes"
      ForceCommand: "/usr/bin/scponly"
```

### SSH Client Host-Specific Configuration

The `ssh_Hosts` variable allows for host-specific configuration blocks in ssh_config:

```yaml
ssh_Hosts:
  - Host: "bastion.example.com"
    User: "jump_user"
    Port: 2222
    IdentityFile: "~/.ssh/jump_key"
  - Host: "*.internal"
    ProxyCommand: "ssh -W %h:%p bastion.example.com"
    User: "app_user"
```

## Dependencies

No dependencies for this role.

## Example Playbook

Here's a comprehensive example showing various capabilities of the role:

```yaml
- hosts: servers
  roles:
    - role: openssh
      ssh_server_enabled: yes
      ssh_version: latest
      ssh_agent_enabled: yes
      ssh_agent_keys_timeout: 3600

      # SSH Server Configuration
      sshd_Port: 2222
      sshd_ListenAddresses:
        - 0.0.0.0
        - "[::]"  # IPv6
      sshd_PermitRootLogin: "no"
      sshd_PasswordAuthentication: "no"
      sshd_PubkeyAuthentication: "yes"
      sshd_AuthenticationMethods: ["publickey"]
      sshd_X11Forwarding: "no"
      sshd_AllowAgentForwarding: "no"
      sshd_AllowTcpForwarding: "yes"

      # Conditional blocks for specific users/sources
      sshd_Matches:
        - user: deploy
          address: "10.0.0.0/8"
          keywords:
            PasswordAuthentication: "no"
            PermitTTY: "no"
            ForceCommand: "/usr/bin/deploy-only"

      # SSH Client Configuration
      ssh_Protocol: 2
      ssh_ServerAliveInterval: 60
      ssh_ServerAliveCountMax: 3
      ssh_ControlMaster: auto
      ssh_ControlPersist: 10m

      # Host-specific client configuration
      ssh_Hosts:
        - Host: "*"
          ForwardAgent: "no"
          AddressFamily: inet
        - Host: "bastion.example.com"
          User: jump_user
          Port: 22
          IdentityFile: "~/.ssh/jump_key"
        - Host: "*.internal"
          ProxyCommand: "ssh -W %h:%p bastion.example.com"
          ForwardAgent: "yes"
```

## WSL Compatibility

The role includes special handling for Windows Subsystem for Linux (WSL) environments, where
standard systemd service management might not work. It provides fallback mechanisms for starting
and stopping the SSH service in these environments.

If you're running this role in a WSL environment, the role will:

1. Attempt to use the standard service module first
2. If that fails, fall back to direct service command execution
3. Suppress failures that are common in WSL environments

This ensures the role works correctly even in environments with limited service control capabilities.

## Advanced Usage

### Setting Up Jump Host Configuration

To configure a jump host (bastion) setup:

```yaml
ssh_Hosts:
  - Host: "bastion.example.com"
    User: jump_user
    IdentityFile: "~/.ssh/bastion_key"
    ControlMaster: auto
    ControlPath: "~/.ssh/cm_%h_%p_%r"
    ControlPersist: 10m

  - Host: "*.production"
    ProxyCommand: "ssh -W %h:%p bastion.example.com"
    User: app_user
    IdentityFile: "~/.ssh/app_key"
```

### Restricting SSH Access

To restrict SSH access to specific users or groups:

```yaml
sshd_AllowUsers: ["admin", "deploy"]
sshd_DenyUsers: ["guest"]
sshd_AllowGroups: ["ssh-users", "admins"]
sshd_DenyGroups: ["restricted"]
```

### Enabling SSH Agent Forwarding Securely

To enable SSH agent forwarding only for specific hosts:

```yaml
ssh_Hosts:
  - Host: "*"
    ForwardAgent: "no"
  - Host: "trusted-server.example.com"
    ForwardAgent: "yes"
```

## Security Recommendations

When configuring SSH, consider the following security best practices:

1. **Disable root login**: Set `sshd_PermitRootLogin` to "no"
2. **Use key-based authentication**: Set `sshd_PasswordAuthentication` to "no"
3. **Restrict forwarding**: Disable `sshd_AllowTcpForwarding` and `sshd_X11Forwarding` unless needed
4. **Implement strong ciphers**: Configure `sshd_Ciphers` with strong encryption algorithms
5. **Use SSH agent with timeouts**: When enabling `ssh_agent_enabled`, set a reasonable timeout
6. **Implement host-specific configurations**: Use `ssh_Hosts` and `sshd_Matches` for targeted rules
7. **Restrict allowed users**: Use `sshd_AllowUsers` and `sshd_AllowGroups` to limit access

## Troubleshooting

### Common Issues

1. **SSH service won't start**:
   - Check for configuration errors with `sshd -t`
   - Verify permissions on key files (should be 600 for private keys)
   - Look for conflicting settings in match blocks

2. **Client configuration not taking effect**:
   - Verify file permissions on `ssh_config` (should be 644)
   - Check for conflicting Host blocks (more specific hosts should be defined first)
   - Restart SSH session to apply changes

3. **WSL-specific issues**:
   - If service commands fail, try manually starting SSH with `sudo service ssh start`
   - Verify that OpenSSH server is installed with `apt list --installed | grep openssh-server`
   - Check Windows Firewall settings if connecting from external machines

4. **Permission denied errors**:
   - Verify key permissions (private keys should be 600, public keys 644)
   - Check that the correct key is being offered (use `ssh -v` for verbose logging)
   - Verify the key is properly added to authorized_keys on the server

## License

MIT

## Author Information

Fabrizio Colonna (@ColOfAbRiX)

## Contributors

Issues, feature requests, ideas, suggestions, etc. are appreciated and can be posted in the Issues
section.

Pull requests are also very welcome. Please create a topic branch for your proposed changes. If you
don't, this will create conflicts in your fork after the merge.
