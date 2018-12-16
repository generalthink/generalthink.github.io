title: 备份hexo配置
date: 2015-10-26 22:33:00
tags: [hexo]
categories: learning
description: 备份hexo配置
keywords: hexo配置 config.yml
---
## 备份hexo配置
```java
# Hexo Configuration
## Docs: http://hexo.io/docs/configuration.html
## Source: https://github.com/hexojs/hexo/

# Site
title: I am TKing
subtitle:
description: 相信自己做一个自信的人
author: generalthink
#主题支持的语言
language: zh-Hans
timezone:
email: general_go@163.com
since: 2014

# URL
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
url: http://yoursite.com
root: /
permalink: :year/:month/:day/:title/
permalink_defaults:

# Directory
source_dir: source
public_dir: public
tag_dir: tags
archive_dir: archives
category_dir: categories
code_dir: downloads/code
i18n_dir: :lang
skip_render:

# Writing
new_post_name: :title.md # File name of new posts
default_layout: post
titlecase: false # Transform title into titlecase
external_link: true # Open external links in new tab
filename_case: 0
render_drafts: false
post_asset_folder: false
relative_link: false
future: true
highlight:
  enable: true
  line_number: true
  auto_detect: true
  tab_replace:

# Category & Tag
default_category: uncategorized
category_map:
tag_map:

# Date / Time format
## Hexo uses Moment.js to parse and display date
## You can customize the date format as defined in
## http://momentjs.com/docs/#/displaying/format/
date_format: YYYY-MM-DD
time_format: HH:mm:ss

# Pagination
## Set per_page to 0 to disable pagination
per_page: 10
pagination_dir: page

# Extensions
## Plugins: http://hexo.io/plugins/
## Themes: http://hexo.io/themes/
theme: next
avatar: /images/avatar.jpg
social:
	github: https://github.com/k
	zhihu: http://www.zhihu.com/people/think123

links_title: 友情链接
# links
links:
  baidu: http://www.baidu.com
  it2048: http://www.it2048.cn

#多说评论
duoshuo_shortname: generalthink
# 多说分享服务
duoshuo_share: true

#hexo-generator-archive plugin
archive_generator:
  per_page: 10
  yearly: true
  monthly: true

#hexo-generator-category
category_generator:
  per_page: 10

#hexo-generator-index
index_generator:
  per_page: 10

toc:
  maxDepth: 3


# Deployment
## Docs: http://hexo.io/docs/deployment.html
deploy: 
  type: git
  repo: git@github.com:generalthink/generalthink.github.io.git
  branch: master

```