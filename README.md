```
$ hexo generate (hexo g) 生成静态文件
$ hexo server (hexo s) 启动本地服务
$ hexo deploy (hexo d) 提交到远程仓库
$ hexo new page "xx"(hexo n page) 创建页面 
$ hexo new "xx" (hexo n "") 创建文章
$ hexo d -g 生成静态并提交到远程仓库
$ hexo s -g 生成静态文件并启动本地预览
$ hexo clean 清除本地 public 文件
```



```
hexo d -g

git checkout hexo
git add .
git commit -m "backup_2019_xx_xx"
git push origin hexo
```

