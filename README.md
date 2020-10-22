# Progress

---

持續更新內容中.....

- [x]  架構圖
- [ ]  服務安裝筆記
    - [x]  Zabbix-Server
    - [ ]  Nginx
    - [ ]  PHP
    - [ ]  MariaDB
- [ ]  代碼筆記

# Introduction

---

## Architecture

![Topology](Architecture.png)

近年資訊業的蓬勃發展下，很多工具或是雲平台如雨後春筍般的出現，像是GCP、Azure、阿里雲、騰訊雲等等平台或像是Ansible、Terraform、SaltStack這種Infrastructure As Code的配置管理工具，而每一家公司至少也都會使用一至多個雲平台，並且當一個雲平台下的帳號不只一個，帳號管理數量一多，常常會發生一些奇葩的事情，項是帳號密碼被某個工程師修改了，但是卻沒有讓到團隊成員知道，或者是某個服務器遷移至某個帳號下管理，團隊內部資訊同樣沒有同步，造成其他成員嘗試多個帳號後才發現已經遷移，而有些團隊很多時候都在這種事情上浪費人力，而這種情況需要被解決，透過一些文章了解ITIL裡面提到的CMDB概念 : 配置管理資料庫(CMDB)是與IT系統所有組件相關的信息庫。它包含IT基礎架構配置項的詳細信息，才衍生出了這個想法，讓團隊成員能夠透過CMDB集中化管理資產訊息、配置等等，並且結合常見的Devops Tools整合所有雲平台內容資產訊息，而團隊成員只需要專注在CMDB上進行機器或者配置的新增、刪除、修改、更新，而不需要再耗費其他時間去熟悉雲平台配置，讓其他成員更有時間去專注在他們的專業領域上。

## Component

使用原因

- Terraform : 透過Terraform將雲平台的API封裝，不需要了解API的實際操作方式就能對雲平台操作(新增、刪除、Blabla)，未來當平台切換時也能夠抽換到對應雲平台的Provider。
- Proxmox VE : 恩...因為我沒錢(ಥ﹏ಥ)，剛好這套免費虛擬化技術安裝簡易Terraform又有支援。
- CMDB : 想透過做中學的方式學習Python Django框架。
- Jenkins : 將每一次Ansible執行結果紀錄 ( 想讓CMDB專職管理資產等等內容，而一些特殊客製化內容透過Jenkins pipeline再進行定義 )。
- Ansible : Terraform作為機器初始化的配置管理，而Ansible作為業務層面的配置管理(例如安裝GO、Java、Python)，ython)，另外個人較偏好於使用Agentless方式管理服務器，並且Ansible Galaxy上也有提供許多的Role能夠使用，秉持著Stop Trying to Reinvent the Wheel的精神。
- Zabbix : 服務器建立後可以監控相關資源，透過Ansible配置，一方面也是能夠學習監控領域。


# Installation

---

## Component

### Zabbix-Server

組成元件

- Nginx
- Zabbix-Server 5.0.4
- MySQL > 【5.5.62 - 8.0.x】  ( 如果MySQL用作Zabbix後端數據庫。 InnoDB引擎是必需的。 MariaDB也能運作在Zabbix之上。)
- PHP > 7.2.0 (PHP在 5.3.3 之後已經將php-fpm寫入php原始碼核心內了。所以已经不需要另外編譯安裝了)

在編譯檔案之前，建議先將基礎編譯所需工具安裝。

```bash
[root@zabbix-server ~]# yum groupinstall -y "Development Tools"
```

共用配置

- Source Code擺放目錄 : `/usr/local/src/`

正式安裝，安裝分為四個部分

- MariaDB ( or MySQL )
- PHP
- Nginx
- Zabbix-Server

MariaDB ，此部分安裝使用最簡單的Yum套件管理工具進行安裝

版本

- 10.1

相關設置

- 建立DB，名為zabbix，設置編碼格式為`utf8`以及collate為`utf8_bin`。
- 建立zabbix用戶，並給予zabbix用戶存取zabbix DB的所有存取權限。

建置步驟

1. 新增MariaDB的Yum Repository文件

    ```bash
    [root@zabbix-server ~]# cat << EOF > /etc/yum.repos.d/MariaDB.repo
    # MariaDB 10.1 CentOS repository list - created 2017-01-27 16:31 UTC
    # http://downloads.mariadb.org/mariadb/repositories/
    [mariadb]
    name = MariaDB
    baseurl = http://yum.mariadb.org/10.1/centos7-amd64
    gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
    gpgcheck=1
    EOF
    [root@zabbix-server ~]# 
    ```

2. 安裝`MariaDB`以及`Zabbix所需的MySQL函式庫`

    ```bash
    [root@zabbix-server ~]# yum install -y MariaDB-server MariaDB-client MariaDB-devel MariaDB-shared
    ```

3. 啟動服務以及設置開機啟動

    ```bash
    [root@zabbix-server ~]# systemctl start mariadb.service
    [root@zabbix-server ~]# systemctl enable mariadb.service
    ```

4. 設定MariaDB安全性設置腳本(預設root密碼為空) , 範例透過腳本設定成111111 。

    ```bash
    [root@zabbix-server ~]# mysql_secure_installation
    ```

5. 建立Zabbix資料庫以及其用戶

    ```bash
    # 建立用戶
    MariaDB [(none)]> create database zabbix character set utf8 collate utf8_bin;

    # 給予權限
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON zabbix.* to 'zabbix'@'%' IDENTIFIED  BY '123456' WITH GRANT OPTION;
    ```

### Nginx

使用編譯方式進行安裝。

版本

- 1.14.0

目錄設置

- 主目錄 : `/usr/local/nginx/`
- 日誌目錄 : `/usr/local/nginx/logs`
- 配置目錄 : `/usr/local/nginx/conf/`
    - vhosts目錄 : `/usr/local/nginx/conf/vshos/`

建置步驟

1. 下載`Nginx source code`

    ```bash
    [root@zabbix-server src]# wget http://nginx.org/download/nginx-1.14.0.tar.gz
    ```

2. 切換目錄到`Nginx源碼目錄`内

    ```bash
    [root@zabbix-server src]# cd nginx-1.14.0
    ```

3. 下載`Zlib`模塊(gzip依賴模塊)

    ```bash
    [root@zabbix-server nginx-1.14.0]# wget http://zlib.net/zlib-1.2.11.tar.gz
    ```

4. 解壓縮`Zlib`模塊 , 切換至目錄內

    ```bash
    [root@zabbix-server nginx-1.14.0]# tar -xvf zlib-1.2.11.tar.gz && cd zlib-1.2.11
    ```

5. 編譯

    ```bash
    [root@zabbix-server zlib-1.2.11]# ./configure 
    [root@zabbix-server zlib-1.2.11]# make 
    [root@zabbix-server zlib-1.2.11]# make install
    ```

6. 回 Nginx 目錄下 , 下載`PCRE`模塊(rewrite依賴)

    ```bash
    [root@zabbix-server zlib-1.2.11]# cd ../ && wget ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.42.zip
    ```

7. 解壓縮`PCRE`, 切換到該目錄下

    ```bash
    [root@zabbix-server nginx-1.14.0]# unzip pcre-8.42.zip && cd pcre-8.42
    ```

8. 編譯

    ```bash
    [root@zabbix-server pcre-8.42]# ./configure 
    [root@zabbix-server pcre-8.42]# make 
    [root@zabbix-server pcre-8.42]# make install
    ```

9. 回上一層目錄開始編譯Nginx , 下列為參數用途說明
    - `--prefix指定安装路徑--with-http_ssl_module`指定安裝https模塊
    - `--with-pcre 指定安装PCRE源码路徑`(rewrite依賴)
    - `--with-zlib 指定安装zlib源码路徑`(gzip依賴)。

    ```bash
    #回上層Nginx目錄
    [root@zabbix-server pcre-8.42]# cd ..

    #過程中出現./configure: error: SSL modules require the OpenSSL library.
    #安裝openssl模塊
    [root@zabbix-server nginx-1.14.0]# yum install -y openssl openssl-devel
    #建立Makefile(設置編譯參數)
    ./configure  \
    --prefix=/usr/local/nginx \
    --with-http_ssl_module \
    --with-pcre=./pcre-8.42 \
    --with-zlib=./zlib-1.2.11 

    #開始編譯Makefile
    [root@zabbix-server nginx-1.14.0]# make 

    #安裝
    [root@zabbix-server nginx-1.14.0]# make install
    ```

10. 建立vhosts目錄，方便往後管理

    ```bash
    [root@zabbix-server nginx-1.14.0]# mkdir -p /usr/local/nginx/conf/vhosts/
    ```

11. 配置Zabbix Vitual Hosting，增加配置後需要進行Reload

    ```bash
    [root@zabbix-server nginx-1.14.0]# vim /usr/local/nginx/conf/nginx.conf 
    # 在主配置內引入vhosts配置
    include /usr/local/nginx/conf/vhosts/*.conf;

    # 在加入下列配置到vhosts，子配置檔可以命名為zabbix.conf，依業務名稱容易辨識即可。
    server {
        listen 80;
        server_name zabbix-server.mytest.com;
    		index index.php;

    		# Zabbix UI放置路徑
        root /opt/zabbix-webui/;
        
    		# 後端動態請求轉到後端PHP
        location ~ \.php$ { 
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }

    }
    ```

12. 設置開機啟動

    ```
    [root@zabbix-server ~]# cat << EOF > /lib/systemd/system/nginx.service
    [Unit]
    Description=nginx
    After=network.target
      
    [Service]
    Type=forking
    ExecStart=/usr/local/nginx/sbin/nginx
    ExecReload=/usr/local/nginx/sbin/nginx -s reload
    ExecStop=/usr/local/nginx/sbin/nginx -s quit
    PrivateTmp=true
      
    [Install]
    WantedBy=multi-user.target

    EOF

    ```

13. 設置開機啟動和啟動服務

    ```bash
    [root@zabbix-server ~]# systemctl start nginx.service
    [root@zabbix-server ~]# systemctl enable nginx.service
    ```

### PHP

使用編譯方式進行安裝。

版本

- PHP 7.2.34

目錄設置

- 主目錄 : `/usr/local/php/`
- 日誌目錄 :  `/var/log/php/`
- 配置目錄 :  `/usr/local/php/etc/`

1. 安裝PHP 前端需要的庫

    ```
    [root@zabbix-server ~]# yum install -y libxml2 libxml2-devel libjpeg-devel libpng-devel freetype-devel
    ```

2. 下載後解壓進入目錄

    ```
    [root@zabbix-server src]# wget https://www.php.net/distributions/php-7.2.34.tar.bz2
    [root@zabbix-server src]# tar -xvf php-7.2.34.tar.bz2 && cd php-7.2.34
    ```

3. 編譯PHP

    ```
    # 設置編譯參數
    # PHP5使用--with-mysql , PHP7使用--with-pdo-mysql

    [root@zabbix-server php-7.2.34]# ./configure --prefix=/usr/local/php \
    --with-pdo-mysql \
    --enable-fpm \
    --with-config-file-path=/usr/local/php/etc \
    --with-zlib \
    --enable-bcmath \
    --enable-mbstring \
    --enable-sockets \
    --with-gettext \
    --with-jpeg-dir \
    --with-png-dir \
    --with-freetype-dir \
    --with-gd

    # 編譯及安裝
    [root@zabbix-server php-7.2.34]# make && make install
    ```

4. 安裝編譯mysqli的PHP擴充套件。

    新版本的PHP(7以上)需要`mysqli`支持 , mysqli是提供給PHP使用的呼叫mysql所用的函數 , 在PHP進行配置時就無法正常使用PHP存取資料庫 , 必須安裝。

    ```bash
    [root@zabbix-server php-7.2.34]# yum install -y autoconf

    # 切換到source code下的ext/mysqli之下
    [root@zabbix-server php-7.2.34]# cd ext/mysqli

    # 使用phpize生成configure
    [root@zabbix-server mysqli]# /usr/local/php/bin/phpize

    # 建立Makefile , 指定PHP下的php-config配置
    [root@zabbix-server mysqli]# ./configure \
    --with-php-config=/usr/local/php/bin/php-config \
    --with-mysqli

    # 編譯及安裝
    [root@zabbix-server php-7.2.34]#  make && make install

    ```

5. PHP配置

    ```bash
    # 配置php.ini文件(Source Code內的php.ini-production 複製到 PHP配置檔案目錄下)
    [root@zabbix-server php-7.2.34]# cp php.ini-production /usr/local/php/etc/php.ini

    # 加上Zabbix系統要求配置
    [root@zabbix-server php-7.2.34]# vim /usr/local/php/etc/php.ini
    max_execution_time = 300
    post_max_size = 16M
    max_input_time = 300
    date.timezone = "Asia/Taipei"
    always_populate_raw_post_data = -1
    extension=mysqli.so

    # 配置php-fpm.ini文件
    [root@zabbix-server php-7.2.34]# cd /usr/local/php/etc/

    # 複製php-fpm配置範本
    [root@zabbix-server etc]# cp php-fpm.conf.default php-fpm.conf

    # 將pid註解掉 , 啟動PHP-FPM時將會產生pid檔案
    [root@zabbix-server etc]# sed -i 's/;pid = run\\/php-fpm.pid/pid = run\\/php-fpm.pid/g' php-fpm.conf
    ```

6. 啟動PHP以及配置PHP開機啟動

    ```
    [root@zabbix-server ~]# cat << EOF > /lib/systemd/system/php-fpm.service

    [Unit]
    Description=php
    After=network.target

    [Service]
    Type=forking
    ExecStart=/usr/local/php/sbin/php-fpm
    ExecStop=/bin/pkill -9 php-fpm
    PrivateTmp=true
    [Install]
    WantedBy=multi-user.target

    EOF

    [root@zabbix-server ~]# systemctl start php-fpm.service

    ```

### Zabbix-Server

版本

- Zabbix Server 5

目錄設置

- 主目錄 : `/usr/local/zabbix/`
- 配置目錄 :  `/usr/local/zabbix/etc/`
- 日誌目錄 :  `/usr/local/zabbix/log/`

1. 安裝依賴套件

    ```
    [root@zabbix-server ~]# yum -y install net-snmp net-snmp-devel perl-DBI php-gd php-xml php-bcmath fping OpenIPMI-devel php-mbstring mysql-devel curl-devel
    ```

2. 下載源碼 , 並切換至該目錄

    ```
    [root@zabbix-server ~]# wget https://cdn.zabbix.com/zabbix/sources/stable/5.0/zabbix-5.0.4.tar.gz
    [root@zabbix-server ~]# tar -xvf zabbix-5.0.4.tar.gz && cd zabbix-5.0.4
    ```

3. 建立Zabbix用戶及Group

    ```
    [root@zabbix-server ~]# groupadd zabbix
    [root@zabbix-server ~]# useradd zabbix -g zabbix
    ```

4. 編譯及安裝
參數說明:
    - `--prefix` : 指定安裝目錄
    - `--enable-server` : 安裝Zabbix-Server
    - `--enable-agent` : 安裝Zabbix agent (監控程序)
    - `--with-mysql` : 使用mysql當作後端資料庫
    - `--with-net-snmp` : 啟動SNMP功能
    - `--with-libcurl` : 為了支持SMTP認證 , 用於發送告警郵件需要
    - `--with-libxml2` : 不清楚

    ```
    #過程中如果出現configure: error: Unable to use libevent (libevent check failed)
    [root@zabbix-server ~]# yum install -y libevent libevent-devel

    # 編譯Zabbix-Server
    [root@zabbix-server ~]# ./configure \
    --prefix=/usr/local/zabbix \
    --enable-server  \
    --enable-agent \
    --with-mysql \
    --with-net-snmp \
    --with-libcurl \
    --with-libxml2 

    #編譯及安裝
    [root@zabbix-server ~]# make install
    ```

5. 設置Zabbix目錄權限及配置Zabbix啟動腳本

    ```
    # 變更權限
    [root@zabbix-server ~]# chown -R zabbix:zabbix /usr/local/zabbix/

    # 配置啟動腳本 (For CentOS6)
    [root@zabbix-server ~]# cp /root/zabbix-4.0.1/misc/init.d/fedora/core/zabbix_server /etc/init.d/

    # 配置啟動腳本 (For CentOS 7)
    [root@zabbix-server ~]# cat << EOF > /usr/lib/systemd/system/zabbix-server.service

    [Unit]
    Description=Zabbix Server
    After=syslog.target
    After=network.target

    [Service]
    User=zabbix
    Group=zabbix
    Environment="CONFFILE=/usr/local/zabbix/etc/zabbix_server.conf"
    EnvironmentFile=-/usr/local/zabbix/etc/zabbix_server.conf
    Type=forking
    Restart=on-failure
    PIDFile=/run/zabbix/zabbix_server.pid
    KillMode=control-group
    ExecStart=/usr/local/zabbix/sbin/zabbix_server -c \$CONFFILE
    ExecStop=/bin/kill $MAINPID
    RestartSec=10s
    TimeoutSec=120s

    [Install]
    WantedBy=multi-user.target

    EOF

    [root@zabbix-server ~]# mkdir -p /run/zabbix/
    [root@zabbix-server ~]# chown -R zabbix:zabbix /run/zabbix/

    [root@zabbix-server ~]# mkdir -p /usr/local/zabbix/log/
    [root@zabbix-server ~]# chown -R zabbix:zabbix /usr/local/zabbix/log/

    # 將zabbix做一個軟鏈接 , 以便於執行命令以及提供啟動腳本所調用
    [root@zabbix-server ~]# ln -s /usr/local/zabbix/sbin/zabbix_server /usr/local/sbin/zabbix_server
    ```

6. 配置Zabbix Server文件

    ```
    vim /usr/local/zabbix/etc/zabbix_server.conf 

    # 請依照自身業務配置 , 此為範例
    ListenPort=10051

    # 需要事先建立日誌目錄，否則會出現錯誤
    LogFile=/usr/local/zabbix/log/zabbix_server.log

    # 啟動後程序PID
    PidFile=/run/zabbix/zabbix_server.pid

    # DB連接地址
    DBHost=localhost
    # DB名稱
    DBName=zabbix
    # 連接資料庫用戶名稱
    DBUser=zabbix          
    # 連接資料庫用戶密碼
    DBPassword=123456      

    # Zabbix-Server監聽IP
    ListenIP=0.0.0.0

    # 是否允許使用root身份運行zabbix，如果值為0，並且是在root環境下
    # zabbix會嘗試使用zabbix用戶運行，如果不存在會告知zabbix用戶不存在。
    AllowRoot=0
    User=zabbix
    ```

7. 將數據表導入MariaDB

    1. 先建立名稱為zabbix的數據庫

        ```
        # 進入數據庫
        mysql -u root -p 123456

        #建立zabbix數據庫
        MariaDB [(none)]> create database zabbix default character set utf8;
        ```

    2. 導入數據表

        ```
        [root@zabbix-server zabbix-4.0.1]# cd database/mysql

        # 依序導入及輸入密碼
        [root@zabbix-server mysql]#  mysql -u root -p zabbix < schema.sql      
        Enter password:
        [root@zabbix-server mysql]#  mysql -u root -p zabbix < images.sql 
        Enter password:
        [root@zabbix-server mysql]#  mysql -u root -p zabbix < data.sql 
        Enter password:

        # 檢查Zabbix資料庫是否有產生數據表
        [root@zabbix-server mysql]# mysql -u root -p -e "use zabbix; show tables"
        ```

8. 建立Zabbix Server 日誌路徑以及PID路徑

    ```
    [root@zabbix-server ~]# mkdir -p /usr/local/zabbix/log 
    [root@zabbix-server ~]# mkdir -p /run/zabbix/

    # 注意權限
    [root@zabbix-server ~]# chown zabbix:zabbix /usr/local/zabbix/log 
    [root@zabbix-server ~]# chown zabbix:zabbix /run/zabbix/

    [root@zabbix-server ~]# /etc/init.d/zabbix_server start

    # 錯誤訊息Starting zabbix_server:  /etc/init.d/functions: line 722: /usr/local/sbin/zabbix_server: No such file or directory
    # 原因是找不到執行檔案 , 確認有做軟鏈結

    # 啟動過程中遇到錯誤/usr/local/sbin/zabbix_server: error while loading shared libraries: libpcre.so.1: cannot open shared object file: No such file or directory
    ln -s /usr/local/lib/libpcre.so.1 /lib64/
    ```

9. 配置Zabbix WebUI頁面 , 將Zabbix Source Code下的frontends/php內的資源檔案副
    1. 將source code 下的frontends/php內的檔案放置

        ```
        [root@zabbix_sever_5 ~]# mkdir /opt/zabbix-webui/
        [root@zabbix_sever_5 ~]# cd /opt/zabbix-webui/
        [root@zabbix_sever_5 zabbix-webui]# cp -R /usr/local/src/zabbix-5.0.4/ui/* .
        cp -a . /usr/local/nginx/html/zabbix/
        ```

10. 啟動Nginx 以及php-fpm

    ```
    service nginx start
     /etc/init.d/php-fpm start
    ```

11. 訪問Zabbix頁面 , 出現Access denied , 排查nginx error.log

    ```
    2018/11/23 21:19:46 [error] 55348#0: *1 FastCGI sent in stderr: "Access to the script '/usr/local/nginx/html/zabbix' has been denied (see security.limit_extensions)" while reading response header from upstream, client: 192.168.247.1, server: localhost, request: "GET /zabbix HTTP/1.1", upstream: "fastcgi://127.0.0.1:9000", host: "192.168.247.150"
    ```

    由於 php-fpm 預設安裝在 security.limit_extensions 是僅有支援 .php 的 , 若是你的 Web site 有執行 .htm .html .pl 等等的副檔名就會被擋下來

    - 解決方法為修改php.ini , 將security.limit_extensions註解取消重啟服務

        ```
        security.limit_extensions = .php .php3 .php4 .php5
        ```

    - 另外一種方式則是在Nginx內使用location將特定的後綴結尾導向後端PHP。

12. 上述錯誤解決 , 但又出現了靜態文件訪問404的問題 , 原因是因為`.jpg`、`.css`、`.js`檔案也被當作是PHP , 透過php-fpm去解析了
    - 解決方法就是在Nginx添加一條規則 , 凡是符合靜態文件結尾的檔案都導向Zabbix目錄下

        ```
        location ~* ^.+\\.(ico|gif|jpg|jpeg|png|html|css|htm|bmp|js|svg)$ {
                   root /usr/local/nginx/html/;
        }
        ```

13. 接著就進行網頁安裝Zabbix服務了，Zabbix會先檢測你目前所有服務是否已經配置正確
如果PHP databases support出現Fail情況，確認你已經安裝好mysqli，也在php.ini內配置extension

    ```
    # 重要! 在Zabbix WEB安裝頁面顯示是否支援是由mysqli.so這個檔案去決定
    # 如果沒有這個檔案則會一直顯示databases support，不管你換了幾個版本的Database
    extension=mysqli.so
    ```

    如果有其他錯誤 , 也是先檢查php.ini文件是否達到需求 , 因為zabbix 是檢測php.ini的

14. 配置完成Zabbix後，記得將setup.php檔案刪除。