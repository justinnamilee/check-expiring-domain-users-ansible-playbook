# Variables Reference (`vars/`)

*Note: The file `vars/config.yml` is ignored, so won't get clobbered.*

This directory contains the configuration file (`config.yml`) that drives how the playbook operates. Below is a description of each variable, its expected type or format, and a brief explanation of its purpose. You should copy `config.yml.example` to `config.yml` and adjust values as needed.

---

## `config.yml` Overview

```yaml

# Name a Windows Domain Controller or Applicable Box to Run PowerShell Script Against
# Note: You can define multiple if you expect different results from each...
# Note: But this is not typical; you should just set this to your PDC, probably.
target_hosts: ""

# How many days before expiry to warn
password_expiry_threshold: 14

# Filter used to select which accounts are in scope (see Get-AdUser in PowerShell)
get_aduser_filter: "Enabled -eq $true -and PasswordNeverExpires -eq $false"

# Where to store logs on the control node and how many days to keep
log_dir: "/path/to/expiry/log/dir"
log_age: "7d"

log_file_owner: "ansible"
log_file_group: "ansible"
log_file_mode: "0640"

log_dir_owner: "ansible"
log_dir_group: "ansible"
log_dir_mode: "0750"

# SMTP relay here; should allow the playbook to send mail
smtp_host: "127.0.0.1"
smtp_port: 25

# Email “From:” address
email_from: "netadmin@yourdomain.org"

# E-mail branding settings
email_branding:
  company_name: "ABC"  # short name used in front of company assets
  admin_team_name: "ABC Network Administration"
  admin_person_title: "ABC Network Administrator"
  office_locations: "Lewisville or Austin"
  admin_email: "netadmin@yourdomain.org"
  company_brand: "ABC Corporation"   # full company name
  password_change_url: "https://password-change.domain.local/"

# Path to the PowerShell Jinja2 template
script_template_path: "templates/check-password-expiry.ps1.j2"

# Path to the Jinja2 templates for the e-mail bodies
expiring_template_path: "templates/password-is-expiring.html.j2"
expired_template_path: "templates/password-is-expired.html.j2"

# “Today” for naming log files; by default, pulls current date in YYYY-MM-DD
today: "{{ lookup('pipe','date +%Y-%m-%d') }}"

```

---

## Variables and Descriptions

Below is a list of every variable defined in `config.yml.example`, grouped by category.

### 1. Target Hosts & AD Filtering

* **`target_hosts`**

  * **Type:** String (YAML list is also supported)
  * **Example:**

    ```yaml

    target_hosts:
      - dc1.domain.local

    ```
  * **Description:**
    *One* (or more) Windows domain controller(s) that the playbook will run against. This is passed to the `hosts:` in the third play, so the Ansible inventory must include these names (or you can specify an IP and credentials via `inventory.yml` or extra vars, whatever).  *This should realy only include _ONE_ host, unless the hosts return different sets of users (they probably don't).*

* **`get_aduser_filter`**

  * **Type:** String
  * **Example:**

    ```yaml

    get_aduser_filter: "Enabled -eq $true -and PasswordNeverExpires -eq $false"

    ```
  * **Description:**
    A PowerShell filter expression for `Get-ADUser` to limit which accounts are in scope. Common filters include checking that the account is enabled and that the password is not set to “never expire.” You can modify this to further narrow scope (e.g., only people in a specific OU).

* **`password_expiry_threshold`**

  * **Type:** Integer
  * **Default Example:**

    ```yaml

    password_expiry_threshold: 14

    ```
  * **Description:**
    Number of days before password expiry at which to send an alert. For example, if set to `14`, any account with 14 or fewer days remaining until expiration will trigger an “expiring soon” email. If `DaysToExpire <= 0`, an “expired” email is sent instead.

---

### 2. Logging & Retention

* **`log_dir`**

  * **Type:** String (absolute path)
  * **Example:**

    ```yaml

    log_dir: "/var/log/ad-password-expiry"

    ```
  * **Description:**
    The directory on the control node where daily log files will be stored. The playbook will attempt to create this directory if it does not already exist, using the specified owner, group, and mode.

* **`log_age`**

  * **Type:** String (duration)
  * **Format:** `<number><unit>` where unit is one of `s` (seconds), `m` (minutes), `h` (hours), `d` (days), or `w` (weeks).
  * **Examples:**

    ```yaml

    log_age: "7d"
    log_age: "24h"

    ```
  * **Description:**
    Any log files older than this threshold (relative to “now”) will be purged before creating/appending the day’s log. Common values are `7d` (one week) or `30d` (one month).

* **`log_file_owner`**, **`log_file_group`**, **`log_file_mode`**

  * **Types:**

    * `log_file_owner`: String (username)
    * `log_file_group`: String (group name)
    * `log_file_mode`: String (UNIX file mode, e.g., “0640”)
  * **Example:**

    ```yaml

    log_file_owner: "ansible"
    log_file_group: "ansible"
    log_file_mode: "0640"

    ```
  * **Description:**
    When the playbook “touches” or creates today’s log file (`password-expiry-<YYYY-MM-DD>.log`), it applies these ownership and permission settings. Adjust if you need different permissions or a different user/group.

* **`log_dir_owner`**, **`log_dir_group`**, **`log_dir_mode`**

  * **Types:**

    * `log_dir_owner`: String
    * `log_dir_group`: String
    * `log_dir_mode`: String (e.g., “0750”)
  * **Example:**

    ```yaml

    log_dir_owner: "ansible"
    log_dir_group: "ansible"
    log_dir_mode: "0750"

    ```
  * **Description:**
    When the playbook creates (or ensures existence of) the `log_dir`, it assigns this owner, group, and mode. Make sure the Ansible control user can create directories under the chosen parent path, or adjust owner/group appropriately.

---

### 3. SMTP & Email Settings

* **`smtp_host`**

  * **Type:** String (hostname or IP)
  * **Example:**

    ```yaml

    smtp_host: "smtp.company.local"

    ```
  * **Description:**
    The SMTP relay (or server) that the playbook will use to send password‐expiry notifications. Must be reachable from the Ansible control node. If your server requires authentication or TLS, you would need to extend the `mail:` task in `send-email-if-required.yml`.

* **`smtp_port`**

  * **Type:** Integer
  * **Example:**

    ```yaml

    smtp_port: 25

    ```
  * **Description:**
    Port on which the SMTP server listens (usually `25`, `587`, or `465`). If you’re using TLS or authentication, additional `mail:` parameters must be added (see `smpt_host` for more details on that).

* **`email_from`**

  * **Type:** String (valid email address)
  * **Example:**

    ```yaml

    email_from: "netadmin@yourdomain.org"

    ```
  * **Description:**
    The “From:” address that appears in every notification email. This should be a monitored mailbox (or alias) so that users can reply if needed.

* **`email_branding`**

  * **Type:** Dictionary, containing subkeys (all strings)
  * **Subkeys and Example Values:**

    ```yaml

    email_branding:
      company_name:       "ABC"  # short company name
      admin_team_name:    "ABC Network Administration"
      admin_person_title: "ABC Network Administrator"
      office_locations:   "Lewisville or Austin"
      admin_email:        "netadmin@yourdomain.org"
      company_brand:      "ABC Corporation"  # full company name
      password_change_url: "https://password-change.ad.domain.local/"

    ```
  * **Description:**
    These variables are used inside your Jinja2‐templated email bodies (`password-is-expiring.html.j2` and `password-is-expired.html.j2`). Note: They are passed into the subtask as `sub_email_branding`, so the placeholders in the HTML templates are:
    * `{{ sub_email_branding.company_name }}`
    * `{{ sub_email_branding.admin_team_name }}`
    * `{{ sub_email_branding.admin_person_title }}`
    * `{{ sub_email_branding.office_locations }}`
    * `{{ sub_email_branding.admin_email }}`
    * `{{ sub_email_branding.company_brand }}`
    * `{{ sub_email_branding.password_change_url }}`

  If you override the templates (see `templates/overrides/`) then make sure you use these variable names in the templates.

---

### 4. Template Paths & “Today” Variable

* **`script_template_path`**

  * **Type:** String (relative or absolute path)
  * **Example:**

    ```yaml

    script_template_path: "templates/check-password-expiry.ps1.j2"

    ```
  * **Description:**
    Path to the Jinja2‐templated PowerShell script (`*.ps1.j2`) that Ansible will render and execute on the Windows host. The resulting script should output JSON to STDOUT.  See the current PowerShell script for details on the desired output.

* **`expiring_template_path`**

  * **Type:** String (relative or absolute path)
  * **Example:**

    ```yaml

    expiring_template_path: "templates/password-is-expiring.html.j2"

    ```
  * **Description:**
    Path to the Jinja2 HTML template for “password expiring soon” emails. This template should reference `sub_email_branding` subkeys, `display_name`, `samaccountname`, `days_to_expire`, etc.  See the existing template for examples on how and where each variable is used.

* **`expired_template_path`**

  * **Type:** String (relative or absolute path)
  * **Example:**

    ```yaml

    expired_template_path: "templates/password-is-expired.html.j2"

    ```
  * **Description:**
    Path to the Jinja2 HTML template for “password has already expired” emails. Similar to the “expiring” template but with messaging for an already‐expired password.

* **`today`**

  * **Type:** String (usually templated)
  * **Default Example:**

    ```yaml

    today: "{{ lookup('pipe','date +%Y-%m-%d') }}"

    ```
  * **Description:**
    Used to build the daily log filename:

    ```
    {{ log_dir }}/password-expiry-{{ today }}.log
    ```

    By default, it runs the shell command `date +%Y-%m-%d`. You can override this if you need a different naming scheme or timezone or whatever.

---

## Tips & Notes

1. **YAML Syntax**

   * Make sure indentation is exactly two spaces per level.
   * Quoted strings are only necessary when they contain special characters (e.g., `get_aduser_filter`).
   * Numeric values (e.g., `smtp_port`) should not be quoted unless you specifically want them to be strings.

2. **Filtering AD Accounts**

   * If you need to filter by Organizational Unit (OU), you can append something like:

     ```powershell

     "Enabled -eq `$true -and PasswordNeverExpires -eq `$false -and DistinguishedName -like '*OU=Users,DC=acme,DC=local'"

     ```
   * Test your `get_aduser_filter` manually in a PowerShell session before placing it in `config.yml`.

3. **Customizing Owners/Permissions**

   * If your Ansible user can’t chown files to `ansible:ansible`, change `log_file_owner`/`log_file_group` or adjust your system so that the playbook user has write permission to `log_dir`.

4. **Advanced SMTP Options**

   * The built‐in `mail:` module supports authentication via `smtp_user`/`smtp_password` and TLS via `encrypt: starttls`. If your relay requires it, add those parameters under the `mail:` task in `tasks/send-email-if-required.yml`.  For simplicity (and because my own local relay doesn't need them) I chose to just omit them entirely.

---

With this reference in hand, you should be able to customize every aspect of the playbook’s behavior by adjusting `vars/config.yml`. Once you have `config.yml` populated and placed under `vars/`, run:

```bash

ansible-playbook -i <inventory> ../check-expiring-domain-users.yml

```

—and the playbook will validate these variables before proceeding. If any required variable is missing or improperly formatted, the playbook will fail early with a message pointing you to the culprit.
