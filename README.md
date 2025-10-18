### 项目说明

本项目为《数字孪生流域》课程作业仓库。仓库包含两个作业，分别存放在不同的 Git 分支中：

- LSTM_camels_homework：作业分支 1，LSTM-CAMELS 流量预测作业
- prcp_basins_mean_homework：作业分支 2，降雨数据算法与流域平均雨量计算作业

当前分支为默认分支 `main`，用于汇总说明与导航。

### 如何切换到各作业分支（网页切换）

1) 打开本仓库的网页，点击分支下拉选择目标分支：`LSTM_camels_homework` 或 `prcp_basins_mean_homework`。

提示：切换到对应分支后，请仔细阅读该分支下的 README.md，按照分支说明进行安装与运行。

2) 获取代码：

- 方式 A：在网页中点击 “Code” → “Download ZIP” 下载当前分支压缩包。
- 方式 B：在网页中点击 “Code” 复制仓库的 HTTPS/SSH 地址，随后在目标环境使用 `git clone -b <分支名> <仓库地址>` 获取指定分支代码。

3) 在服务器上运行的三种常见做法：

- 服务器直接复制（少量文件/快速试跑）

当只需运行或查看少量文件时，可在网页切换到目标分支后，直接复制文件内容到服务器：

1) 在网页切换到所需分支，打开目标文件，点击 “Raw”（或“查看原始”），选择并复制所需内容。

2) 登录服务器后在启动页新建Python文件并粘贴内容；

3) 如果有多个文件，重复上述步骤，或使用原始链接直接下载单个文件：

完成复制后，在服务器进入该目录，按文件内或分支的说明运行代码即可。

完成以上步骤后，进入对应作业目录，按分支内的说明在服务器上安装依赖并运行。

- 从本地上传到服务器：

```bash
# macOS/Linux 示例（将本地解压后的目录上传到服务器）
scp -r ./repo_dir username@server:/path/to/destination
```

- 在服务器直接下载（无需本地中转）：

```bash
# 使用网页“Download ZIP”复制得到的下载链接
wget "<ZIP 下载链接>" -O repo.zip
unzip repo.zip -d ./repo_dir

# 或基于仓库地址直接拉取指定分支
git clone -b LSTM_camels_homework <仓库地址>
git clone -b prcp_basins_mean_homework <仓库地址>
```

### 本地切换（将代码clone到本地以后的操作）

1) 拉取远端分支信息（建议先执行一次）：

```bash
git fetch origin --prune
```

2) 查看可用分支：

```bash
git branch -a
```

3) 切换到作业分支（本地已存在该分支时）：

```bash
# 切换到作业1分支
git switch LSTM_camels_homework

# 切换到作业2分支
git switch prcp_basins_mean_homework
```

提示：切换到对应分支后，请仔细阅读该分支下的 README.md，了解环境依赖、数据准备与运行步骤。

4) 如果本地尚未创建对应的远端跟踪分支，可先基于远端分支创建本地分支再切换：

```bash
# 作业1分支（首次本地使用）
git switch -c LSTM_camels_homework origin/LSTM_camels_homework

# 作业2分支（首次本地使用）
git switch -c prcp_basins_mean_homework origin/prcp_basins_mean_homework
```

5) 可替代命令（适用于旧版 Git 或习惯用法）：

```bash
git checkout LSTM_camels_homework
git checkout prcp_basins_mean_homework
```

6) 返回主分支：

```bash
git switch main
```

以上命令在 Windows PowerShell、CMD 以及类 Unix Shell 中均可使用（确保已安装 Git）。

