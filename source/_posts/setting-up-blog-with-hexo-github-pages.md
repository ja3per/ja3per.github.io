---
layout: 使用hexo和github
title: 使用hexo和github Pages搭建个人博客
date: 2019-03-26 10:57:52
tags:
---
### Refered document
* https://github.com/hexojs/hexo
* https://hexo.io/docs/

### Install Hexo
```
$ sudo npm install -g hexo-cli

$ hexo -v
hexo-cli: 1.1.0
os: Darwin 17.5.0 darwin x64
http_parser: 2.8.0
node: 10.9.0
v8: 6.8.275.24-node.14
uv: 1.22.0
zlib: 1.2.11
ares: 1.14.0
modules: 64
nghttp2: 1.32.0
napi: 3
openssl: 1.1.0i
icu: 62.1
unicode: 11.0
cldr: 33.1
tz: 2018e
```

### Create a project for your GitHub Pages
```
$ hexo init ja3per.github.io
INFO  Cloning hexo-starter to ~/ws/blogs/ja3per.github.io
...
✨  Done in 41.28s.
INFO  Start blogging with Hexo!

$ cd ja3per.github.io

$ npm install
```

### Run a test server for your page on Mac
```
$ hexo server
INFO  Start processing
INFO  Hexo is running at http://localhost:4000 . Press Ctrl+C to stop.
```
<!-- more -->
### Set information for your new blog
https://hexo.io/docs/configuration.html
```
$ vi _config.yml

~~~~~~~~~~~~~~~~~~ _config.yml ~~~~~~~~~~~~~~~~~~
# Site
title: Jasper's Blog
subtitle:
description: 
author: Jaspser
language:
timezone: 

# URL
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
url: http://ja3per.github.io/
root: /
permalink: :year/:month/:day/:title/
permalink_defaults:
```

### Set information to use Git
https://github.com/hexojs/hexo-deployer-git
```
$ npm install hexo-deployer-git --save
$ vi _config.yml

~~~~~~~~~~~~~~~~~~ _config.yml ~~~~~~~~~~~~~~~~~~
# Deployment
## Docs: http://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: git@github.com:ja3per/ja3per.github.io.git
  branch: master
```

### Set "watch" before starting your work
"watch" command can monitor your files.  
https://hexo.io/docs/generating.html
```
$ hexo generate --watch
```

### Create a new post file
```
$ hexo new first-post
INFO  Created: ~/ws/blogs/ja3per.github.io/source/_posts/ferst-post.md
```

### Edit the above file with Markdown or Hexo's Helper
Hexo's Helper  
https://hexo.io/docs/helpers.html  
I use Atom with "shift + control + m" when I use Markdown :-)  
https://atom.io/

### Delete "source/_posts/hello-world.md"
It's not necessary to deploy.

### Config ssh key for github
use ssh-keygen to generat keys
```
ssh-keygen -t rsa -b 4096 -C "sp2g88@gmail.com"
...
Your public key has been saved in /Users/sipeng.zhu/.ssh/id_rsa.pub.
```
copy the key value and setting in github
https://github.com/settings/keys


### Deploy your new blog!!
https://hexo.io/docs/deployment.html
```
$ hexo clean
$ hexo deploy
```
After writting the above command, you can see your new blog on GitHub Pages.  
http://ja3per.github.io/

### Change your blog theme
https://github.com/hexojs/hexo/wiki/Themes
```
For instance, How to use the following theme.
https://hexo.io/hexo-theme-next/

## Install it
$ cd ja3per.github.io
$ git clone https://github.com/iissnan/hexo-theme-next themes/next

## Set information to use the theme
$ vi _config.yml

~~~~~~~~~~~~~~~~~~ _config.yml ~~~~~~~~~~~~~~~~~~
# Extensions
## Plugins: http://hexo.io/plugins/
## Themes: http://hexo.io/themes/
theme: next
```
### Config theme settings 
https://theme-next.iissnan.com/theme-settings.html
```
vi theme/next/_config.yml
```

### Use "Read More"
Write `<!-- more -->` in your articles.  

### Use Plugins
https://github.com/hexojs/hexo/wiki/Plugins

### Binding domain
add CNAME to source folder
add domain info
```
www.aiwaner.cn
```

### 使用github对静态文件和Hexo源码管理 Source files and generated files management with github
1. create branch in github named "src" to store source files
2. edit _config.yaml
```
# Deployment
## Docs: https://hexo.io/docs/deployment.html

### master branch => static files for blogs
### src    branch => hexo source code

  - type: git 
    repo: git@github.com:ja3per/ja3per.github.io.git
    branch: master
  - type: git
    repo: git@github.com:ja3per/ja3per.github.io.git
    branch: src
    extend_dirs: /
    ignore_hidden: false
    ignore_pattern:
      public: .
      /: "node_modules|public"
```
3. deploy
```
$ hexo clean && hexo deploy
```
