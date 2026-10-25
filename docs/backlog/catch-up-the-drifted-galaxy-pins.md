---
kind: change
waits_on: []
design_with: [cover-the-unwatched-manifests-and-settle-the-watcher]
re_checked: 2026-10-19
---

`ansible/requirements.yml` pins `hetzner.hcloud` 7.0.0 and `geerlingguy.docker` 8.0.0 (current on 2026-09-08). Three collections are at least a major version behind: `community.general` 9.5.0 (13.4.0 then), `ansible.posix` 1.6.2 (2.2.2), `community.docker` 4.1.0 (5.3.0). Dependabot has no `ansible-galaxy` ecosystem, so nothing proposes bumps here. They are used by `hardening` (`community.general.ufw`), `deploy_user` (`ansible.posix.authorized_key`'s `key_options`, `community.docker.docker_login`), `swap` (`ansible.posix.mount`, `ansible.posix.sysctl`) and the dynamic inventory. Molecule is the gate: `molecule test --all` per role, reading the SCENARIO RECAP rather than the exit code. `ansible-core` (pinned in `ansible/requirements-test.txt` and `ansible/requirements.txt`) and these collections were verified together as one set.

- **Kind**: A four-major migration across several collections, too large for one review sitting.
- **Design with**: `cover-the-unwatched-manifests-and-settle-the-watcher`, since `ansible-core` and the collections are a matched set and a bot bump landing mid-migration would confuse which half broke.
- **Recorded by**: an inventory taken while archiving `open-autoupdate-pr-with-app-token` (2026-09-08); a migration was not a rider on that archive.
