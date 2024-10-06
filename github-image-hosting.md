---
cover: >-
  .gitbook/assets/DALL·E 2024-10-03 20.39.45 - An illustration representing a
  secure image file storage network in a more colorful and vibrant style. The
  picture frame should remain as the central .webp
coverY: 0
layout:
  cover:
    visible: true
    size: hero
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
---

# PicGo + Github私库作图床，Cloudflare Workers加速，Gitlab 实时备份

## 1 前言和项目特点

针对新建博客中大量使用图片，故需搭建一个私人图床。既要避免使用收费的商业云 oss 储存；又要在白嫖的前提下找个长期稳定的存放平台；还要兼顾访问速度；最好有异地灾备。为了实现这些目标，便有了以下综合解决方案。

方案特点和优势：

* **存储在 GitHub 私库**：图片存放在 GitHub 私有仓库中，路径经过脱敏处理，确保安全性。
* **自动压缩图片**：通过自动压缩图片，提升加载速度，优化用户体验。
* **Cloudflare CDN 加速**：利用 Cloudflare CDN 提供的全球加速服务，确保图片快速加载。
* **自定义域名**：支持使用自定义域名，提升品牌形象和专业性。
* **多种链接形式输出**：生成多种形式的图片链接，方便在博客或其他平台使用。
* **实时备份到 Gitlab 私库（可选）**: 将 GitHub 实时备份到 GitLab 私有库提供了数据冗余和跨平台备份，增强了数据安全性和保障了业务连续性。

## 2 条件

* **GitHub 账号**，https://github.com
* **GitLab 账号**，可选项，如不需要备份可忽略，https://gitlab.com
* **Cloudflare 账号**，https://www.cloudflare.com
* **Tinypng 账号**，https://tinypng.com/developers
* **安装 Snipaste**，https://zh.snipaste.com
* **安装 PicGo**，https://molunerfinn.com/PicGo

## 3 流程和工具介绍

1. **截图**：使用 Snipaste 进行截图，这一款简单但强大的贴图工具，支持截屏、标注等功能，适用于 Windows 和 Mac。
2. **PicGo 上传图片到 GitHub**：使用 PicGo ，并安装 picgo-plugin-compress-next 插件，自动压缩图片后上传到 GitHub。
3. **获取经过 Worker 处理的图片链接**： 自动生成经过 Worker 隐藏 GitHub 私库的 PAT，增加 CDN 和双栈，输出自定义域名 url。

## 4 详细关键步骤

### 4.1 前置条件，获取 GitHub PAT 和 Tinypng API Key

* 登陆 GitHub，访问 https://github.com/settings/tokens 获取 PAT，并建一个私库用于存放图片，现在 GitHub 每个项目为 4G 大小，超过就再建新的可以了

![](https://img.forvps.gq/pic1/main/xblog/202410021723640.png)

![](https://img.forvps.gq/pic1/main/xblog/202410021727192.png)

![](https://img.forvps.gq/pic1/main/xblog/202410021728651.png)

* 在 GitHub 里建一个私库用于存放图片，访问 https://github.com/new ，现在 GitHub 每个项目为 4G 大小，超过就再建新的可以了

![](https://img.forvps.gq/pic1/main/xblog/202410021754757.png)

* 注册 Tinypng.com 账号，访问 https://tinypng.com/developers 注册

![](https://img.forvps.gq/pic1/main/xblog/202410021732916.png)

![](https://img.forvps.gq/pic1/main/xblog/202410021737590.png)

![](https://img.forvps.gq/pic1/main/xblog/202410021741463.png)

### 4.2 PicGo 设置

* 安装好 PicGo 后，继续在软件里安装 picgo-plugin-compress-next 插件（此步可能需要开启科学） ![](https://img.forvps.gq/pic1/main/xblog/202410021748338.png)

![](https://img.forvps.gq/pic1/main/xblog/202410021759241.png)

* 继续设置 PicGO 主程序

![](https://img.forvps.gq/pic1/main/xblog/202410021803360.png)

![](https://img.forvps.gq/pic1/main/xblog/202410021804466.png)

![](https://img.forvps.gq/pic1/main/xblog/202410021810527.png)

![](https://img.forvps.gq/pic1/main/xblog/202410021919688.png)

### 4.3 Cloudflare 创建 worker

* 登陆 Cloudflare，访问 https://dash.cloudflare.com/ ，新建 worker ![](https://img.forvps.gq/pic1/main/xblog/202410021821406.png)
* 部署后，编辑代码，复制并按图修改相应两处位置

```
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  const url = new URL(request.url)
  const path = url.pathname

  // 环境变量 PAT，如果未设置则使用默认值
  const PAT = '<改为你的pat，只开放repo权限即可>'

  // 构建 GitHub raw 内容的 URL
  const githubUrl = `https://raw.githubusercontent.com/<替换为你的github用户名，包括前后的尖括号也要去掉>${path}`

  // 创建新的请求，添加必要的头部
  const modifiedRequest = new Request(githubUrl, {
    method: request.method,
    headers: {
      ...request.headers,
      'Authorization': `token ${PAT}`,
      'Accept': 'application/vnd.github.v3.raw'
    }
  })

  // 发送请求并返回响应
  try {
    const response = await fetch(modifiedRequest)

    // 如果响应不成功，抛出错误
    if (!response.ok) {
      throw new Error(`GitHub API responded with ${response.status}: ${response.statusText}`)
    }

    // 创建新的响应，保留原始内容但移除敏感头部
    const newResponse = new Response(response.body, response)
    newResponse.headers.delete('Authorization')

    return newResponse
  } catch (error) {
    return new Response(`Error: ${error.message}`, { status: 500 })
  }
}
```

![](https://img.forvps.gq/pic1/main/xblog/202410021826382.png)

* 添加自定义域名 ![](https://img.forvps.gq/pic1/main/xblog/202410021833665.png)

## 5. 正式使用流程

### 5.1 截图

* 使用 Snipaste 进行截图到粘贴板，可以使用快捷键 (默认 Ctrl + F1)

### 5.2 上传图片

* 通过 PicGo 上传图片到 GitHub，可以使用快捷键（默认 Ctrl + Shift + P）或拖拽图片到 PicGo 界面
* 上传成功进度条是蓝色的，如果是红色，请检查设置是否有误。上传成功后自定义域名链接会自动在粘贴板
* 在相应的场景 Blog 或者论坛等地址粘贴出来 ![](https://img.forvps.gq/pic1/main/xblog/202410021903739.png)

![](https://img.forvps.gq/pic1/main/xblog/202410021910158.png)

## 6 实时备份到 Gitlab （可选）

### 6.1 Gitlab 设置

* 登陆 Gitlab，新建 Project，访问 https://gitlab.com/projects/new#blank\_project ![](https://img.forvps.gq/pic1/main/xblog/202410041737895.png)
* 新增 PAT 和权限 ![](https://img.forvps.gq/pic1/main/xblog/202410041746430.png) ![](https://img.forvps.gq/pic1/main/xblog/202410041748347.png)
* 设置允许强制推送，以便 Github workflow 能把项目推送过来 ![](https://img.forvps.gq/pic1/main/xblog/202410041752702.png)

### 6.2 Github 设置

* 设置 3 个 secrets

| Name             | secret               |
| ---------------- | -------------------- |
| GITLAB\_USERNAME | Gitlab 用户名           |
| GITLAB\_REPO     | Gitlab 上创建的项目名       |
| GITLAB\_PAT      | 上面生成的 Gitlab 项目的 PAT |

![](https://img.forvps.gq/pic1/main/xblog/202410041822855.png)

* 创建备份到 Gitlab 的工作流，路径和文件为

`.github/workflows/sync-to-gitlab.yml` 代码如下：

```
name: Sync to GitLab

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  sync:
    runs-on: ubuntu-latest

    env:
      GITLAB_USERNAME: ${{ secrets.GITLAB_USERNAME }}
      GITLAB_REPO: ${{ secrets.GITLAB_REPO }}
      GITLAB_PAT: ${{ secrets.GITLAB_PAT }}

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4.1.1
      with:
        fetch-depth: 0

    - name: Set up Git
      run: |
        git config --global user.name 'github-actions'
        git config --global user.email 'github-actions@github.com'

    - name: Add GitLab remote
      run: git remote add gitlab https://${{ env.GITLAB_USERNAME }}:${{ env.GITLAB_PAT }}@gitlab.com/${{ env.GITLAB_USERNAME }}/${{ env.GITLAB_REPO }}.git

    - name: Force push to GitLab
      run: git push gitlab main --force
```

![](https://img.forvps.gq/pic1/main/xblog/202410041756675.png)

![](https://img.forvps.gq/pic1/main/xblog/202410041831458.png)

![](https://img.forvps.gq/pic1/main/xblog/202410041833372.png)

![](https://img.forvps.gq/pic1/main/xblog/202410041836988.png)

## 7 图片案例

https://img.forvps.gq/pic1/main/example/202410021622722.png

![image](https://img.forvps.gq/pic1/main/example/202410021622722.png)
