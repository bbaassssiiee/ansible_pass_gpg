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


# Encrypted secrets with password in home directory

In automation providing a password on a prompt is not always feasible or desirable. Then we can use one, or several files in the home directory: `.vault_pass_cloud`, or `.vault_pass_acc`, and a `.vault_pass_prod password`.

These files don't need to be plain-text, but when they are ensure restrictive permissions. Much better to encrypt them with GPG and delete tha plain-text files:

```bash
gpg -e -r john.doe@example.com ~/.vault_pass_cloud
rm ~/.vault_pass_cloud
```

The environment variable ANSIBLE_VAULT_PASSWORD_FILE points to a file to find the password. If that file is an executable, then it is run to retrieve the password from stdout.

You can simply start with one file:
`export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass_cloud`

To edit the `secrets` file you can run this command:

`ansible-vault edit --encrypt-vault-id cloud --vault-id cloud@~/.vault_pass_cloud linux_test/secrets`

## This role

This role lets you decrypt the vault files (`.vault_pass_whatever.gpg`) using pretty good privacy in a transparent way, using gpg agent. You don't need to type the GPG passphrase all the time, and the vault passwords for the vaults are encrypted on disk with personal keys.

First you need to encrypt the ansible-vault password into such a file:
```
echo 'Your_vault_password' > "${HOME}/.vault_pw"
gpg -e -r $(gpg --list-secret-keys|grep ultimate|head -1|cut -d\< -f2|cut -d\> -f1) "${HOME}/.vault_pw"
rm "${HOME}/.vault_pw"
```

# Encrypted secrets in multiple vaults

There are situations in which you want to use multiple Vault files with different passwords (e.g., when you have different Vaults per environment that have different passwords). That’s where Vault Identities come in.

Let's say we need separate passwords, one for the "cloud" and one each for the "acc" and "prod" environment. We place these Vault files in their respective locations, $INFRA/group_vars/<ANSIBLE_INVENTORY/secrets.

To distinguish between the password files, we use Vault Identities to separate them and the vaults. To make this work, add the necessary vault-id flag to the ansible-vault commands. 

`ansible-vault edit --encrypt-vault-id cloud --ask-vault-pass linux_test/secrets` 

## Troubleshooting

Check if you can decrypt the file:
```
gpg -q -d "${HOME}/.vault_pass_cloud.gpg"
```

Check if the environment variable is set correctly:
```
cat $ANSIBLE_VAULT_PASSWORD_FILE
```

This should contain:
`exec gpg -q -d "${HOME}/.vault_pass_cloud.gpg"`

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


