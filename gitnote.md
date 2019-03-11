---
title: Git使用笔记（持续记录）
date: 2018-01-07
---

- 由于git总是有意无意的切换，经常忘记git的操作，干脆无脑的记下来，便于查看。
- 本机现在global用户名和邮箱是GitHub的仓库的，在某个code.aliyun仓库下局部用户名和邮箱是单独的，虽然局部的是和目录强相关，但是个人感觉毕竟很少切换，便于管理多个RSA。

---

## 通用命令

`git init`
`git add .`
`git add xxx(目录名或者完整文件名)`
`git commit -m "first commit"`

HTTPS关联远程仓库（将远程主机命名为origin），使用用户名和密码
`git remote add origin https://github.com/xxx/xxx.git`

SSH关联远程仓库（将远程主机命名为origin），使用RSA
`git remote add origin git@github.com:xxxx/xxxx.git`

配置局部用户名和邮箱
`git config user.name "xxx"`
`git config user.email "xxx@xxx.com" `

列出已经存在的远程主机
`git remote`

删除已经存在的关联的远程主机origin
`git remote rm origin`

push命令格式（将本地分支提交到远程主机的远程分支）
`git push <远程主机名> <本地分支名>:<远程分支名>`

push代码到远程仓库（本地master提交到远程主机origin）
`git push origin master`

如果省略本地分支名，则表示删除指定的远程分支，因为这等同于推送一个空的本地分支到远程分支。
`git push origin :master`等同于`git push origin --delete master`

---

## IDEA中Git操作

对于**“本地新建工程首次推送到远程”**和**“ IDEA拉取远程仓库已有的工程，并修改提交”**两种情况分别记录一种自己喜欢的操作

#### IDEA创建工程并与远程仓库关联

>这种情况我倾向于使用命令，比较方便准确。而且还是倾向于使用https，因为后续提交使用IDEA可以记住用户名和密码，免去配置各种RSA。连续命令如下：
`git init`
`git add xxxx`
`git commit -m "first commit"`
`git remote add origin https://github.com/xxx/xxx.git`
`git push -u origin master`

#### IDEA拉取远程仓库已有的工程，并修改提交

>这种情况我喜欢使用IDEA插件的功能，比较无脑方便
>
>首先检出工程：
>
>![从已经存在的远程仓库检出工程](https://upload-images.jianshu.io/upload_images/3727888-1de93728534be4d1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>
>由于我的global已经配置了另一个仓库的RSA，而这个仓库我不想添加RSA，所以使用HTTPS方式拉取。不过即便如此，clone也不需要用户名和密码，所以不会弹出用户名和密码的输入框。
>
>![HTTPS方式拉取远程仓库代码](https://upload-images.jianshu.io/upload_images/3727888-478111770d4133b5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>
>等到本地有修改之后，git进行commit
>
>![提交本地的修改](https://upload-images.jianshu.io/upload_images/3727888-64a8ca71814feef3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>
>然后push到远程仓库，此时如果是第一次push，会弹出用户名和密码输入框。
>
>![第一次push时需要输入用户名和密码](https://upload-images.jianshu.io/upload_images/3727888-84b1641e033ced78.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>
>与使用git命令不同的是，idea可以记住用户名和密码，不需要每次提交都输入用户名和密码。所以对于IDEA，我倾向于不添加RSA，直接使用用户名和密码。

## GitHub的Hexo博客源码分支管理

>在此记录一下Hexo博客的源码管理（此方法是引用的网络资源经过个人实践验证，但具体链接已无从寻起）。
>#### 开源码分支
>把Hexo的源码（一系列markdown文件）备份到Github分支里面，思路就是在Github的博客仓库中创建一个分支，如下图：
>
>![blog源码分支](https://upload-images.jianshu.io/upload_images/3727888-aeeed407fa8bf6ec.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>
>其中master是正常的博客文件，mdfile是所有markdown的源文件，每次修改本地的时候先上传源代码到mdfile分支存储，再发布，更换电脑的时候再下载下来源文件即可（以下内容有些年久失修，可能不太靠谱，有待再次实践验证）。
>
>#### 分支开好之后，首次进入到博客根目录下：
>`git init`
>`git remote add origin git@github.com:xxxx/xxx.github.io.git`
>`git branch mdfile`(本地开一个mdfile分支)
>`git checkout mdfile`(切换到mdfile分支)
>#### 然后进入源文件目录(_post)下：
>`git add . `
>`git commit -m "this is md file"`
>`git push origin mdfile:mdfile`(将本地的mdfile推送到orgin主机的mdfile分支，如果加上-f，则强制上传)
>
>#### 以后修改/发布博文过程（已验证）
>进入到源文件目录（_post）下（如果发生过分支切换则需要切换到源码分支操作）
>`git add .` 或者 `git add xxx`
>`git commit -m "xxx"`
>`git push origin mdfile:mdfile `
>然后 `hexo d -g `进行更新静态文件。