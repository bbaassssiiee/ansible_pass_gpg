# ansible\_pass\_gpg
Install GPG as a helper for ansible-vault

GPG is useful if you want to sign commits in git, or if you want to use encryption, decryption, signing and verification.

## Ansible-vault

The ansible-vault command-line tool allows us to create and edit an encrypted file that ansible-playbook will recognize and decrypt automatically, given the password.

This tool ensures the data is encrypted at rest (i.e, on disk) only. It is your own responsibility to set `no_log: true` on tasks that use this data.

We can encrypt an existing file like this:

```
 $ ansible-vault encrypt secrets.yml
```

Alternately, we can create a new encrypted file in the special directory `group_vars/all/` next to our playbook. I store global variables in `group_vars/all/vars.yml` and secrets in `group_vars/all/vault` (without extension, to not confuse linters and editors).

```
$ mkdir -p group_vars/all/
$ ansible-vault create group_vars/all/vault
```

ansible-vault prompts for a password, and will then launch a text editor so that you can work in the file. It launches the editor specified in the **$EDITOR** environment variable. If that variable is not defined in your shell’s profile (export EDITOR=code), it defaults to vim.

Use the `vars_files` section of a play to reference a file encrypted with ansible-vault the same way you would access a regular file.

ansible-playbook needs to prompt us for the password of the encrypted file, or it will simply error out. Do so by using the `--ask-vault-pass` argument:

```
 $ ansible-playbook --ask-vault-pass playbook.yml
```

You can also store the password in a text file and tell ansible-playbook its location by using the **ANSIBLE_VAULT_PASSWORD_FILE** environment variable or the `--vault-password-file` argument:

```
 $ ansible-playbook playbook.yml --vault-password-file ~/password.txt
```

If the argument to `--vault-password-file` has the **executable** bit set, Ansible will execute it and use the contents of standard out as the vault password. This allows you to use a script to supply the password to Ansible.

## This role

This role lets you decrypt another file (`.vault_pw.gpg`) using pretty good privacy in a transparent way, using gpg agent. You don't need to type the password all the time, and the password is encrypted on disk with personal keys.

First you need to encrypt the ansible-vault password into such a file:
```
echo 'Your_vault_password' > "${HOME}/.vault_pw"
gpg -e -r $(gpg --list-secret-keys|grep ultimate|head -1|cut -d\< -f2|cut -d\> -f1) "${HOME}/.vault_pw"
rm "${HOME}/.vault_pw"
```


## Troubleshooting

Check if you can decrypt the file:
```
gpg -q -d "${HOME}/.vault_pw.gpg"
```

Check if the environment variable is set correctly:
```
cat $ANSIBLE_VAULT_PASSWORD_FILE
```

This should contain:
`exec gpg -q -d "${HOME}/.vault_pw.gpg"`

Run the $ANSIBLE_VAULT_PASSWORD_FILE executable:

```
$ANSIBLE_VAULT_PASSWORD_FILE
```
This should print the password.

## Development
You cannot run molecule localhost straightaway, first unset this environment variable and authenticate:

```sh
unset ANSIBLE_VAULT_PASSWORD_FILE
sudo -v
molecule test -s localhost
molecule converge -s localhost
```

### Required packages

For python scripting:

```
pip install --user -r requirements.txt
```

### Role variables

- `default_cache_ttl:` Interval in seconds (Default: 7200).
- `vault_user`: The user for whom the configuration should be applied (Default: $USER).


### Role dependencies

Ansible


