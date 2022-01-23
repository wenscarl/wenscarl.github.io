---
title: How to deploy hexo blog on two machines
date: 2020-10-04 22:04:39
layout: post
tags: 
- Hexo
- git
---

### Why we need this?

Yes, Hexo is able to deploy with `npm install hexo-deployer-git --save` for us by simply setting up `_config.yml` file. It looks like normally: 

<!--more-->

```git
deploy: 
  type: git
  repository: git@github.com:username/username.github.io.git
  branch: master
```

Once you push to master branch, github stores the static webpage files instead of source file. Among all files it pushes, there is one `.deploy_git` which records how hexo tracks remote master branches. On computer 2, `git clone` only pull down the static webpage files instead of source files.  So you need a branch to keep source files. 

### About blogging with Hexo

Check [here](https://hexo.io/)!

## How?

* On computer 1's source file folder, you may checkout a new local branch *hexo*

  * Let it track to remote's *hexo* branch.

    ```
    git branch --set-upstream-to origin/hexo hexo
    ```

    

  * Push it to remote as

    ```
    git push origin hexo
    ```

  * Don't forget to get rid of `.git` folder for your themes if they're cloned. 

* On compute 2, pull down only *hexo* branch by

  ```
  git init && git remote add origin <remote_repo>
  git fetch origin hexo:hexo && git checkout hexo or
  git checkout --track origin/hexo (most recent versions of Git)
  or git clone -b hexo <remote_repo> <folder_name>
  ```
* On compute 2, pull down only *hexo* branch by

* ```
  git init && git remote add origin <remote_repo>
  git fetch origin hexo:hexo && git checkout hexo or
  git checkout --track origin/hexo (most recent versions of Git)
  or git clone -b hexo <remote_repo> ....
  ```

  * cd into this folder, and `npm install & npm install hexo-deployer-git --save`

  * write new blog, generate and deploy

    ```
    hexo g
    hexo new post "post_name"
    hexo d
    ```

  * once finished, push to remote

    ```
    git add source
    git commit -m "xxxx"
    git push
    ```

    
