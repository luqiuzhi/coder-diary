# 迅速找出Linux的大文件

在Linux系统中，快速找出大文件或目录的常用方法如下：

---

### **方法1：使用 `du` + `sort` 命令（推荐）**
**定位大目录：**
```bash
sudo du -h --max-depth=2 / | sort -rh | head -n 20
```
- **说明**：
    - `du -h`：以人类可读格式（如GB/MB）显示目录大小。
    - `--max-depth`：指定文件夹的深度，可以根据文件夹一层一层往下查找大文件。
    - `sort -rh`：按数值倒序排序（`-r` 倒序，`-h` 适配单位）。
    - `head -n 20`：显示前20个结果。
    - 替换 `/` 可指定其他目录（如 `/home`）。

**定位所有大文件（包括子目录）：**
```bash
sudo du -ah / | sort -rh | head -n 20
```
- `-a`：显示所有文件（默认仅目录）。

---

### **方法2：使用 `find` 命令**
**直接查找指定大小的文件：**
```bash
sudo find / -type f -size +1G -exec ls -lh {} \;
```
- **说明**：
    - `-type f`：仅搜索文件。
    - `-size +1G`：查找大于1GB的文件（可调整，如`+500M`）。
    - `-exec ls -lh {} \;`：显示文件详细信息。
    - 排除目录（如 `/proc`）：
      ```bash
      sudo find / -path /proc -prune -o -path /sys -prune -o -type f -size +1G -exec ls -lh {} \;
      ```

---

### **方法3：使用 `ncdu` 工具（交互式）**
1. **安装工具**：
   ```bash
   sudo apt install ncdu    # Debian/Ubuntu
   sudo yum install ncdu    # CentOS/RHEL
   ```
2. **扫描目录**：
   ```bash
   sudo ncdu /
   ```
    - 按 **Enter** 进入子目录，**d** 删除文件，**q** 退出。
    - 交互界面直观显示文件/目录大小，支持排序。

---

### **方法4：`ls` 命令（仅当前目录）**
```bash
ls -lSh | head -n 20
```
- **说明**：
    - `-l`：长格式显示。
    - `-S`：按文件大小排序。
    - `-h`：人类可读格式。
    - 仅适用于当前目录，不递归子目录。

---

### **注意事项**
1. **权限问题**：建议使用 `sudo` 避免遗漏系统文件。
2. **排除干扰目录**：如 `/proc`、`/sys`、`/dev` 等虚拟文件系统。
3. **谨慎操作**：删除前确认文件用途，避免误删系统关键文件。

---

通过上述方法，可快速定位大文件并释放磁盘空间。推荐优先使用 `ncdu` 或 `du + sort` 组合。