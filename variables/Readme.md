
| Order | Variable Type | Source / Location |
| :--- | :--- | :--- |
| 1 | **command line values** | Global settings (e.g., `-u user`, `-i inventory`) |
| 2 | **role defaults** | `defaults/main.yml` inside a role |
| 3 | **inventory vars** | `vars` defined directly in inventory files/scripts |
| 4 | **inventory group_vars/all** | `group_vars/all` defined in inventory |
| 5 | **inventory group_vars/target** | `group_vars/{{ group_name }}` defined in inventory |
| 6 | **playbook group_vars/all** | `group_vars/all` located next to the playbook |
| 7 | **playbook group_vars/target** | `group_vars/{{ group_name }}` next to the playbook |
| 8 | **inventory host_vars** | `host_vars/{{ host }}` defined in inventory |
| 9 | **playbook host_vars** | `host_vars/{{ host }}` located next to the playbook |
| 10 | **host facts** | Discovered via `setup` module (e.g., `ansible_facts`) |
| 11 | **play vars** | `vars:` section defined at the play level |
| 12 | **play vars_prompt** | `vars_prompt:` section defined at the play level |
| 13 | **play vars_files** | `vars_files:` section defined at the play level |
| 14 | **role vars** | `vars/main.yml` inside a role |
| 15 | **block vars** | `vars:` section defined inside a `block` |
| 16 | **task vars** | `vars:` section defined directly on a specific task |
| 17 | **include_vars** | Loaded at runtime via the `ansible.builtin.include_vars` module |
| 18 | **set_facts / registered vars** | Created during execution via `set_fact` or `register` |
| 19 | **role params** | Variables passed to a role via the `roles:` play keyword |
| 20 | **include_role params** | Variables passed to the `ansible.builtin.include_role` module |
| 21 | **include_tasks params** | Variables passed to the `ansible.builtin.include_tasks` module |
| 22 | **extra vars** | Variables passed via the command line with `-e` or `--extra-vars` |
