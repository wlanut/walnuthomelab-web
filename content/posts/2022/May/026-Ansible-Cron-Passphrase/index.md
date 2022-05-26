---
title: "Ansible Unattended With SSH Passphrases"
date: 2022-05-26 23:21:18 #(use ctrl+Shift+I to insert date string)
# weight: 1
# aliases: ["/first"]
tags: ["Ansible","SSH"]
categories: ["tutorials"]
author: "Omar El-Sherif"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: true
draft: false
hidemeta: false
comments: false
#description: ""
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: true
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: false
ShowBreadCrumbs: true
ShowPostNavLinks: true

cover:
    image: "img/cover.png" # image path/url
    alt: "There should be an image here..." # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
editPost:
    URL: "https://github.com/wlanut/walnuthomelab-web/tree/master/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

## SSH Key Auth

While I am pretty new to the realm of SSH and Ansible, I learned early on that SSH key authentication is the gold standard for security and practicality. Ideally, you also want a passphrase on your key. This doesn't negate the need to rotate your keys or change them upon a compromising event, but it does add an extra layer of security, i.e. if a bad actor gained access to your private key, they would still need your passphrase in order to use it.

With this knowledge, I created a private key with a passphrase for my Ansible server to use. I ran into some obstacles early on with playbooks prompting me for the passphrase, but solved them by learning to use an [SSH agent (launched on login) to keep my keys loaded](https://serverfault.com/a/672386/957243), and [SSH forwarding](https://www.calazan.com/using-ssh-agent-forwarding-with-ansible/) to pass those keys to ansible when necessary.

However, I've been rebuilding and reorganizing my playbook structure, breaking things out into roles, and finally wanted to run some playbooks on a set schedule using cron. For a while now, I've struggled with figuring out a workflow for running the playbooks unattended, because without me being there to start the SSH agent and add the key with my passphrase, there would be a necessary interactive step preventing execution no matter what.

So after some trial and error, and a lot of Googling, I think I came up with a solution to my problem.

## The Problem

We can't easily run Ansible playbooks unattended when using an SSH key with a passphrase to make our remote connections, because you have to be there to enter the passphrase at runtime (or when starting your agent).

So how can we enter the passphrase in a non-interactive way? We can use an "expect" script to pass our passphrase (credit to [Thomas Nyman on this post](https://unix.stackexchange.com/a/90869) for doing the legwork), like so:

```bash
#!/usr/bin/expect -f
spawn ssh-add /home/username/.ssh/id_rsa
expect "Enter passphrase for /home/username/.ssh/id_rsa:"
send "passphrase\n";
expect "Identity added: /home/username/.ssh/id_rsa (username@host)"
interact
```

This shows your passphrase in plain text, so on its own this is not a great solution. But we'll get to that in a bit.

We can utilize this script using Ansible in one of two ways.

**Method 1 - ansible.builtin.script**

```yaml
- name:
    ansible.builtin.script: /home/username/ssh-add.exp
    args:
      executable: expect
```

**Method 2 - ansible.builtin.shell** (I'm using this method, I'll explain why later)

```yaml
- name: add ssh key
    ansible.builtin.shell: |
      spawn ssh-add "{{ssh_key_location}}"
      expect "Enter passphrase for {{ssh_key_location}}:"
      send "{{ssh_passphrase}}\n";
      expect "Identity added: {{ssh_key_location}}"
      interact
    args:
      executable: /usr/bin/expect
```

As long as an SSH agent exists and is running, these tasks will add our key to the agent, allowing us to successfully connect to remote hosts.

**Note**: I can't seem to get the agent to start from within Ansible. Running it as a separate task (either as a script or with the shell module) does not persist the agent through to the "expect' script. Trying to run both the agent and the "expect" script from the same task using the shell module will technically "work", but not once the script is vaulted.

## Encryption At Rest

The secret sauce for all of this is **Ansible Vault**. Since we're using Ansible to execute our script/commands, we can just `ansible-vault encrypt` our secrets, and have our passphrase - previously plain text - now encrypted at rest. If you need more info on Ansible Vault, check out [the documentation](https://docs.ansible.com/ansible/latest/user_guide/vault.html).

You can use vault with either of the above described methods. For **Method 1**, you'll want to encrypt the expect script you're calling, using [vault **file** encryption](https://docs.ansible.com/ansible/latest/user_guide/vault.html#encrypting-files-with-ansible-vault). In **Method 2**, you'll want to list the passphrase as a variable in your task, and encrypt your vars file (or use [vault **variable** encryption](https://docs.ansible.com/ansible/latest/user_guide/vault.html#encrypting-files-with-ansible-vault)).  The reason I'm using Method 2 because I like being able to use variables to keep things nice and short. It also means I can encrypt just the variable I want to hide without having to obfuscate the entire file.

In addition to encrypting your secret, you can also add the ["no_log" argument](https://docs.ansible.com/ansible/latest/reference_appendices/faq.html#how-do-i-keep-secret-data-in-my-playbook) to any task or playbook to prevent the decrypted file/string from showing in the verbose ansible logging. Make sure to leave this off while debugging.

## Role Example

The tasks we created can be used as either a playbook or a role. I'm using it as a role so that when I make a playbook that will run unattended, I can just stick the role at the beginning (just make sure to run it against localhost).

```yaml
---
- hosts: localhost
  roles:
  - roles/ssh-add

- hosts: ubnt-servers
  become: yes
  roles:
  - roles/deploy-packages

- hosts: localhost
  roles:
  - roles/ssh-kill_agent
```

Note that I've added an additional role at the end "ssh-kill_agent", the tasks/main.yml looks like this:

```yaml
- name:
    ansible.builtin.shell: eval "$(ssh-agent -k)"
    args:
      executable: bash
```

When creating a cron job for this playbook, we'll need to manually start the SSH agent using `eval "$(ssh-agent -s)"` at the beginning of the line. The role at the end sill simply kill the SSH agent that was created by cron to ensure that we aren't leaving additional agents behind. You can use `pidof ssh-agent` to verify how many SSH agent processes are running, `eval "$(ssh-agent -k)"` to kill the current session, and `killall ssh-agent` to kill all running SSH agent processes.

## Crontab

I'm very new to cron and therefore not an expert with it, so if you take issue with my syntax, I will not be offended. This is essentially my current cron job:

```bash
0 2 * * * eval "$(ssh-agent -s)" && /usr/bin/ansible-playbook /path/to/playbook.yml >> /path/to/cronlog 2>&1 && curl -fss -m 10 --retry 5 https://hc-ping.com/f8-n9v3ytn432-dsnj3e3210-ddl7
```

- `crontab -e` \- Opens the crontab file for editing
- `0 2 * * *` \- Run at 2am every day (helpful [cron generator](https://crontab-generator.org/) tool if you need it)
- `eval "$(ssh-agent -s)"` \- Start the SSH agent
- `&&` Only run next task if previous one completes successfully
- `/usr/bin/ansible-playbook` \- Path to ansible-playbook executable (locate by running `whereis ansible-playbook` in your terminal)
- `/path/to/playbook.yml` \- Path to your ansible playbook
- `>>` \- append output to file
- `/path/to/cronlog 2>&1` \- path to desired log file, [see here for an explanation](https://unix.stackexchange.com/a/163359) of the last bit. If you'd like to [include the date](https://serverfault.com/a/117365/957243), replace this part with something like ``$HOME/.logs/`date +%Y-%m-%d_%H:%M:%S`-cron.log 2>&1``
- `curl -fss -m 10 --retry 5 https://hc-ping.com/f8-n9v3ytn432-dsnj3e3210-ddl7` \- My [Healthchecks.io monitoring](https://healthchecks.io/) check

Feel free to use all, some, or none of the above. I'm leaving in the extra file logging step in the middle for now because it'll be helpful for debugging if/when something goes wrong (and my healthcheck ping will hopefully tell me when something goes wrong). You can run `sudo grep -a "CRON" /var/log/syslog` to verify that your cron job is actually firing off.

Another thing to note is that in this case, the && operators work on the assumption that Ansible returns a 0 exit code. Ansible can apparently be [weird about exit codes](https://jwkenney.github.io/ansible-return-codes/). For example, if a host in the inventory is unreachable, even if the playbook successfully completes despite that unreachable host, it will not count as a 0 exit code, and therefore && will not work as expected. With that said, ensure that you plan around failures as necessary.

## Wrap Up

That's about it! I understand this use case may not be solving some great conundrum. Maybe others have either decided to simply not use a passphrase on the key for their Ansible server, or maybe they have a separate one just for automated tasks. This solved my problem of not wanting to remove my passphrase, but still be able to automate tasks in an unattended manner ***WITHOUT*** having to leave my passphrase out in the open in plaintext.

If you'd like the view these examples in code format, please visit my [ansible scripts repository](https://github.com/wlanut/ansible-public/tree/master/ssh-add-passphrase).