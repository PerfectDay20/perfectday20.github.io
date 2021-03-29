---
title: 20180819 Login Shell & Variable
date: 2018-08-19 21:30:58
tags: Shell
---
1. Login shell 就是登录时的 shell， 用 echo $0 会前缀 '-'，如 '-bash'；会读取 profile 类文件，如 ~/.bash_profile，.bash_profile 中一般有语句会读取 ~/.bashrc
2. Non-login shell 是被调用的 shell 或者脚本，不会读取 profile 类文件，如果是 interactive 的，会读取 .bashrc 文件
3. rc 文件都是 interactive shell 会读取的
4. Environment variable 一般放在 profile 文件中，用 export a=b 配置，这样整个 session 只会读一次
5. Local variable 放在 rc 文件中，用 a=b 配置，这个每个打开的 shell 都会读一次

---
参考：
[Difference between Login shell and Non login shell](http://howtolamp.com/articles/difference-between-login-and-non-login-shell/)
[Bash startup scripts](http://howtolamp.com/articles/bash-startup-scripts/)
[Difference between Local Variables and Environment Variables](http://howtolamp.com/articles/difference-between-local-and-environment-variables#environment)
[What's the difference between .bashrc, .bash_profile, and .environment?](https://stackoverflow.com/questions/415403/whats-the-difference-between-bashrc-bash-profile-and-environment)