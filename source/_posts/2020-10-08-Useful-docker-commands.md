---
title: Useful docker commands
date: 2020-10-07 23:06:57
tags: docker
layout: post
---

```bash
 ~/.bash_aliases
# kill all running containers
alias dockerkill='docker kill $(docker ps -a -q)'
# delelte all stopped containers
alias dockercleanc='docker rm $(docker ps -a -q)'
# delete all untaged images
alias dockercleani='docker rmi $(docker images -q -f dangling=true)'  
# delte all stopped and untaged images
alias dockerclean='dockercleanc || true && dockercleani'
```
