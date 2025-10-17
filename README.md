# Ansible Automation Platform configuration with CI/CD

This repository contains playbooks and automation to:
- Create a GitHub repository from a template and integrate it with Ansible Automation Platform (AAP).
- Create AAP SCM credentials and projects.
- Create or update GitHub webhooks to trigger Event Driven Ansible flows.
- Run CI checks (ansible-lint, yamllint, molecule) and sign content.

## Contents (important paths)
- ansible/init-git-repo.yml        — create GitHub repo, mirror content, create AAP resources, create webhook
- ansible/deploy_aap_config.yml    — deploy CaC exports to the controller
- ansible/run_ansible_lint.yml     — run ansible-lint
- ansible/run_molecule_tests.yml   — run molecule tests
- ansible/vars/main.yml            — main variables to edit for your environment
- collections/requirements.yml     — required Ansible collections
- .ansible/collections/requirements.txt — python libs used by collections (when present)

## Prerequisites
- Ansible 2.14+ (or supported version for infra.aap_configuration collection)
- ansible-galaxy collections installed:
  ansible-galaxy collection install -r collections/requirements.yml
- Python requirements for collections:
  pip install -r .ansible/collections/requirements.txt
- GitHub token with repo and admin:repo_hook scopes (do not commit token)
- Access to an AAP controller with an API user (or token)
- (Optional) Slack token if notifications are desired

## Variables to configure
Edit `ansible/vars/main.yml` or provide values via an encrypted vault file / extra-vars.

Required variables (examples shown):
- github_user: "my-github-user-or-org"
- source_repo_url: "https://github.com/my-org/template-repo.git"
- new_repo_name: "new-repo-name"
- new_repo_description: "Repository created by Ansible for AAP"
- webhook_url: "https://gateway.example.com/webhooks/user"         # where GitHub will send event payloads
- webhook_key: "super-secret-webhook-signing-key"                  # store in vault

AAP connection and resources (pass via extra-vars or vault):
- controller_host: "aap.example.com"
- controller_username: "automation_admin" or use controller_token
- controller_password: "REDACTED"
- controller_validate_certs: false
- aap_organization: "CustomerOrg"
- aap_inventory_name: "Production"      # inventory to use in AAP
- aap_playbook_name: "onboard_user.yml" # optional

Optional:
- github_token: "GITHUB_PERSONAL_ACCESS_TOKEN"  # required to create repos/webhooks
- slack_token, slack_channel                        # for Slack notifications

Recommended: place secrets in an ansible-vault file (vault.yml) and pass with `-e @vault.yml`.

Example minimal vault.yml (do not commit):
```yaml
github_token: "ghp_xxx"
webhook_key: "super-secret"
controller_password: "AAP_password"
slack_token: "xoxb-xxx"
```

## How to run
1. Validate or install collections and python deps:
```bash
ansible-galaxy collection install -r collections/requirements.yml
pip install -r .ansible/collections/requirements.txt
```

2. Dry-run (syntax/check):
```bash
ansible-playbook ansible/init-git-repo.yml --check -e @ansible/vars/main.yml -e @vault.yml
```

3. Execute creation (real run):
```bash
ansible-playbook ansible/init-git-repo.yml -e @ansible/vars/main.yml -e @vault.yml
```

Pass controller connection variables either in vault.yml or as extra-vars:
```bash
-e controller_host="aap.example.com" -e controller_username="admin"
```

## GitHub webhook notes
- webhook_url must be reachable by GitHub (public or via tunnel).
- webhook_key is used to validate signatures on incoming requests.
- GitHub token must have permissions to create webhooks for the target repository.

## AAP / infra.aap_configuration notes
- This repo uses the supported infra.aap_configuration collection to create AAP resources as code.
- Prefer token-based controller auth where possible.
- If TLS validation fails during development, set `controller_validate_certs: false`.

## CI / Tests
- To run linting and tests, see the playbooks in `ansible/`:
  - ansible/run_ansible_lint.yml
  - ansible/run_yamllint.yml
  - ansible/run_molecule_tests.yml
- Example run for molecule (depends on scenario):
```bash
molecule test -s default
```

## Troubleshooting
- Increase verbosity: add `-v` or `-vvv` to ansible-playbook.
- Check controller API connectivity and user permissions.
- Verify GitHub token scopes and rate limits.

## Support
For changes to variables or to generate a vault template with specific placeholders, run:
```bash
ansible-playbook ansible/init-git-repo.yml --tags prepare-vault -e @ansible/vars/main.yml
```
(Adjust or create a playbook task that creates a sample vault if you need it.)

