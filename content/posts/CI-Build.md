+++
title = 'CI 构建学习'
author = 'Kirito'
date = 2024-08-31T16:00:27+08:00
tags = ['build']
categories = ['CI']
keywords = ['CI', '持续集成', '构建']
summary = '***CI***（Continuous Integration，持续集成）构建是一种软件开发实践，旨在提高开发效率和代码质量。在工作的过程当中使用过，只不过使用的环境为***FinalBuilder***，现在学习一下***CI***的基本概念，以及简单学习一下如何使用Github来完成CI构建。'
draft = false
+++

# 1. CI构建是什么？

CI（Continuous Integration，持续集成）构建是一种软件开发实践，旨在提高开发效率和代码质量。它涉及到自动化构建、测试和部署流程。以下是对 CI 构建的详细解释：

## **1.1 CI（持续集成）的定义**

持续集成是一种软件开发方法，其中开发人员经常将代码更改合并到主代码库中。每次代码更改都会触发自动化构建和测试过程，以便及时检测和修复集成问题。

## **1.2 CI 构建的关键组成部分**

- **自动化构建**：
  - 每次代码提交后，CI 系统会自动执行构建过程。构建过程通常包括编译代码、打包应用程序和生成可执行文件等步骤。
- **自动化测试**：
  - 构建完成后，CI 系统会运行自动化测试，包括单元测试、集成测试和其他类型的测试。测试结果将被用来判断代码更改是否引入了新的问题或错误。
- **持续反馈**：
  - CI 系统会生成构建和测试报告，并将结果反馈给开发人员。开发人员可以快速查看构建状态、测试结果以及可能出现的问题，从而在代码集成过程中及时进行修复。

## **1.3 CI 构建的工作流程**

1. **代码提交**：
   - 开发人员将代码更改提交到版本控制系统（如 Git）。
2. **触发 CI 构建**：
   - CI 系统检测到代码更改后，自动触发构建和测试过程。
3. **执行构建和测试**：
   - CI 系统自动执行构建脚本，编译代码、生成构建工件，并运行测试用例。
4. **生成报告**：
   - CI 系统生成构建和测试报告，报告结果包括构建成功或失败的状态、测试覆盖率、失败的测试用例等信息。
5. **反馈和修复**：
   - 开发人员根据报告结果修复代码中的问题，然后再次提交代码，触发新的 CI 构建过程。

## **1.4 CI 工具和平台**

常见的 CI 工具和平台包括：

- **Jenkins**：一个开源的自动化服务器，支持多种插件和自定义构建流程。
- **GitHub Actions**：GitHub 提供的 CI/CD 服务，集成在 GitHub 仓库中。
- **GitLab CI/CD**：GitLab 的集成 CI/CD 工具，支持在 GitLab 仓库中自动构建、测试和部署。
- **CircleCI**：一个云端 CI/CD 服务，提供快速的构建和测试能力。
- **Travis CI**：一个云端 CI 服务，支持与 GitHub 仓库的集成。

## **1.5 CI 构建的优点**

- **早期发现问题**：自动化测试帮助及早发现和修复集成问题。
- **提高开发效率**：自动化构建和测试减少了手动操作，提高了开发速度。
- **增强代码质量**：持续集成过程确保每次代码更改都经过验证，提升了代码质量。
- **简化部署过程**：通过自动化的构建和测试，部署过程更加可靠和一致。

CI 构建是现代软件开发流程的重要组成部分，有助于提高代码质量、减少错误和加快开发周期。

# 2. Final-Builder CI

Final-Builder是一种用户自动化构建和发布过程的工具，页面像是一个IDE是有一个可视化页面，我们可以通过拖拽的方式来创建复杂的构建脚本。该构建方式是在公司里面遇到的，主要就是通过一些现有的FB构建项目(某些人在维护的构建项目)。该FB工程会在运行的时候读取相应的配置文件，大多是*.ini*格式的配置文件，来完成整个项目的构建过程。

**关于为什么使用CI来构建？上面给出的回答是ChatGPT所给出的。**

如果平时是一个人开发一个项目，CI构建可以有也可以没有，因为你拥有开发该项目所有的依赖项和源码文件。但是为什么可有可无呢？就拿我正在书写的博客举例，我的博客是利用GitHub的Pages和Hugo博客框架进行搭建的。从我写博客到将博客生成的内容上传到GitHub并渲染这个过程如果没有借助于***GitHub CI***构建工具，整个流程可能是这样的：

1. 书写相应的文章；
2. 在本地使用Hugo进行编译；
3. 将生成的编译文件上传到GitHub上，大部分是一个***public***文件夹；
4. 在这个上传的过程当中，你需要设置好相关的路径等等，可能会遇到一些各种各样的问题；

但是如果你使用GitHub CI来部署我们的项目的时候，流程是这样的：

1. 编写**GitHub CI所需要的配置文件(workflow)**并上传到GitHub仓库当中；
2. 书写相应的文章，之后直接推送到远端仓库即可；

这个过程你会发现使用GitHub CI的话，可能就是编写GitHub CI所需要的配置文件的时间，之后部署的话便完全交给GitHub CI来帮助我们完成所有的事情，我们只需要编写我们的文章就可以，推送的时候会自动触发CI构建来帮助我们进行博客的更新。

如果平时是团队开发一个大的项目，该项目可能由一些版本管理工具进行集中管理，而且***我们每一个人对于该仓库的访问权限并不一样***（可能只有我们负责的那一部分源码的访问权限）。我们开发的时候可能依赖于别人开发的模块，该模块可能会以***动态链接库.dll***(.bpl)或者***可执行文件.exe***的形式存在，这样就会导致如果我们需要最新的依赖项，如果没有CI构建工具，我们可能会去对应模块的负责人，让他主动的为我们提供相应的依赖文件，这个过程就会很繁琐麻烦。***但是如果有CI构建工具，我们就可以实时的去构建相应依赖项模块***，获取到对应的最新的依赖文件，这样就避免了和别人去交涉的过程，完全可以直接拿到最新的依赖项文件。

我在公司进行开发的时候，有一个公司自己的CI构建平台用来构建公司内部所有的模块和项目，并且最终会将那些构建生成的文件放在一个公共可以供所有人访问的地方。这样平时开发的时候，如果需要最新的依赖项的话，我只需要找到对应的项目，点击构建获取到最新的依赖项即可。

# 3. Github CI

GitHub的CI构建是通过仓库当中`.github/workflows/config.yaml`配置文件以GitHub Action来完成的，实际上就是编写一个脚本文件，该脚本文件所做的事情就是部署该项目所需要的环境以及依赖项，之后对其进行编译以及部署。

下面文件是我使用GitHub CI来搭建我的个人博客所使用的配置文件：

```yaml
# Sample workflow for building and deploying a Hugo site to GitHub Pages
name: Deploy Hugo site to Pages

on:
  # Runs on pushes targeting the default branch
  push:
    branches:
      - main

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# Sets permissions of the GITHUB_TOKEN to allow deployment to GitHub Pages
permissions:
  contents: read
  pages: write
  id-token: write

# Allow only one concurrent deployment, skipping runs queued between the run in-progress and latest queued.
# However, do NOT cancel in-progress runs as we want to allow these production deployments to complete.
concurrency:
  group: "pages"
  cancel-in-progress: false

# Default to bash
defaults:
  run:
    shell: bash

jobs:
  # Build job
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.128.0
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb          
      - name: Install Dart Sass
        run: sudo snap install dart-sass
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5
      - name: Install Node.js dependencies
        run: "[[ -f package-lock.json || -f npm-shrinkwrap.json ]] && npm ci || true"
      - name: Build with Hugo
        env:
          HUGO_CACHEDIR: ${{ runner.temp }}/hugo_cache
          HUGO_ENVIRONMENT: production
          TZ: America/Los_Angeles
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"          
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  # Deployment job
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

之后更新仓库的时候会触发这一系列的Action:

![action](/action.png)


