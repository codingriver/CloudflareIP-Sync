Cloudflare IP 段自动同步部署指南

本文档介绍如何通过 GitHub Actions 自动从 Cloudflare 官方同步 ip.txt / ipv6.txt，
并配合脚本内置的自动下载功能，实现 IP 段文件的全自动维护。



一、GitHub Actions 自动同步

仓库内置的 .github/workflows/sync-cf-ips.yml 工作流会每周自动从 Cloudflare 官方同步最新 IP 段。

设置步骤





在 GitHub 创建仓库（或使用已有仓库 codingriver/CloudflareSpeedTest-Python）



将项目推送到仓库：

 git init
 git add .
 git commit -m "init"
 git remote add origin https://github.com/codingriver/CloudflareSpeedTest-Python.git
 git push -u origin main



进入仓库 → Settings → Actions → General → 确保 Allow all actions 已勾选



Actions 会在每周一 UTC 04:00（北京时间 12:00）自动运行，也可手动触发：





进入 Actions 标签页 → 选择 Sync Cloudflare IP Ranges → 点击 Run workflow

工作流执行内容





从 cloudflare.com/ips-v4/ 和 cloudflare.com/ips-v6/ 下载最新数据



验证文件非空



对比是否有变化，有变化才提交



自动 commit + push，保持仓库中的 ip.txt / ipv6.txt 始终最新

修改同步频率

编辑 .github/workflows/sync-cf-ips.yml 中的 cron 表达式：

# 每周一次（默认，推荐）
- cron: '0 4 * * 1'

# 每两天一次
- cron: '0 4 */2 * *'

# 每天一次
- cron: '0 4 * * *'

# 每月一次
- cron: '0 4 1 * *'



Cloudflare IP 段通常一年只变动 2~4 次，每周同步已足够及时。



二、脚本自动下载配置

cfst.py 内置了自动下载功能：本地没有 ip.txt 或 ipv6.txt 时，运行测速会自动从以下源依次尝试下载：





Cloudflare 官方（cloudflare.com/ips-v4/、cloudflare.com/ips-v6/）



jsDelivr CDN 加速（需配置 GITHUB_REPO）



ghfast 镜像加速（需配置 GITHUB_REPO）



GitHub Raw（需配置 GITHUB_REPO）

配置方法

编辑 cfst.py 顶部的 GITHUB_REPO：

GITHUB_REPO = "codingriver/CloudflareSpeedTest-Python"

配置后，即使 Cloudflare 官网被墙，也能通过国内 CDN 加速镜像自动下载。

下载优先级

Cloudflare 官方 → jsDelivr CDN → ghfast 镜像 → GitHub Raw
     ↓ 失败           ↓ 失败          ↓ 失败         ↓ 失败
   尝试下一个       尝试下一个      尝试下一个      提示手动下载



三、交互模式手动更新

在交互菜单中选择 [U] 更新 IP 文件，可以手动触发下载更新：





显示当前 ip.txt / ipv6.txt 的文件状态（大小、修改时间）



选择更新 IPv4、IPv6 或全部



从上述多个源依次尝试下载



四、工作流文件参考

工作流文件位于 .github/workflows/sync-cf-ips.yml：

name: Sync Cloudflare IP Ranges
on:
  schedule:
    - cron: '0 4 * * 1'
  workflow_dispatch:
permissions:
  contents: write
jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Download Cloudflare IP ranges
        run: |
          curl -sL --fail --retry 3 https://www.cloudflare.com/ips-v4/ -o ip.txt
          curl -sL --fail --retry 3 https://www.cloudflare.com/ips-v6/ -o ipv6.txt
      - name: Validate files
        run: |
          if [ ! -s ip.txt ]; then echo "ip.txt is empty"; exit 1; fi
          if [ ! -s ipv6.txt ]; then echo "ipv6.txt is empty"; exit 1; fi
          echo "ip.txt: $(wc -l < ip.txt) lines"
          echo "ipv6.txt: $(wc -l < ipv6.txt) lines"
      - name: Commit if changed
        run: |
          git diff --quiet ip.txt ipv6.txt && echo "No changes" && exit 0
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add ip.txt ipv6.txt
          git commit -m "sync: update Cloudflare IP ranges $(date +%Y-%m-%d)"
          git push

