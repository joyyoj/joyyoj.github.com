新gerrit注册和申请权限
1、登录http://git.inf.baidu.com:8888/
2、设置Contact Information，包括用户名和邮箱。
（PS：邮箱会收到邮件，需要访问链接确认，确认链接末尾可能会有‘=’，需要完整）
3、设置SSH Public Keys
将旧gerrit上的SSH拷贝到新gerrit
http://cq01-testing-ds015.cq01.baidu.com:8888/#/settings/ssh-keys
4、hi史吉冬，告诉所属项目，申请加入相应的权限组

将原gerrit上的已存在的评审推送到新gerrit上操作步骤：
1).cd到原来的本地git目录下，添加新的git仓库(名字叫inf，指向新的git库)
git remote add inf ssh://${USER}@git.inf.baidu.com:29418/baidu
2). 将本地最新的git分支推送至新的inf仓库
git push inf HEAD:refs/for/master 将本地最新代码（inf）推送至新仓库

git remote rm origin


git log --name-status
or

git log --name-only
or

git log --stat

git reflog

mvn test -Dtest=TestQueryLogger -DfailIfNoTests=alse
mvn cobertura:cobertura
-Dmaven.test.skip=true

git remote show origin
git remote update
// checkout 远程分支
git checkout -b hst-master hive/hst
// push 远程分支
git push hive HEAD:refs/for/hst
