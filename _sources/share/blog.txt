打造Rst的GitHub博客
=============================

:date: 2013-08-17

之前的博客使用markdown写过几篇，不过最近发现rst更加方便，因此上午折腾了一把，打算改用rst来记录博客。

我先是搜索了一把，找到 `这篇博客 <http://blog.xlarrakoetxea.org/posts/2012/10/creating-a-blog-with-pelican/>`_ 
感觉挺好的。于是，我也打算用pelican来构建，可惜折腾了一个上午，始终没有达到预期效果。只好放弃，改用rst+sphinx来弄。
默认的主题我不太喜欢，搜索了几个，最后决定使用 `armstrong <https://github.com/armstrong/armstrong_sphinx/>`_ 。(bootstrap也不错的)

最后，我在layout.html中加上disqus的链接，虽然看起来有些唐突，好歹算是博客的功能都有了！

写博客的流程如下 ::

 #先在github上建库,为了方便，我是建立了两个库blog.git和joyyoj.github.com
 $ git clone git@github.com:user/blog.git
 $ cd ./blog
 #master分支下，提交改动
 $ git add .
 $ git commit -m "First post"
 $ git push origin master
 $ make html && cp _build/html/* ../joyyoj.github.com
 $ cd ../joyyoj.github.com
 $ git add .
 $ touch .nojekyll 文件
 $ git commit -m "Publish hello world post"
 #发布到博客上
 $ git push origin master

 git 其他常用的操作
 Removing untracked files from your git working copy
 If you want to also remove directories, run git clean -f -d
 If you just want to remove ignored files, run git clean -f -X
 If you want to remove ignored as well as non-ignored files, run git clean -f -x

touch .nojekyll 原因可以参考 `这里 <https://help.github.com/articles/using-jekyll-with-pages/>`_

帮助链接
----------

`reStructuredText入门 <http://www.pythondoc.com/sphinx/rest.html>`_

`sphinx使用文档 <http://docs.kissyui.com/1.3/docs/html/tutorials/tools/use-sphinx.html>`_
