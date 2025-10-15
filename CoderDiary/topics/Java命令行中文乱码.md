# Java命令行中文乱码

中文乱码问题通常是由于字符编码不一致导致的。以下是几种解决方案：

## 1. 设置JVM默认编码

在启动时指定UTF-8编码：
```bash
java -Dfile.encoding=UTF-8 -jar your-app.jar
```

## 2. 检查并设置系统终端编码

### Windows CMD
```bash
# 查看当前编码
chcp

# 设置为UTF-8（65001代表UTF-8）
chcp 65001
java -Dfile.encoding=UTF-8 -jar your-app.jar
```

### Windows PowerShell
```powershell
# 设置输出编码
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
java -Dfile.encoding=UTF-8 -jar your-app.jar
```

### Linux/Mac
```bash
# 检查当前locale
echo $LANG

# 启动时设置环境变量
LANG=zh_CN.UTF-8 java -Dfile.encoding=UTF-8 -jar your-app.jar
```

## 3. 完整的启动命令示例

```bash
# Linux/Mac
java -Dfile.encoding=UTF-8 -Dsun.jnu.encoding=UTF-8 -jar your-app.jar

# Windows
chcp 65001
java -Dfile.encoding=UTF-8 -jar your-app.jar
```
