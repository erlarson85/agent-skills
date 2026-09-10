---
name: ansible-vault-read
description: Read secrets from HashiCorp Vault in Ansible playbooks/roles. Use whenever a task needs to fetch a secret from Vault, or when reviewing/migrating existing Ansible tasks that use the legacy hashivault_read module.
---

## Use `community.hashi_vault.vault_read`, not `hashivault_read`

The legacy `hashivault_read` module (with `url`/`token`/`secret`/`key` params) is deprecated. All
Vault reads must use the `community.hashi_vault.vault_read` module instead. Reference: PR #160
("restoring ansible") in `snitsm-servicenow-mid`, which migrated every `hashivault_read` call in
that repo to `vault_read`.

**Old pattern (do not use):**

```yaml
- name: get secret from vault
  hashivault_read:
    url: "{{ vault_addr }}"
    token: "{{ vault_token }}"
    secret: "{{ snowSecretPath }}"
    key: "username"
  register: "snow_username"
  check_mode: no
  ignore_errors: true
  diff: no
  connection: local
  become: no
```

**New pattern:**

```yaml
- name: get secret from vault
  community.hashi_vault.vault_read:
    path: "secret/{{ snowSecretPath }}"
  register: "snow_creds"
  check_mode: false
  no_log: true
  connection: local
  become: false
```

Key differences:

- One task reads the whole secret (all keys at that path) instead of one task per key. Consolidate
  multiple `hashivault_read` calls against the same secret path into a single `vault_read` task.
- No `url`/`token` params — Vault auth comes from the pipeline's `vault_callback` (e.g.
  `opsVaultLogin()` in the Jenkinsfile), not from `vault_addr`/`vault_token` extra-vars.
- `no_log: true` replaces `diff: no` — secrets must never be logged.
- Use `check_mode: false` / `become: false` (explicit booleans), not the old `no`/`yes` YAML
  shorthand.
- Drop `ignore_errors: true` — a failed secret read should fail the play, not silently continue.

## Always prefix the path with `secret/`

`community.hashi_vault.vault_read` talks to Vault's KV v2 engine, which requires the mount prefix
in the `path`. Any secret path — whether hardcoded or from a variable like `snowSecretPath` — must
resolve to a string starting with `secret/`.

- If the path is hardcoded, write it with the prefix directly: `path: "secret/services/dco/..."`.
- If the path comes from a variable that itself does NOT include the prefix (e.g.
  `snowSecretPath: 'services/dco/jenkins/snitsm/snow/nonprod/linux/dev'` in group_vars), prepend it
  at the call site: `path: "secret/{{ snowSecretPath }}"`.
- Never store `secret/` inside the variable's default AND prepend it again at the call site — pick
  one place. This repo's convention is: keep the raw path in the variable, prepend `secret/` in the
  task.

## Reading the result

`vault_read` returns the full KV v2 response under `.data.data.<key>`, not `.value` like the old
module:

```jinja2
{{ snow_creds.data.data.username }}
{{ snow_creds.data.data.pass }}
{{ slackHook.data.data.webhookURL }}
```

When migrating templates (`.j2` files) that reference old registered vars like `snow_username.value`
or `ednaDBPass.value`, update them to the new registered var name and `.data.data.<key>` form.

## Checklist when adding or migrating a Vault read task

1. Module is `community.hashi_vault.vault_read` (not `hashivault_read`).
2. `path:` is the only vault-connection param — no `url`/`token`.
3. The resolved path string starts with `secret/`.
4. One `vault_read` task per secret path, covering all keys needed from that path (don't split by
   key).
5. `no_log: true` is set.
6. `check_mode: false`, `become: false`, `connection: local` (explicit booleans, matching existing
   tasks in the role).
7. Downstream `.j2` templates and other tasks reference `<register_name>.data.data.<key>`.
8. Auth to Vault is handled by the pipeline's `vault_callback` (e.g. `opsVaultLogin()`), not by
   `vault_addr`/`vault_token` extra-vars passed into the playbook.
