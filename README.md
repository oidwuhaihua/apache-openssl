一、Apache TLS 1.0 升级 TLS 1.2 问题 。

 ```cpp
 
1、最初版本 
  A )   OpenSSL 0.9.8e-fips-rhel5
        -bash-3.2$ openssl version -a
        OpenSSL 0.9.8e-fips-rhel5 01 Jul 2008
        built on: Wed Jan 18 10:10:45 EST 2012
        platform: linux-x86_64
        options:  bn(64,64) md2(int) rc4(ptr,int) des(idx,cisc,16,int) blowfish(ptr2) 
        compiler: gcc -fPIC -DOPENSSL_PIC -DZLIB -DOPENSSL_THREADS -D_REENTRANT -DDSO_DLFCN -DHAVE_DLFCN_H -DKRB5_MIT -       
        I/usr/kerberos/include -DL_ENDIAN -DTERMIO -Wall -DMD32_REG_T=int -O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -
        fstack-
        protector --param=ssp-buffer-size=4 -m64 -mtune=generic -Wa,--noexecstack -DOPENSSL_USE_NEW_FUNCTIONS -fno-strict-aliasing -
        DOPENSSL_BN_ASM_MONT -DSHA1_ASM -DSHA256_ASM -DSHA512_ASM -DMD5_ASM -DAES_ASM
        OPENSSLDIR: "/etc/pki/tls"
        engines:  dynamic 
   B ) Apache 版本
        [root@mtest conf]# /usr/local/apache3/bin/apachectl -version -f /usr/local/apache3/conf/httpd.conf
        Server version: Apache/2.2.26 (Unix)
        Server built:   Nov 15 2016 15:50:10
        
   C ) 下载 最新版本的openssl and compile (官方描述只有1.0以上版本才支TLS1.2,http://httpd.apache.org/docs/2.2/mod/mod_ssl.html#sslprotocol)
       
       1、wget http://www.openssl.org/source/openssl-1.0.1g.tar.gz
       2、tar解压 cd 到 openssl 目录下编译
       3、./config  --prefix=/usr/local/openssl --openssldir=/usr/local/ssl (注意：不需要替换之前版本的openssl 文件，放到新的目录 避免发生不 
       可预料的问题。)
       4、检查版本
          [root@mtest conf]# openssl.bak version
          OpenSSL 1.0.1g 7 Apr 2014
    D ) 重新编译Apache  
        cd 到 apache 目录下执行如下命令：
        1、./configure --prefix=/usr/local/apache3 --enable-so  --enable-proxy --enable-proxy-connect --enable-proxy-http --enable-  
        proxy-scgi --enable-proxy-ajp --enable-proxy-balancer --enable-rewrite --enable-ssl --with-ssl=/usr/local/openssl/ --enable-cgi         --with-apr=/usr/local/apr --with-apr-util=/usr/local/apr-util/
        2、make
        3、make install
     E ) 通过openssl 生成证书（认证流程：通过openssl 会生成一个service.key文件，通过service.key文件可以生成service.crt文件，crt文件会上送到证书          公司申请ca证书认证）注意：如果自己之前有证书 并且通过第三方公司认证是支持TLS 1.2的 直接拿service.key service.crt当相对路径，再通过
        httpd-ssl-conf 配置即可，以下生成的key crt 用于测试 。
         1、cd 到 /usr/local/apache3/conf 目录
         2、执行：/usr/local/openssl/bin/openssl genrsa -out server.key 2048
         3、执行：/usr/local/openssl/bin/openssl req -new -key server.key -out server.csr
                You are about to be asked to enter information that will be incorporated
                into your certificate request.
                What you are about to enter is what is called a Distinguished Name or a DN.
                There are quite a few fields but you can leave some blank
                For some fields there will be a default value,
                If you enter '.', the field will be left blank.
                -----
                Country Name (2 letter code) [AU]:CH
                State or Province Name (full name) [Some-State]:china
                Locality Name (eg, city) []:rebby
                Organization Name (eg, company) [Internet Widgits Pty Ltd]:rebby
                Organizational Unit Name (eg, section) []:rebby
                Common Name (e.g. server FQDN or YOUR name) []:mtest.xxx.com 注意：输入域名地址
                Email Address []:haihua@163.com

                Please enter the following 'extra' attributes
                to be sent with your certificate request
                A challenge password []:   注意：密码直接不要输入 回车。
                An optional company name []:rebby
         4、执行： /usr/local/openssl/bin/openssl x509 -req -days 3650 -in server.csr -signkey server.key -out server.crt
                Signature ok
                subject=/C=CH/ST=china/L=rebby/O=rebby/OU=rebby/CN=mtest.xxx.com/emailAddress=haihua@163.com
                Getting Private key
      F ) 更改配置
         1、vim /usr/local/apache3/conf/httpd.conf 
             #Include conf/extra/httpd-ssl.conf 注意：找到这行把 注释去掉 
         2、vim /usr/local/apache3/conf/extra/httpd-ssl.conf 注意：配置你自己的域名映射 即可
            <VirtualHost _default_:443>
            ServerName mtest.xxx.com
            ProxyPass /          http://xx.xx.xxx.11:7001/MBSServer/
            ProxyPassReverse  /          http://xx.0.223.11:7001/MBSServer/
                
      G ) 启动Apache 
         1、[root@mtest bin]# /usr/local/apache3/bin/apachectl start 注意：执行的时候可能导致文件找不到，我们可以换种方式执行。
            httpd: Could not open configuration file /usr/local/apache/conf/httpd.conf: No such file or directory
         2、 执行： /usr/local/apache3/bin/apachectl -k stop -f /usr/local/apache3/conf/httpd.conf
            [root@mtest bin]# ps -ef | grep apache
            root     24885     1  0 16:15 ?        00:00:00 /usr/local/apache3/bin/httpd -k start -f /usr/local/apache3/conf/httpd.conf
            daemon   24886 24885  0 16:15 ?        00:00:00 /usr/local/apache3/bin/httpd -k start -f /usr/local/apache3/conf/httpd.conf
            daemon   24887 24885  0 16:15 ?        00:00:00 /usr/local/apache3/bin/httpd -k start -f /usr/local/apache3/conf/httpd.conf
            daemon   24888 24885  0 16:15 ?        00:00:00 /usr/local/apache3/bin/httpd -k start -f /usr/local/apache3/conf/httpd.conf
            daemon   24889 24885  0 16:15 ?        00:00:00 /usr/local/apache3/bin/httpd -k start -f /usr/local/apache3/conf/httpd.conf
            daemon   24890 24885  0 16:15 ?        00:00:00 /usr/local/apache3/bin/httpd -k start -f /usr/local/apache3/conf/httpd.conf
            daemon   24891 24885  0 16:15 ?        00:00:00 /usr/local/apache3/bin/httpd -k start -f /usr/local/apache3/conf/httpd.conf
            daemon   24896 24885  0 16:17 ?        00:00:00 /usr/local/apache3/bin/httpd -k start -f /usr/local/apache3/conf/httpd.conf
            root     25302  7428  0 18:54 pts/2    00:00:00 grep apache 注意：可以看的出进程已经启动了。
            
         3、测试 google浏览器测试 F12 --> Security 可以看出已经支持TLS 1.2了 ，也可以通过第三方网站测试（https://www.ssllabs.com/ssltest/）
         
         The connection to this site is encrypted and authenticated using a strong protocol (TLS 1.2), a strong key exchange      
         (ECDHE_RSA), and a strong cipher (AES_128_GCM).
         
         编译了很多次遇到很多问题，最终上面这个版本编译成功。过程中遇到的问题如下所示：
         
         Cannot load /usr/local/apache/modules/mod_ssl.so into server: /usr/local/apache/modules/mod_ssl.so: undefined symbol: 
         EC_KEY_free 
         注意：编译成功后启动提示 mod_ssl.so已经被内置加载，不需要重新加载wiki有各种解释，不要纠结重新编译。
         
         configure: error: ... Error, SSL/TLS libraries were missing or unusable 
         通过如下命令解决：export LDFLAGS=-ldl
         
         Syntax error on line 56 of /usr/local/apache/conf/extra/httpd-ssl.conf:
         Invalid command 'SSLPassPhraseDialog', perhaps misspelled or defined by a module not included in the server   
         configuration
         注意：没去纠结重新编译Apache
         
         undefined symbol: ssl_cmd_SSLMutex 
         解决方法：/usr/local/apache/bin/apxs -i -c -a -D HAVE_OPENSSL=1 -I /usr/include/openssl *.c
         
         undefined symbol: X509_INFO_free 
         解决方法：undefined symbol: X509_INFO_free
         
         mod_ssl.so: undefined symbol: EC_KEY_free 注意：没去纠结，重新编译。 
         
         
         一些Apache 命令使用：
         
           /usr/local/apache3/bin/apachectl -k stop -f /usr/local/apache3/conf/httpd.conf 停止

           /usr/local/apache3/bin/apachectl -k start -f /usr/local/apache3/conf/httpd.conf 启动

           /usr/local/apache3/bin/apachectl -k configtest -f /usr/local/apache3/conf/httpd.conf 检查所有配置文件是否有问题

           cat /usr/local/apache3/build/config.nice 查看Apache 加载的内置模板

           /usr/local/apache/bin/apxs -c -i mod_ssl.c 重新编译Apache 某个源文件到modules目录
            

       
       
       
       
       
       
       
