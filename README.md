# introduction

* This playbook maintains a list of desired public SSH keys in root's `authorized_keys` file. 
* A different user can be specified with variable `target_user`. 

# how this playbook works

## files used in the playbook

```
.
├── ansible.cfg
├── files
│   └── ssh_pubkey_files
│       ├── admins.pub
│       ├── cto.pub
│       └── webdevelopers.pub
├── group_vars
│   └── all
├── inventory.txt
├── README.md
└── ssh_authorized_keys_playbook.yml

```

* `ansible.cfg` - Ansible settings to disable host key checking and enable retry files
* `files/ssh_pubkey_files` - This is where pubkey files should be put. A file can contain multiple keys (i.e. group file). You can have pubkey files for individual users and at the same time have those very same keys in group files. Ansible will take care of deduplication.
* `group_vars/all` - Here we specify default keyfiles. These will be put on the target host whether they are specified in the inventory file or not. 
* `inventory.txt` - List of target hosts. 
    * Here we specify which **extra key files** to put on the target host with `extra_key_files='["webdevelopers.pub","cto.pub"]'`
    * **Default key files** are already specified in `group_vars/all` and don't need to be written here.

## tasks performed by the playbook

* **Get target user's home directory** - retrieve info about target user from host's passwd database using Ansible's getent module and register the returned . 
* **Set user home directory fact** - take output of previous task (`user_info`) and set the 5th field of the `passwd` entry as a fact called `user_home`.
* **Collect public keys** - contents of key files specified in `extra_key_files` and `default_key_files` get put into in `key_contents`
* **Create compound authorized_keys file and replace existing one** - `authorized_keys` file is updated with deduplicated `key_contents`. 
    * Previous contents of the file are discarded.
    * If `.ssh` directory doesn't exist, it will be created.
* **Configure sshd to disable reverse DNS lookup** - `/etc/ssh/sshd_config` is updated to disable reverse DNS lookup to speed things up.
    * This task calls a helper task to restart ssh daemon if needed.

# how to use this playbook

* Put all your users' public ssh keys into `files/ssh_pubkey_files` under their own names. e.g.: `timapple.pub`
* Put all your teams' public ssh keys into `files/ssh_pubkey_files` under their team names. e.g.: `admins.pub`
    * It is ok to have the same key in a group file as well as an individual one. Ansible will deduplicate the compound key.
* In `group_vars/all` specify default key files you want to be on all your targets whether they're specifically requested or not.
* In `inventory.txt` specify nondefault, a.k.a. `extra_key_files`. See the existing example in the inventory file. 
* Running the playbook with `ansible-playbook -i inventory.txt ssh_authorized_keys_playbook.yml` will set up the `authorized_keys` file in root's home. i.e.: `/root/.ssh/authorized_keys`
    * If you append `-e "target_user=timapple"` to the ansible command above, then `authorized_keys` will be created in timapple's home directory instead of root's. 
    * If you append `-e "enable_debug=true"` to the ansible command above, then Debug tasks will run too. 

