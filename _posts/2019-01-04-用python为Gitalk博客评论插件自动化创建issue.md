---
layout:     post
title:      用python为Gitalk博客评论插件自动化创建issue
subtitle:   
date:       2019-01-04
author:     chujunjie
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - python
---



## 前言

​	由于**Disqus**在国内加载比较慢，所以选了Gitalk作为博客的评论插件，支持markdown语法。但是Gitalk 需要手动初始化所有文章的评论或者一个一个点开界面才会创建对应的 issue，非常麻烦。这篇 [自动初始化 Gitalk 和 Gitment 评论](https://draveness.me/git-comments-initialize)，解决了这个问题，但是自己不会Ruby，所以用python写一个脚本，顺便记录下自己踩坑的过程。

## sitemap

​	sitemap可以列出网站中的网址以及关于每个网址的其他元数据，用于数据抓取，所以先通过 [jekyll-sitemap](https://github.com/jekyll/jekyll-sitemap) 插件为博客创建对应的 sitemap 文件。

​	 jekyll-sitemap插件需要Ruby环境。。。所以还是要先安装ruby（需要2.1以上的版本，注意CentOS7 yum默认安装的是2.0）

​	1.更换gem源

````
# 删除默认的gem源 
gem sources --r http://rubygems.org/
# 增加ruby-china作为gem源，taobao那个已经停止维护了 
gem sources -a https://gems.ruby-china.org/
# 查看当前的gem源
gem sources
# 清空源缓存
gem sources -c
# 更新源缓存
gem sources -u
````

​	2.安装jekyll-sitemap

![1](https://raw.githubusercontent.com/chujunjie/chujunjie.github.io/master/img/post_img/2019-01-04/1.png)

​	3.在_config.yml中添加

```
plugins:
	- jekyll-sitemap
	- jekyll-paginate
    
gems: [jekyll-paginate]
```

​	4.安装jekyll-paginate

![2](https://raw.githubusercontent.com/chujunjie/chujunjie.github.io/master/img/post_img/2019-01-04/2.png)

​	5.运行jekyll，默认在4000端口开启博客服务

![3](https://raw.githubusercontent.com/chujunjie/chujunjie.github.io/master/img/post_img/2019-01-04/3.png)

​	6.在博客项目目录的_site文件夹下生成sitemap.xml

## 创建issue

​	1.首先要在 GitHub 创建一个新的 [Personal access tokens](https://github.com/settings/tokens)，选择 `Generate new token` 后，并为该 Token 添加所有 Repo 的权限

![4](https://raw.githubusercontent.com/chujunjie/chujunjie.github.io/master/img/post_img/2019-01-04/4.png)

​	2.抓取sitemap.xml中的所有文章url

```python
def capture():
    url = 'https://xxx/sitemap.xml'
    html = urllib.request.urlopen(url).read()
    html = html.decode('utf-8')
    r = re.compile(r'(xxx/2018.*?</loc>)')  # 截取sitemap.xml中的所有文章url
    big = re.findall(r, html)
    for i in big:
        str = i[:-6]  # 去掉</loc>标签
        op_sitemap_url = open('sitemap_url.txt', 'a')  # 保存到sitemap_url.txt
        op_sitemap_url.write('%s\n' % str)
```

​	3.自动化创建issues，并生成相关标签

```python
def create_issues():
    suffix = ' - xxx的博客 | MY Blog'  # 自定义标题后缀
    g = Github(login_or_token="xxxxxx")  # 使用第一步创建的token登陆
    repo = g.get_repo("xxx/xxx.github.io")  # 指定仓库
    # open_issues = repo.get_issues(state='open')  # 获取仓库下open的issues
    for line in open("sitemap_url.txt"):  # 指定生成的sitemap_url
        line_ = line.rsplit('/', 2)  # 截取url获取标题部分
        title = unquote(line_[1]) + suffix  # unquote url_decode 拼接标题
        body = 'https://' + line  # 拼接https
        label = ['Gitalk', md5_label(line[13:].rstrip("\n"))]  # 标签
        repo.create_issue(title, body=body, labels=label)  # 创建issue
        
def md5_label(arg):
    hash = hashlib.md5()
    hash.update(arg.encode("utf8"))
    return hash.hexdigest()
```

​	搞定！

![5](https://raw.githubusercontent.com/chujunjie/chujunjie.github.io/master/img/post_img/2019-01-04/5.png)

​	GitHub 中 issue 的可以创建但是并不能删除，所以在配置时请一定检查好所有的配置项是否正确，虽然可以关闭issue，但是看起来还是非常头疼，强迫症表示接受不了。