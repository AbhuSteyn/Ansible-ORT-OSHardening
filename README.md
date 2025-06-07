
# Ansible Operations Readiness & OS Hardening

This repository contains two primary playbooks:

1. **Operations Readiness Report Generation:**  
   Gathers system facts, OS hardening settings, and service statuses from both Linux and Windows machines, then renders an HTML report using a custom Jinja2 template.

2. **OS Hardening Playbook:**  
   Applies best practices for OS hardening on target systems using modular, role‑based tasks. Separate tasks are provided for Linux and Windows environments.

Both playbooks follow advanced Ansible practices with idempotency, proper error handling, and tagging support.

---

## Repository Structure

```plaintext
repository-root/
├── ansible.cfg                      # Ansible configuration settings
├── inventory/
│   └── hosts.ini                    # Inventory file defining Linux and Windows groups
├── playbooks/
│   ├── generate_report.yml          # Playbook for generating the HTML operations report
│   └── os_hardening.yml             # Playbook for applying OS hardening best practices
├── roles/
│   ├── report_generator/            # Role for data collection & HTML report generation
│   │   ├── tasks/
│   │   │   └── main.yml             
│   │   ├── templates/
│   │   │   └── report_template.j2   
│   │   └── vars/
│   │       └── main.yml
│   └── os_hardening/                # Role to apply OS hardening measures to Linux/Windows hosts
│       ├── tasks/
│       │   ├── linux_hardening.yml  
│       │   └── windows_hardening.yml
│       ├── handlers/
│       │   └── main.yml            
│       └── vars/
│           └── main.yml
├── group_vars/                      # Group-specific variables (e.g., Linux and Windows settings)
└── docs/
    └── README.md                    # This documentation file with the full code examples
```

---

## Files & Code

### 1. Ansible Configuration

**ansible.cfg**  
_Configures Ansible defaults including the inventory file, disabling host key checking, and specifying the roles path._

```ini
[defaults]
inventory = inventory/hosts.ini
host_key_checking = False
retry_files_enabled = False
roles_path = roles
```

---

### 2. Inventory File

**inventory/hosts.ini**  
_Defines your Linux and Windows hosts. Adjust the hostnames/address as needed, and ensure WinRM is set up for Windows targets._

```ini
[linux]
linux1.example.com
linux2.example.com

[windows]
windows1.example.com
windows2.example.com
```

---

### 3. Playbooks

#### a. Report Generation Playbook

**playbooks/generate_report.yml**  
_This playbook gathers system information and then uses the `report_generator` role to render an HTML report via a Jinja2 template._

```yaml
---
- name: Generate Operations Readiness Report
  hosts: all
  gather_facts: yes
  roles:
    - report_generator
```

#### b. OS Hardening Playbook

**playbooks/os_hardening.yml**  
_This playbook applies security hardening measures through the `os_hardening` role. It is targeted to both Linux and Windows hosts._

```yaml
---
- name: Apply OS Hardening Measures
  hosts: all
  gather_facts: yes
  roles:
    - os_hardening
  tags: os_hardening
```

---

### 4. Roles and Their Code

#### a. Report Generator Role

**roles/report_generator/tasks/main.yml**  
_This file gathers OS hardening settings and service statuses from Linux or Windows hosts, sets a fact for rendering the report, and then calls a Jinja2 template to create an HTML report._

```yaml
---
# roles/report_generator/tasks/main.yml

- name: Gather OS Hardening Settings
  debug:
    msg: "Checking OS Hardening settings on {{ inventory_hostname }}"

- name: Gather Service Status on Linux
  when: ansible_os_family == "Linux"
  shell: systemctl status sshd
  register: sshd_status
  changed_when: false

- name: Gather Service Status on Windows
  when: ansible_os_family == "Windows"
  win_command: PowerShell -Command "Get-Service -Name 'WinRM'"
  register: winrm_status
  changed_when: false

- name: Set fact report_data for Linux
  when: ansible_os_family == "Linux"
  set_fact:
    report_data:
      hostname: "{{ inventory_hostname }}"
      os_family: "{{ ansible_os_family }}"
      os_distribution: "{{ ansible_distribution | default('N/A') }}"
      service_status: "{{ sshd_status.stdout | default('N/A') }}"

- name: Set fact report_data for Windows
  when: ansible_os_family == "Windows"
  set_fact:
    report_data:
      hostname: "{{ inventory_hostname }}"
      os_family: "{{ ansible_os_family }}"
      os_distribution: "{{ ansible_distribution | default('N/A') }}"
      service_status: "{{ winrm_status.stdout | default('N/A') }}"

- name: Render HTML Report from Template
  template:
    src: report_template.j2
    dest: "/tmp/report_{{ inventory_hostname }}.html"
```

**roles/report_generator/templates/report_template.j2**  
_This Jinja2 template defines the structure and styling of the generated HTML report._

```html
<!DOCTYPE html>
<html>
<head>
    <title>Operations Readiness Report for {{ report_data.hostname }}</title>
    <style>
        body { font-family: Arial, sans-serif; }
        h1 { color: #0275d8; }
        table { width: 100%; border-collapse: collapse; }
        th, td { border: 1px solid #ddd; padding: 8px; }
        th { background-color: #f2f2f2; }
    </style>
</head>
<body>
    <h1>Operations Readiness Report</h1>
    <h2>Host: {{ report_data.hostname }}</h2>
    <table>
        <tr>
            <th>OS Family</th>
            <td>{{ report_data.os_family }}</td>
        </tr>
        <tr>
            <th>OS Distribution</th>
            <td>{{ report_data.os_distribution }}</td>
        </tr>
        <tr>
            <th>Service Status</th>
            <td>{{ report_data.service_status }}</td>
        </tr>
    </table>
</body>
</html>
```

**roles/report_generator/vars/main.yml**  
_Define any report-specific default variables (if required)._

```yaml
---
# roles/report_generator/vars/main.yml

# Default output directory for reports (overridable via group_vars)
report_output_dir: "/tmp"
```

---

#### b. OS Hardening Role

This role applies best practices for OS hardening across Linux and Windows systems using OS-specific tasks.

**roles/os_hardening/tasks/linux_hardening.yml**  
_Linux-specific hardening tasks including firewall configuration, SSH hardening, and disabling unused services._

```yaml
---
# roles/os_hardening/tasks/linux_hardening.yml

- name: Ensure UFW (firewall) is installed (Debian-based)
  become: yes
  apt:
    name: ufw
    state: present
  when: ansible_os_family == "Debian"

- name: Enable UFW firewall (Debian-based)
  become: yes
  ufw:
    state: enabled
  when: ansible_os_family == "Debian"

- name: Disable Telnet service if installed
  become: yes
  service:
    name: telnet
    state: stopped
    enabled: false
  ignore_errors: yes

- name: Secure SSH - Disallow root login
  become: yes
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PermitRootLogin'
    line: 'PermitRootLogin no'
  notify: Restart SSH
```

**roles/os_hardening/tasks/windows_hardening.yml**  
_Windows-specific hardening tasks including disabling the Guest account, disabling SMBv1, enforcing a password policy, and ensuring the firewall is enabled._

```yaml
---
# roles/os_hardening/tasks/windows_hardening.yml

- name: Disable the Guest account
  win_user:
    name: Guest
    state: absent

- name: Disable SMBv1 protocol
  win_shell: |
    Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters" -Name "SMB1" -Value 0
  become: yes

- name: Enforce Password Policy
  win_password_policy:
    minimum_password_length: 12
    password_complexity: true

- name: Ensure Windows Firewall is enabled
  win_firewall:
    state: enabled
```

**roles/os_hardening/handlers/main.yml**  
_Defines handlers for alerting or restarting services after configuration changes._

```yaml
---
# roles/os_hardening/handlers/main.yml

- name: Restart SSH
  become: yes
  service:
    name: sshd
    state: restarted
```

**roles/os_hardening/vars/main.yml**  
_Default variables for OS hardening (customize as needed)._

```yaml
---
# roles/os_hardening/vars/main.yml

# Example hardening variables
firewall_enabled: true

password_policy:
  minimum_length: 12
  complexity: true
```

---

## Usage

1. **Configure Your Inventory:**  
   Update the `inventory/hosts.ini` file with the addresses of your Linux and Windows hosts. Ensure that SSH keys (for Linux) and WinRM (for Windows) are properly configured.

2. **Customize Variables:**  
   Edit the variable files under `group_vars/` or within the roles (in `vars/`) to tailor settings (such as report paths or hardening parameters) to your security policies.

3. **Generate the Operations Readiness Report:**  
   Run the following command:
   ```bash
   ansible-playbook -i inventory/hosts.ini playbooks/generate_report.yml
   ```
   The Jinja2 template will render an HTML report for each host under `/tmp/` (or your defined `report_output_dir`).

4. **Apply OS Hardening Measures:**  
   Execute the OS hardening playbook:
   ```bash
   ansible-playbook -i inventory/hosts.ini playbooks/os_hardening.yml --tags "os_hardening"
   ```
   Use tags (e.g., `--tags "firewall,ssh"`) for selective execution if needed.

---

## Best Practices & Next Steps

- **Role-Based Organization:**  
  Keeping reporting and hardening in separate roles makes the playbooks modular and easier to maintain or extend.

- **Idempotency & Error Handling:**  
  Tasks are designed to be idempotent, and error handling (using `ignore_errors`, conditionals, and handlers) ensures smooth execution even when some tasks are non-critical.

- **Customizable Reporting:**  
  The Jinja2 template can be extended to include more data such as compliance status and detailed logs. Consider integrating with centralized logging or dashboards.

- **Continuous Improvement:**  
  Integrate these playbooks into your CI/CD pipelines and run Molecule tests to validate changes before production rollout.

