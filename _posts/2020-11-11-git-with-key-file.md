---
title:  "使用指定的 SSH key 操作 git"
author: adream307
date:   2020-11-11 21:01:00 +0800
categories: [linux,git]
tags: [git, ssh]
---
Starting from Git 2.3.0 we also have the simple command (no config file needed):

```bash
GIT_SSH_COMMAND='ssh -i private_key_file -o IdentitiesOnly=yes' git clone user@host:repo.git
```



With git 2.10+ (Q3 2016: released Sept. 2d, 2016), you have the possibility to set a config for `GIT_SSH_COMMAND`


A new configuration variable `core.sshCommand` has been added to specify what value for `GIT_SSH_COMMAND` to use per repository.

```bash
cd /path/to/my/repo/already/cloned
git config core.sshCommand 'ssh -i private_key_file -o IdentitiesOnly=yes' 
# later on
git pull
```

## 参考资料
- <https://stackoverflow.com/questions/4565700/how-to-specify-the-private-ssh-key-to-use-when-executing-shell-command-on-git>
- <https://stackoverflow.com/questions/4565700/how-to-specify-the-private-ssh-key-to-use-when-executing-shell-command-on-git/38474137#38474137>