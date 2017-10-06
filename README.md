源码分支：

把Hexo的源码备份到Github分支里面，思路就是上传到分支里存储，修改本地的时候先上传存储，再发布，更换电脑的时候再下载下来源文件。

打开git-bash

$ git init
$ git remote add origin git@github.com:username/username.github.io.git		
$ git add .
$ git commit -m "this is md file"
$ git push origin master:mdfile

现在你会发现github你的博客仓库已经有了一个新分支mdfile，我们的备份工作完成

以后，本地写好博文之后，可以先执行

$ git add .
$ git commit -m "blog"
$ git push origin master:mdfile
进行备份，然后

$ hexo d -g
进行更新静态文件。
这里推荐先进行备份，因为万一更新网站之后不小心丢失了源文件，就又得重新再来了


