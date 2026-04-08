### Ansible Vault commands
you can set the DEFAULT_VAULT_PASSWORD_FILE in your ansible.cfg 

#### Encrypt variable
```
ansible-vault encrypt_string -ask-vault-pass -name "ansible_become_pass' 'password'
```
#### Encrypt variable file 
```
 ansible-vault encrypt external_vault_vars.yaml
```
#### Run playbook using encrypted variable file added to var_files in playbook
```
ansible-playbook --ask-vault-pass playbook.yaml
```
#### Decrypt variable file
```
ansible-vault decrypt external_vault_vars.yaml
```
#### Password rotation ```rekey``` will change the vault password to a new one
```
ansible-vault rekey external_vault_vars.yaml
```
#### View an ecrypted file
```
ansible-vault view external_vault_vars.yaml
```

#### Read the password for a file 
`--vault-password-file password_file` 
This is the legacy method. 
It simply points to a file containing a single global password. 
It doesn't support labels, so if you have multiple vaults with different passwords.
```
ansible-vault view --vault-password-file password_file.txt external_vault_vars.yaml
```
#### Prompt for a password
`--vault-id @password_file:`
This uses the modern "Vault ID" system. 
The @ symbol tells Ansible to read the password from a file. 
This method allows you to "label" passwords (e.g., dev@file or prod@file), 
making it easier to manage multiple encrypted files with different passwords in the same project.

-   
```
ansible-vault view -vault-id @prompt external_vault_vars. yaml
```

-   
```
ansible-vault view —vault-id @password_file external_ vault_vars. yaml



```

