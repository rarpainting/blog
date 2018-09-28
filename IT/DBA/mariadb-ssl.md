# MariaDB SSL 开启

GRANT ALL PRIVILEGES ON *.* TO 'user'@'host' IDENTIFIED BY 'password' REQUIRE SSL; 
在通过 SSL 认证下, 赋予某用户(user:password@host)所有权限(*.*)
