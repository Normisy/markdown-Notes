## 1.1 上传到自己的仓库
首先，需要对仓库进行克隆，其路径在`<>code`这栏可以看到：
![[Pasted image 20240906170706.png]]
利用`git clone <相应路径>` 就可以**克隆该仓库到本地了**

在对已建好仓库进行克隆以后：
```
34382@Normist MINGW64 /
$ git clone https://github.com/Normisy/machineLearn.git
Cloning into 'machineLearn'...
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Compressing objects: 100% (2/2), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
Receiving objects: 100% (3/3), done.

```
通过`find`指令可以发现该仓库直接就在根目录下：
```
34382@Normist MINGW64 /
$ find . -name "machineLearn" -type d
./machineLearn

```
cd到仓库：
```
34382@Normist MINGW64 /
$ cd ./machineLearn

34382@Normist MINGW64 /machineLearn (main)
```

**此时仓库里是没有我们想加入的文件的，需要使用`mv`指令或者`cp`指令把想加入仓库的文件移动或拷贝到仓库目录下：**
```
34382@Normist MINGW64 /machineLearn (main)
$ mv D:/opencvLearn/machineLearn_1/.venv/Perceptron.py .

# 拷贝成功了
```
这样仓库目录下就有了该文件，我们就可以使用`git add`指令把它加入**暂存区**：
```
34382@Normist MINGW64 /machineLearn (main)
$ git add Perceptron.py

# 加入成功了
```
然后使用`git commit`命令**提交至仓库并且书写提交日志：**
```
34382@Normist MINGW64 /machineLearn (main)
$ git commit -m "上传了感知器类的python实现代码"
[main 0c2942b] 上传了感知器类的python实现代码
 1 file changed, 30 insertions(+)
 create mode 100644 Perceptron.py
```
可以使用`git log`命令查看提交日志：
```
34382@Normist MINGW64 /machineLearn (main)
$ git log
commit 0c2942b2de7db3510309fc230f897502b46bbd7c (HEAD -> main)
Author: Normist <3438252006@qq.com>
Date:   Fri Sep 6 17:02:09 2024 +0800

    上传了感知器类的python实现代码

commit 86775675000ea6bee93fd1171f3071b692d4a553 (origin/main, origin/HEAD)
Author: Zhihan Zeng <105109665+Normisy@users.noreply.github.com>
Date:   Thu Sep 5 22:14:35 2024 +0800

    Initial commit

```

然后，只需要**执行`git push`，Github上的仓库就会被相应更新了：**
```
34382@Normist MINGW64 /machineLearn (main)
$ git push
Enumerating objects: 4, done.
Counting objects: 100% (4/4), done.
Delta compression using up to 20 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 1.24 KiB | 1.24 MiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
To https://github.com/Normisy/machineLearn.git
   8677567..0c2942b  main -> main

```
（第一次会让你在浏览器或代码上登陆github）

完成！
![[Pasted image 20240906170540.png]]

