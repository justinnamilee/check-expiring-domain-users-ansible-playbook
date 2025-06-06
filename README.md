# Domain Password Expiry Notifier

A small Ansible‐based project to check Active Directory user password expiry on a Windows domain controller and notify users via email if their passwords are about to expire (or have already expired). This playbook runs remotely against one or more domain controllers, gathers password‐expiry information via a PowerShell script, and sends templated HTML emails to affected users.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Prerequisites](#prerequisites)
3. [Installation](#installation)
4. [Configuration](#configuration)
5. [Usage](#usage)
6. [Directory Structure](#directory-structure)
7. [How It Works](#how-it-works)
8. [License](#license)

---

## Project Overview

Many organizations enforce password‐expiration policies for Active Directory users. Rather than waiting until accounts lock out, this playbook:

1. Connects to one or more Windows domain controllers (via WinRM).
2. Runs a PowerShell snippet (templated by Jinja2) to list all AD users whose passwords will expire within a configurable threshold.
3. Parses the JSON output from PowerShell.
4. Loops over each user account, checks if an email address exists, and sends a warning (or expired) email using a templated HTML body.
5. Writes a daily log file (named `password-expiry-YYYY-MM-DD.log`) under a configurable directory and purges old log files beyond a configured retention period.

---

## Prerequisites

- **Ansible Control Node**
  - Ansible 2.9+ (tested with 2.9.x and 2.10.x)
  - Python 3.6+ (with Jinja2, which ships with Ansible)
  - `pywinrm` & `xmltodict` (if you’ll be using WinRM for Windows targets):
    ```bash
    pip install "pywinrm>=0.4.2" "xmltodict>=0.12.0"
    ```
- **Windows Domain Controller or Server**
  - PowerShell 5.1 (or higher)
  - WinRM enabled & configured to accept remote commands
  - The account used by Ansible must have permission to run `Get-ADUser` and query AD password info.
  - The accounts fetched by the PowerShell script should have their "E-mail" field set.

- **SMTP Relay**
  - A reachable SMTP relay (or a server, but better to have a local relay) that allows unauthenticated (or authenticated, if you modify the playbook) email sending.

---

## Installation

1. **Clone this repository**
   ```bash

   git clone https://github.com/your-org/ansible-ad-password‐expiry.git
   cd ansible-ad-password‐expiry

   ```

2. **Install Python dependencies** (on your Ansible control node)

   ```bash

   pip install ansible pywinrm xmltodict

   ```

3. **Ensure WinRM connectivity**

   * Test your WinRM connection from the control node to the domain controller:

     ```bash

     ansible -i inventory.yml YOUR_HOST -m win_ping

     ```
   * (Adjust `inventory.yml` to point at your DC and supply the correct credentials.)

---

## Configuration

1. **`vars/config.yml`** (example template)

   * Create a copy of `vars/config.yml.example` (or simply add your own `vars/config.yml`) and fill in all required values.

   > **Note:** A detailed explanation of all variables can be found in [`vars/README.md`](vars/README.md).

2. **Templates**

   * **PowerShell Script Template** (`templates/check-password-expiry.ps1.j2`)
     Contains the logic to run `Get-ADUser`, calculate days until password expiry, and emit JSON.
   * **Email Body Templates** (`templates/password-is-expiring.html.j2` & `templates/password-is-expired.html.j2`)
     Provide an HTML structure for “expiring soon” and “already expired” emails. You can customize these to match your corporate branding.

---

## Usage

1. **Run the Playbook**
   From the project root:

   ```bash

   ansible-playbook check-expiring-domain-users.yml

   ```

   * If you’re not using an inventory file, you can pass `-l <hostname>` or use `--extra-vars "ansible_host=DC1 ansible_user=Admin ansible_password='TopSecret'"`

2. **Check Logs**

   * By default, a new log file named `password-expiry-<YYYY-MM-DD>.log` is created under the directory specified by `log_dir`.
   * If any users were emailed (or if there were errors), you’ll see lines like:

     ```

     2025-06-06 14:23:45 INFO:  John Doe (jdoe) expires in 5 days — email sent successfully to jdoe@domain.com
     2025-06-06 14:23:45 WARNING: Jane Smith (jsmith) expires in 20 days — no notice needed

     ```

3. **Cron / Automation (Optional)**

   * Schedule this playbook to run daily (e.g. via a CI/CD pipeline or a simple cron job).
   * Example crontab entry (runs at 8 AM every day):

     ```

     0 8 * * * cd /path/to/ansible-ad-password-expiry && \ # this assumes you want to capture the raw ansible output
       ansible-playbook -i inventory.yml check-expiring-domain-users.yml >> /var/log/ansible/expiry-notifier.log 2>&1

     ```

---

## Directory Structure

```
playbook/
├── check-expiring-domain-users.yml      # Main playbook
├── README.md                            # ← You are here
├── templates/
│   ├── overrides/                       # Ignored directory, put your own templates here
│   ├── check-password-expiry.ps1.j2     # PowerShell script template
│   ├── password-is-expiring.html.j2     # “Password expiring soon” email template
│   └── password-is-expired.html.j2      # “Password expired” email template
├── tasks/
│   └── send-email-if-required.yml       # Included tasks to send mail & log results
└── vars/
    ├── config.yml.example               # Example configuration (copy to config.yml)
    └── README.md                        # Detailed variable explanations
```

---

## How It Works

1. **Validation & Setup**

   * The first play (on `localhost`) ensures that `vars/config.yml` exists, that required variables are defined and valid (e.g. positive integers, valid email format), and that template files (`script_template_path`, `expiring_template_path`, `expired_template_path`) actually exist.
   * It also creates (or ensures existence of) the log directory and touches a daily log file named `password-expiry-<today>.log`. Any old logs older than `log_age` are purged.

2. **Gathering AD Data**

   * The second play targets your domain controller(s).
   * It runs a PowerShell snippet (using `win_shell` + `lookup('template', ...)`) that:

     * Queries all user accounts matching `get_aduser_filter`.
     * Calculates days until password expiry for each user.
     * Outputs a JSON array of objects, each containing at least:

       ```json
       {
         "DisplayName": "John Doe",
         "SamAccountName": "jdoe",
         "EmailAddress": "jdoe@domain.com",
         "DaysToExpire": 5
       }
       ```

3. **Processing & Notification**

   * Back on the control node, Ansible parses that JSON into the `expiring_user_list` variable.
   * It loops over each user object and:

     * Checks if `DaysToExpire` is less than or equal to `password_expiry_threshold`.
     * Verifies there’s an email address on file.
     * Chooses either the “expiring soon” or “already expired” HTML template.
     * Sends an email via Ansible’s `mail` module using the configured SMTP server.
     * Logs the outcome (success/failure/no-email/no-notice) to the daily log file.

---

## License

This project is licensed under the [GNU General Public License v3.0](https://www.gnu.org/licenses/gpl-3.0.en.html) (GPL-3.0). See the full text in the [LICENSE](LICENSE) file.
