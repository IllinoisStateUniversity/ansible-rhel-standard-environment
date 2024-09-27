# RHEL - Standard Environment

## Overview

This Ansible playbook automates the setup of a standard environment on Red Hat Enterprise Linux (RHEL) servers. It is designed to run within Ansible Automation Platform (AAP) or AWX and includes roles for configuring common system settings, time synchronization, networking, package installation, firewall configuration, authentication, and user management.

## Features

- **Hostname Configuration**: Sets the server's hostname to its fully qualified domain name (FQDN).
- **Time Synchronization**: Configures the system timezone and synchronizes time using NTP servers.
- **Networking Configuration**: Sets up `dnsmasq` for DNS caching and local DNS resolution.
- **Package Installation**: Installs essential packages, including Cockpit for web-based system management.
- **Firewall Configuration**: Manages firewall settings using RHEL System Roles.
- **Authentication**: Supports both indirect and direct Active Directory (AD) integration.
  - **Indirect AD Integration**: Configures Kerberos and SSSD for AD authentication without domain joining.
  - **Direct AD Integration**: Joins the server to the AD domain using realm and configures SSSD.
- **User Management**: Manages local and AD users, including sudo privileges.

## Prerequisites

- **Ansible Automation Platform (AAP) or AWX**: The playbook is intended to run within AAP or AWX.
- **Ansible Version**: Compatible with Ansible 2.9 or later.
- **Collections**: Ensure the required Ansible collections are installed:
  - `community.general`
  - `redhat.rhel_system_roles`
  - `ansible.posix`
  - `ansible.controller`
  - `redhat.insights`
  - `fedora.linux_system_roles`
- **Variables**: Ensure that variables are filled out within playbook.

## Inventory Setup

- **Hosts**: Define target hosts in your inventory.
- **Group Names**: Use group names to control role execution:
  - `tag_application_bind`: Skips the `networking` role when present.
  - `tag_application_satellite`: Skips the `packages` role when present.
  - `tag_environment_dev`, `tag_environment_test`, `tag_environment_prod`: Sets environment-specific variables.
  - `tag_resourceowner_*`: Specifies which team owns the server.
  - `config_redhat_environment_standard_adjoin`: Determines direct AD integration method.
  - `config_redhat_environment_standard_samba`: Enables Samba configurations.

## Usage

1. **Customize Variables**:
   - Update `vars/aap_prod.yml` and `vars/aap_test.yml` with your controller credentials.
   - Edit `roles/auth/defaults/main.yml` with AD integration credentials.
   - Modify any other variables in `defaults/main.yml` as needed.
2. **Run the Playbook**:
   - **From AAP/AWX**:
     - Create a project and sync it with your repository.
     - Create an inventory and define your hosts and groups.
     - Create a job template using the `site.yml` playbook.
     - Configure surveys if needed for variables like `survey_hostname`.
     - Launch the job template to run the playbook.

## Roles and Their Functions

### Common Role

- **Tasks**:
  - Sets the hostname to the server's FQDN.
  - Adds the server to the `config_redhat_environment_standard` group in the Ansible Controller inventory.

### Time Role

- **Tasks**:
  - Sets the system timezone.
  - Configures NTP servers using the `timesync` RHEL system role.

### Networking Role

- **Condition**: Skipped if `tag_application_bind` is in group names.
- **Tasks**:
  - Installs and configures `dnsmasq`.
  - Modifies `/etc/resolv.conf` and disables NetworkManager's DNS plugin.

### Packages Role

- **Condition**: Skipped if `tag_application_satellite` is in group names.
- **Tasks**:
  - Registers the server with Red Hat Satellite.
  - Installs Cockpit and other specified packages.
  - Registers the server with Red Hat Insights.

### Firewall Role

- **Tasks**:
  - Configures firewall settings using the `firewall` RHEL system role.
  - Firewall rules should be defined in `roles/firewall/defaults/main.yml`.

### Auth Role

- **Tasks**:
  - **Indirect AD Integration**:
    - Installs Kerberos and SSSD packages.
    - Configures Kerberos (`krb5.conf`) and SSSD (`sssd.conf`).
    - Sets SSH configurations.
  - **Direct AD Integration**:
    - Joins the server to the AD domain.
    - Configures SSSD with custom settings.
    - Manages realm permitted logins and sudoers files.

### Users Role

- **Tasks**:
  - Manages user accounts based on the AD integration method.
  - Configures sudo privileges for infrastructure and resource owner teams.

## Variables and Configuration

### Global Variables

- **Environment Variables**:
  - `aap_environment`: Determines whether to use `aap_prod.yml` or `aap_test.yml`.
- **Controller Variables** (in `vars/aap_prod.yml` and `vars/aap_test.yml`):
  - `controller_host`
  - `controller_username`
  - `controller_password`
  - `validate_certs`

### Role-Specific Variables

- **Auth Role** (in `roles/auth/defaults/main.yml`):
  - `auth_ad_integration_user`: AD user for domain joining.
  - `auth_ad_integration_password_dev`
  - `auth_ad_integration_password_test`
  - `auth_ad_integration_password_prod`
- **Packages Role** (in `roles/packages/defaults/main.yml`):
  - `cockpit_packages`: List of Cockpit packages to install.
  - `cockpit_enabled`
  - `cockpit_started`
  - `cockpit_port`
  - `cockpit_manage_firewall`
- **Firewall Role** (in `roles/firewall/defaults/main.yml`):
  - `firewall`: Define firewall rules as per RHEL System Roles documentation.
- **Users Role** (in `roles/users/defaults/main.yml`):
  - `domain_suffix_dev`, `domain_suffix_test`, `domain_suffix_prod`: AD domain suffixes.
  - `infra_accounts`: List of infrastructure accounts to manage.
  - `teamxyz_accounts`: Accounts for specific resource owner teams.

## Important Files and Directories

- **Playbook**:
  - `site.yml`: The main playbook file.
- **Variables**:
  - `vars/aap_prod.yml` and `vars/aap_test.yml`: Environment-specific variables.
- **Roles**:
  - Each role is located under `roles/` and contains tasks, handlers, defaults, and files specific to its function.
- **Collections Requirements**:
  - `collections/requirements.yml`: Specifies the Ansible collections required.
- **Templates and Files**:
  - Configuration files and templates are located under each role's `files/` directory.

## Handlers

- **Restart Services**:
  - `Restart sshd`: Restarts the SSH daemon when configuration changes.
  - `Restart sssd`: Restarts the SSSD service after configuration changes.
  - `Restart dnsmasq`: Restarts the dnsmasq service when networking configurations change.
  - `Restart NetworkManager`: Restarts NetworkManager after modifying its configuration.

## Customization

- **Adjust Variables**: Modify variables in `defaults/main.yml` and `vars/` to suit your environment.
- **Modify Roles**: You can add or remove tasks within roles as needed.
- **Add Custom Roles**: Incorporate additional roles if required for your setup.
- **Firewall Rules**: Define custom firewall rules in `roles/firewall/defaults/main.yml` following the RHEL System Roles documentation.

## Notes and Best Practices

- **Sensitive Data**: Do not hardcode sensitive information like passwords. Use Ansible Vault or AAP/AWX credentials.
- **Testing**: Always test the playbook in a non-production environment before deploying to production.
- **Group Names**: Properly assign group names in your inventory to control task execution flow.
- **Documentation**: Refer to the RHEL System Roles and Ansible documentation for advanced configurations.

## References

- **Ansible Documentation**: [https://docs.ansible.com/](https://docs.ansible.com/)
- **Ansible Collections**: [https://galaxy.ansible.com/](https://galaxy.ansible.com/)
- **RHEL System Roles**: [https://access.redhat.com/articles/3050101](https://access.redhat.com/articles/3050101)
- **Ansible Automation Platform**: [https://www.ansible.com/products/automation-platform](https://www.ansible.com/products/automation-platform)
