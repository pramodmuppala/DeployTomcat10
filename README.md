# DeployTomcat10

An Ansible-based project to generate, validate, and deploy **Tomcat 10 configuration** for a local Tomcat instance, then restart the instance.

This playbook currently:
- Generates `setenv.sh` from a Jinja2 template
- Validates `setenv.sh` syntax and a policy rule for `CATALINA_OPTS`
- Generates `server.xml` from a Jinja2 template
- Validates `server.xml` using `xmllint`
- Runs Tomcat `configtest` using `catalina.sh configtest`
- Copies validated files into the Tomcat instance
- Shuts down and starts up the Tomcat instance

## Project Structure

```text
DeployTomcat10/
├── Deploy.yml                 # Main playbook
├── ansible.cfg                # Ansible config (currently mostly default comments)
├── hosts                      # Inventory
├── vars/
│   └── main.yml               # Tomcat paths and runtime config values
└── roles/
    ├── create_setenv_script/  # Render + validate staged setenv.sh
    ├── validate_setenv_script/# Additional setenv validation role
    ├── create_server_xml/     # Render/validate/deploy server.xml + Tomcat configtest
    ├── copy_config_files/     # Copies staged setenv.sh into CATALINA_BASE/bin
    ├── shutdown/              # Runs shutdown.sh
    ├── startup/               # Runs startup.sh
    └── healthcheck/           # Present but not wired into main playbook
```

## Requirements

- **Ansible** (ansible-core)
- **Tomcat 10** installed locally
- **Java/JDK** installed locally
- `xmllint` available on the system (used to validate `server.xml`)
- Permission to write to:
  - `{{ tomcat.catalina_base }}/bin`
  - `{{ tomcat.catalina_base }}/conf`
  - staging paths under `/opt/temp/...`

> The playbook uses `connection: local` and `become: yes`, so it is intended to run on the same host where Tomcat is installed.

## Configurable Variables

Edit `vars/main.yml` to match your environment.

Key variables include:
- `tomcat.catalina_base`
- `tomcat.catalina_home`
- `tomcat.java_home`
- `tomcat.catalina_opts`
- `tomcat.connectors` (HTTP connector definitions)
- `tomcat.hosts` (Tomcat `<Host>` entries)
- `staging_dir`
- `validates_base`

### Current defaults (example)

- `CATALINA_BASE`: `/opt/tomcat/app1`
- `CATALINA_HOME`: `/opt/products/tomcat/tomcat10`
- `JAVA_HOME`: `/opt/products/jdk/jdk25`
- HTTP port: `8080`

## Inventory

The current inventory file (`hosts`) contains a local target group:

```ini
[Tomcat]
locahost ansible_connection=local
```

### Important note

`locahost` appears to be a typo and should usually be `localhost`:

```ini
[Tomcat]
localhost ansible_connection=local
```

Because the playbook also sets `connection: local`, it may still run depending on your environment, but correcting the inventory entry is recommended.

## How It Works

The main playbook is `Deploy.yml` and runs the following roles in order:

1. `create_setenv_script`
2. `validate_setenv_script`
3. `create_server_xml`
4. `validate_server_xml` *(referenced in playbook; role is currently missing from the repository)*
5. `copy_config_files`
6. `shutdown`
7. `startup`

### Important note about missing role

`Deploy.yml` references a role named `validate_server_xml`, but that role directory does **not** exist in the current project tree.

You have two options:
- **Remove** `validate_server_xml` from `Deploy.yml` (since `create_server_xml` already validates XML and runs `configtest`), or
- **Add** a `roles/validate_server_xml/` role if you want validation separated into its own role.

## Running the Playbook

From the project root:

```bash
ansible-playbook -i hosts Deploy.yml
```

If privilege escalation requires a password:

```bash
ansible-playbook -i hosts Deploy.yml --ask-become-pass
```

## Validation and Safety Checks Implemented

### `setenv.sh` validation

The project validates the generated `setenv.sh` in staging using:
- `bash -n` (shell syntax check)
- A policy grep to ensure `CATALINA_OPTS` is **not overwritten directly** with only memory/JVM flags

The template currently appends to `CATALINA_OPTS` safely:

```bash
export CATALINA_OPTS="$CATALINA_OPTS ..."
```

### `server.xml` validation

The project validates generated `server.xml` via:
- `xmllint --noout`
- `catalina.sh configtest` with `CATALINA_BASE`, `CATALINA_HOME`, and `JAVA_HOME` exported in the task environment

## Backups Created

Before deploying updated files, the project attempts to back up:
- `{{ tomcat.catalina_base }}/bin/setenv.sh` → `setenv.sh.bak` (if present)
- `{{ tomcat.catalina_base }}/conf/server.xml` → `server.xml.bak` (if present)

## Troubleshooting

### 1) `validate_server_xml` role not found
**Error:** Ansible reports missing role `validate_server_xml`

**Fix:** Remove it from `Deploy.yml` or create the missing role.

### 2) `xmllint` command not found
Install XML utilities package for your OS (e.g., `libxml2-utils` on Debian/Ubuntu).

### 3) Permission denied writing to Tomcat directories
Ensure the user can run with `become: yes` and has sudo privileges.

### 4) Tomcat `configtest` fails
Verify:
- `tomcat.catalina_home` is correct
- `tomcat.catalina_base` is correct
- `tomcat.java_home` points to a valid JDK/JRE
- Generated `server.xml` matches your Tomcat version/modules

## Suggested Improvements

- Add the missing `validate_server_xml` role or remove it from the playbook
- Create staging directories explicitly before rendering templates
- Add a real `healthcheck` role step after startup using `tomcat.health_url`
- Fix inventory typo (`locahost` → `localhost`)
- Add tags (`config`, `validate`, `restart`) for selective execution
- Add idempotent service management using `systemd` when Tomcat runs as a service

## License

Add your preferred license information here.