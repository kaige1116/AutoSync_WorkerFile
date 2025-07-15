# Sync_Worker_File

要实现同步 `yonggekkk/cloudflare` 仓库的 `_worker.js` 文件到你的私人仓库，并每 12 小时自动检测更新，可按以下步骤操作：

---

### **整体方案**
1. **创建私人仓库**（存放同步文件）
2. **设置 GitHub Actions 工作流**（定时检测 + 同步）
3. **使用脚本比较文件差异**（检测更新）
4. **自动更新文件并修改 README**

---

### **详细步骤**

#### 一、准备工作
1. **创建私人仓库**  
   - 在 GitHub 创建新仓库（如 `my-cloudflare-worker`），设置为 **Private**。

2. **生成访问令牌**  
   - 进入 GitHub **Settings → Developer settings → Personal access tokens**  
   - 生成新 Token（勾选 `repo` 和 `workflow` 权限），保存为 `ACCESS_TOKEN`。

3. **添加 Token 到仓库 Secrets**  
   - 在你的私人仓库中：**Settings → Secrets → Actions**  
   - 添加 Secret：  
     - `PAT`: 值为刚生成的 Token（用于推送代码）  
     - `SOURCE_REPO`: 值为 `yonggekkk/cloudflare`（源仓库）

---

#### 二、创建 GitHub Actions 工作流
在私人仓库创建文件：  
`.github/workflows/sync_worker.yml`

```yaml
name: Sync Worker File

on:
  schedule:
    - cron: '0 */12 * * *'  # 每12小时运行一次（UTC时间）
  workflow_dispatch:        # 支持手动触发

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Private Repo
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.PAT }}

      - name: Fetch Source File
        run: |
          curl -s -L https://raw.githubusercontent.com/${{ secrets.SOURCE_REPO }}/main/_worker.js -o _worker_new.js

      - name: Compare Files
        id: check_diff
        run: |
          if [ -f _worker.js ]; then
            if cmp -s _worker.js _worker_new.js; then
              echo "no_change=true" >> $GITHUB_OUTPUT
            else
              echo "no_change=false" >> $GITHUB_OUTPUT
            fi
          else
            echo "no_change=false" >> $GITHUB_OUTPUT
          fi

      - name: Update File and README
        if: ${{ steps.check_diff.outputs.no_change == 'false' }}
        run: |
          # 更新文件
          mv _worker_new.js _worker.js
          
          # 更新时间到 README
          echo "# Last Sync Time" > README.md
          echo "Updated at $(date -u +'%Y-%m-%d %H:%M:%S UTC')" >> README.md
          
          # 配置 Git
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"

      - name: Push Changes
        if: ${{ steps.check_diff.outputs.no_change == 'false' }}
        run: |
          git add _worker.js README.md
          git commit -m "Auto-sync: Update _worker.js"
          git push origin main
```

---

#### 三、工作流说明
1. **定时触发**  
   - `cron: '0 */12 * * *'`：每天 0:00、12:00（UTC）运行。

2. **文件比较逻辑**  
   - 使用 `curl` 下载源文件到 `_worker_new.js`  
   - 通过 `cmp` 命令比较新旧文件内容是否相同。

3. **更新逻辑**  
   - 如果文件有变化：  
     - 覆盖旧文件 `_worker.js`  
     - 更新 `README.md` 为最新时间戳  
     - 自动提交并推送到私人仓库。

4. **无变化时跳过推送**  
   - 当文件无变化时，不执行后续步骤。

---

### **验证流程**
1. **手动触发测试**  
   - 在仓库的 **Actions** 标签页，选择 `Sync Worker File` → **Run workflow**。

2. **检查结果**  
   - 查看 `Actions` 运行日志是否成功。  
   - 仓库中应出现：  
     - `_worker.js`（同步的文件）  
     - `README.md`（显示最后更新时间）

---

### **注意事项**
1. **时区问题**  
   - cron 使用 UTC 时间，如需本地时区调整，修改 cron 表达式（例如北京时间 UTC+8 需减 8 小时）。

2. **源文件路径**  
   - 如果 `yonggekkk/cloudflare` 中文件不在根目录，修改 curl 路径：  
     ```bash
     curl .../path/to/_worker.js -o _worker_new.js
     ```

3. **首次运行**  
   - 如果私人仓库初始为空，工作流会创建 `_worker.js` 和 `README.md`。

4. **错误排查**  
   - 如果推送失败，检查 `PAT` 权限是否包含 `repo`。

---

通过以上步骤，即可实现全自动同步，并在 README 中记录更新时间。
