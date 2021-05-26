---
title: "使用Github Actions自动部署Hugo文档"
date: 2021-05-26T16:09:21+08:00
draft: true
tags: ["github"]
categories: ["DevOps"]
---
> Hugo是由Go语言实现的静态网站生成器。简单、易用、高效、易扩展、快速部署。

我们可以使用Hugo配合Github Page方便地搭建自己的博客，但是由于需要同时管理Hugo编辑和发布两个不同的环境，在处理完文档后我们往往需要对两个环境分别进行提交，会涉及到多次git操作，略微繁琐。其实通过Github Action功能，我们可以在提交编辑环境的同时，自动化发布和部署Hugo页面。

### 工具
* [Github Action](https://docs.github.com/en/actions)
* [GitHub Actions for Hugo](https://github.com/peaceiris/actions-hugo)：在Github Action中安装Hugo
* [GitHub Actions for GitHub Pages](https://github.com/peaceiris/actions-gh-pages)：在Github Action将静态文件发布到Github Page

### 设置
通常我们使用Github Page指定Repo:[github-name].github.io来管理发布环境，我们假设也使用该Repo中另一个分支在管理编辑环境：
* hugo-src分支管理Hugo编辑环境
* main分支管理Hugo发布环境

添加Action脚本(.github/workflows/main.yaml):
```yaml
name: Hugo Deploy

on:
  push:
    branches:
      - hugo-src  # 当Hugo编辑分支有推送时触发

jobs:
  build-deploy:
    runs-on: ubuntu-18.04
    steps:
      - name: Checkout      # 获取编辑
        uses: actions/checkout@master

      - name: Setup Hugo    # 初始化Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: latest  # 指定Hugo的版本
      
      - name: Build     # 编译Hugo，生成的网站会保存在./public文件夹中
        run: hugo -D
      
      - name: Deploy    # 将public文件夹中生成的网站提交到发布环境中(main分支)
        uses: peaceiris/actions-gh-pages@v3
        with: 
          github_token: ${{ secrets.GITHUB_TOKEN }} # 使用Github Token进行验证
          publish_branch: main  # 指定发布的分布
          publish_dir: ./public # 指定需要发布的文件夹
          commit_message: ${{ github.event.head_commit.message }}
```

其中Token验证的方式一共有3种: [详细参考这里](https://github.com/peaceiris/actions-gh-pages/blob/main/README.md#supported-tokens)
* github_token: 使用Github Actions自动生成的GITHUB_TOKEN进行验证。
* deploy_key：由自己生成密钥对，该属性值指向私钥对应的值（通常将私钥保存Secrets中），并将生成的公钥保存在发布环境的Settings/Deploy keys中来验证编辑环境。
* personal_token：使用Github账号的Personal access tokens进行验证，需要在Github Settings/Developer settings/Personal access tokens中添加token并赋予repo和workflow权限。
| Token | Private repo | Public repo | Protocol | Setup |
|---|:---:|:---:|---|---|
| `github_token` | ✅️ | ✅️ | HTTPS | Unnecessary |
| `deploy_key` | ✅️ | ✅️ | SSH | Necessary |
| `personal_token` | ✅️ | ✅️ | HTTPS | Necessary |

这样当我们更新编辑并提交推送后，Github Action会自动发布Hugo并推送到发布环境，省去了发布环境的提交推送操作。
![avatar](/images/deploy-hugo-by-github-action/01.png)