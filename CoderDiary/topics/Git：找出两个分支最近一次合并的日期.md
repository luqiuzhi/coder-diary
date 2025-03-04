# Git：找出两个分支最近一次合并的日期

要快速查找 `dev` 分支和 `master` 分支最近一次的合并时间点，可以通过以下 Git 命令和策略实现：

---

### **1. 使用 `git log` 过滤合并提交**
通过 `git log` 的 `--merges` 参数筛选合并记录，并结合分支范围语法和日期格式化参数，定位最近一次的合并时间点。
```bash
# 查看 dev 合并到 master 的最近一次记录（从 master 视角）
git checkout master
git log -1 --merges --format="合并时间：%cd" --grep="Merge branch 'dev'" --date=iso

# 查看 master 合并到 dev 的最近一次记录（从 dev 视角）
git checkout dev
git log -1 --merges --format="合并时间：%cd" --grep="Merge branch 'master'" --date=iso
```
- **参数说明**：
    - `-1`：仅显示最近一次提交。
    - `--merges`：仅显示合并提交。
    - `--format="%cd"`：自定义输出格式，`%cd` 表示提交日期。
    - `--grep`：按合并描述中的关键字过滤（需根据实际合并信息调整）。
    - `--date=iso`：显示标准化的时间格式（如 `2025-02-24 10:00:00 +0800`）。

---

### **2. 使用分支对比语法缩小范围**
通过 `master...dev` 语法筛选两个分支之间的差异提交，快速定位合并点：
```bash
# 查看两个分支之间的合并记录
git log --oneline --merges --left-right master...dev
```
- **输出示例**：
  ```
  < 1234567 (master) Merge dev into master
  > abcdef0 (dev) Merge master into dev
  ```
    - `<` 表示从 `dev` 合并到 `master` 的记录。
    - `>` 表示从 `master` 合并到 `dev` 的记录。

---

### **3. 可视化工具辅助分析**
使用图形化工具（如 GitKraken、SourceTree）直接查看分支合并历史：
- **GitKraken**：在分支图中，合并点会以节点形式展示，悬停可查看具体时间。
- **命令行工具**：运行 `gitk --all` 或 `git log --graph --oneline --branches`，通过图形化界面定位合并时间点。

---

### **4. 补充说明**
- **合并方向**：需明确是 `dev→master` 还是 `master→dev` 的合并，两者可能不同步。
- **默认合并描述**：若合并时未自定义描述，可改用 `--grep="Merge branch"` 或省略 `--grep` 参数，手动筛选结果。
- **时间格式调整**：通过 `--date=relative` 可显示相对时间（如 `2 hours ago`）。

---

通过上述方法，可快速定位两个分支间最近的合并时间点。若需更精确的历史追溯，建议结合多个命令交叉验证。