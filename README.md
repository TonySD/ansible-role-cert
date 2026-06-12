# ansible-role-cert

One role, three modes, one stable output contract. Whatever the mode, after the
role runs you get two facts:

- `cert_cert_path` — path to the certificate (fullchain in LE mode)
- `cert_key_path` — path to the private key

A consumer (nginx, haproxy, ...) only ever references those two facts. That is
the whole point: the consumer does not care how the cert was produced.

## Install

The role lives in its own repo and is consumed via `requirements.yml` — don't copy it into your project.

In your Ansible project, create (or extend) `requirements.yml`:

```yaml
---
roles:
  - name: cert
    src: git+https://github.com/TonySD/ansible-role-cert.git
    version: main          # pin to a tag once released, e.g. v1.0.0
collections:
  - name: community.crypto # required for self_signed mode
```

Make sure your `ansible.cfg` points to a local roles path:

```ini
[defaults]
roles_path = roles
collections_path = collections
```

Install both the role and its collection dependency:

```bash
ansible-galaxy install -r requirements.yml
```

Add the install targets to `.gitignore` (don't vendor them):

```
roles/
collections/
```

Use it in a play — referenced simply as `cert`:

```yaml
- hosts: web
  roles:
    - role: cert
      vars:
        cert_mode: self_signed
        cert_domains: ["app.example.com"]
```

### Updating

Bump `version:` in `requirements.yml`, then force a re-install (Galaxy won't
overwrite an existing role otherwise):

```bash
ansible-galaxy install -r requirements.yml --force
```

### Quick one-off install (no requirements file)

```bash
ansible-galaxy role install git+https://github.com/TonySD/ansible-role-cert.git
```

Note: this names the directory after the repo (`ansible-role-cert`), so reference
it as `- role: ansible-role-cert`. Use the `requirements.yml` method above to get
the shorter `cert` name.

## Modes

### self_signed
```yaml
- hosts: web
  roles:
    - role: certificate
      vars:
        cert_mode: self_signed
        cert_domains: ["app.internal.lan"]
```

### static (existing files / vault)
```yaml
- role: certificate
  vars:
    cert_mode: static
    cert_name: app
    cert_static_cert_src: "files/app.crt"
    cert_static_key_src: "files/app.key"
    # or inline:
    # cert_static_cert_content: "{{ vault_app_cert }}"
    # cert_static_key_content:  "{{ vault_app_key }}"
```

### letsencrypt (certbot, webroot HTTP-01)
```yaml
- role: certificate
  vars:
    cert_mode: letsencrypt
    cert_domains: ["app.example.com", "www.app.example.com"]
    cert_le_email: "admin@example.com"
    cert_le_webroot: /var/www/letsencrypt
    cert_le_staging: true            # flip to false once it works
    cert_le_deploy_hook: "systemctl reload nginx"
```

## Ordering (the part that usually breaks)

Webroot HTTP-01 needs a web server already answering on **:80** at
`/.well-known/acme-challenge/` pointing to `cert_le_webroot` **before** certbot
runs. This is a chicken-and-egg problem if the same nginx vhost also references
a cert that does not exist yet. Resolve it by splitting the play:

1. Configure nginx with an **HTTP-only** vhost that serves `cert_le_webroot`.
2. Run this role in `letsencrypt` mode.
3. Add the HTTPS vhost (using `cert_cert_path` / `cert_key_path`) and reload.

The shared `cert_le_webroot` variable is the only coupling between the two roles.

## Idempotency notes

- LE issuance is guarded by `creates:` on `live/<name>/fullchain.pem`, so the
  task runs **once**. Renewals are handled by `certbot.timer`, not by re-running
  the play. The `--cert-name` flag keeps the live/ dir stable (no `-0001` dirs).
- Adding a domain to an existing LE cert will NOT re-issue automatically
  (the cert already exists). Delete the cert or add `--expand` manually.
- self_signed uses `ignore_timestamps` semantics, so a relative `not_after`
  does not trigger regeneration every run.

## Requirements

```bash
ansible-galaxy collection install community.crypto   # self_signed mode
```
