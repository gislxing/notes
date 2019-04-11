# 设置mysql允许外部连接访问

## 命令行登陆mysql

1. 进入mysql安装的bin目录

   Mac 的目录一般在：/usr/local/mysql/bin

2. 输入: mysql -u root -p

3. 登陆后输入: update user set host = ‘%’ where user =’root’; 

4. flush privileges; 

5. Quit

