# ansible-ivly

Ansible Galaxy role for deploying the ivly invoicing stack on Raspberry Pi.

## Hindsight Memory

Bank ID: `ansible-ivly`

At session start, load project context:

```text
recall("project context and recent decisions", bank_id="ansible-ivly")
```

## Publishing

- Published under namespace `head1328` (not `ivly`)
- Galaxy name: `head1328.ivly`
- Source: GitHub (mirrored from Codeberg)
- Galaxy requires GitHub, ivly itself has no GitHub presence

## Local development

```bash
make symlink   # links to ~/.ansible/roles/head1328.ivly
make lint      # ansible-lint
make test      # molecule test
```
