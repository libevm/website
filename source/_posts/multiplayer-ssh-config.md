---
title: Multiplayer SSH Config
date: 2021-12-22
---

To manage multiple SSH configs I use the [ssh-agent](https://github.com/ohmyzsh/ohmyzsh/blob/master/plugins/ssh-agent/ssh-agent.plugin.zsh) plugin.

```bash
...

plugins=(git ssh-agent command-time)

# Will look in ~/.ssh directory
zstyle :omz:plugins:ssh-agent identities github_rsa anon1_rsa anon2_rsa

source $ZSH/oh-my-zsh.sh

...
```

With the following `~/.ssh/config`:

```bash
Host github.com
        HostName github.com
        User git
        IdentityFile ~/.ssh/github_rsa

Host github.com-anon1
        HostName github.com
        User git
        IdentityFile ~/.ssh/anon1_rsa

Host remote-vpc
        Hostname ec1-11-111-121-11.eu-central-1.compute.amazonaws.com
        User ubuntu
        IdentityFile ~/.ssh/aws_key
        StrictHostKeyChecking no
```

Git configuration for multiplayer

```bash
git commit --date="$(TZ=PST date)" -m "MESSAGE"

git config user.email "gmail.com"
git config user.name "libevm"
```