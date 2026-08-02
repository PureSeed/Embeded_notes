
**shell:** 本质是应用程序，接收到命令后会去执行
**Windows：** 无根目录，有硬盘分区
**乌班图：** 一切皆文件的思想，隐藏了复杂的物理存储细节，让用户专注于文件本身。
**挂载：** 即把新硬盘与相应的文件夹连接起来，系统会自动把数据写到对应的硬盘上
**PATH环境变量：** 相当于shell的搜索地图，有了它就不用每次都输入程序的完整路径，直接输入命令名即可运行程序


**乌班图目录的含义
![[Pasted image 20260802122925.png]]

------------------------------
## 常用命令

#### ls  列出目录的内容  list
ls -l   以长格式显示详细信息
ls -a   显示所有文件
ls -la  组合显示所有文件的详细信息
ls -lh  以人类可读的格式显示文件大小

#### cd  切换目录  change directory
cd ..  回到上一级目录
cd ~   回到当前用户主目录
cd /   回到根目录
cd -   回到上一次所在目录

#### pwd  显示当前工作目录  print working directory
pwd     显示当前所在目录的完整路径
pwd -P  显示物理路径，解析符号链接

#### cp  复制文件或目录  copy
cp -r  递归复制目录及其内容
cp -i  覆盖前询问确认
cp -b  显示复制过程详情


#### mv  移动或重命名文件  move
mv 旧文件名 新文件名       （重命名）
mv 文件 目标目录          （移动文件）
mv -i  覆盖前询问确认

#### rm  删除文件或目录  remove
rm -r   递归删除目录及其内容
rm -f   强制删除，不提示确认
rm -rf  强制递归删除

#### cat  查看文件内容  concatenate
cat file.txt  查看文件全部内容
cat -n        显示符号
cat file1.txt file2.txt >combined.txt   合并文件

#### sudo  以管理员身份执行命令  superuser do
sudo apt update  更新软件源
sudo reboot      重启系统
sudo -i          切换到root用户

#### ssh  远程登陆  secure shell
ssh username@ip  远程登录服务器
shh -p 端口号 username@ip  指定端口登录
shh-copy-id username@ip   免密登录配置

#### find  文件查找
find /path-name ".txt"         按文件名查找
find /path-type f -size+100M   查找大于100M的文件
find /path -mtime -7           查找7天内修改的文件
find .path -exec rm{}\         找到文件后执行删除

#### grep  搜索文件内容  global regular expression print
grep "关键词" file.txt    搜索文件中的关键词
grep -r "关键词" dir      递归搜索目录下所有文件
grep -i   忽略大小写
grep -n   显示匹配行的行号


#### chmod  修改文件权限
chmod 755 file  给文件设置rwxr-xr-x权限
chmod +x file   给文件添加可执行权限


#### chown  修改文件所有者
chown user:group file  修改文件的用户和组
chown _R user:group dir 递归修改目录权限


#### tar  压缩
tar -zcvf archive.tar.gz dir  打包并压缩
tar -zxvf archive.tar.gz      解压文件
tar -tf archive.tar.gz        查看压缩包内容


#### upzip  解压缩
unzip file.zip   解压zip文件
unzip file.zip -d dir  解压到指定目录

#### top/htop  查看系统资源使用情况
实时显示CPU、内存、进程占用情况

#### df -h  查看磁盘使用情况
以易读格式显示磁盘空间情况


### 高频选项
-l  long 长格式显示详细信息
-a  all 显示所有内容，包括隐藏文件
-h  human-readable 易读格式显示大小
-r  reverse 反向排序或递归操作
-v  verbose 显示详细执行过程
-f  force 强制操作，不提示确认
-i  interactive 交互式操作，需要确认
-n  number 显示行号或编号

**[ ] 可选       < > 必须提供

-----------

**终端命令行：** 和Linux系统对话的窗口，==负责系统层面的操作==。比如：创建目录，删除文件，安装软件，启动服务，查看系统状态等等
**VI编辑器：** 是编辑文本内容的工具，==负责文件内部的操作==。比如：写代码，改配置，编辑文本，处理文本内容等等。

# VI编辑器基础知识

### 打开和退出：
vi filename  打开或新建文件
:q   退出（文件未修改时）
:q!  强制退出，放弃所有修改
:wq  保存并退出
ZZ   快捷键：保存并退出

### 三种模式的切换：
**命令模式（默认）：** 打开文件自动进入，用于执行命令
**插入模式：** 按i/a/o进入，用于编辑文字
**底行模式：** 按冒号进入，用于执行高级操作
**通用返回：** 按ESC键可以从任何模式返回命令模式

### 文本编辑操作：
**插入文本：** 
i  在光标前插入
a  在光标后插入
o  在当前行下方新建一行并插入
O  在当前行上方新建一行并插入
**删除文本：** 
x
dw
ddndd
D
d$
**复制粘贴：** 
yy
nyy
yw
p
P
**撤销重做：** 
u
Ctrl+r
U

### 查找替换
**查找：** 
/
？
n
N
**替换：** 
:s/old/new
:s/old/new/g
:%s/old/new/g
:%s/old/new/gc

### 高级实用功能
:!command



### VI编辑器效率提升快捷键





