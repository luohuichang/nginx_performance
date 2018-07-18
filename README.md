# openresty

	sudo yum install yum-utils
    sudo yum-config-manager --add-repo https://openresty.org/package/centos/openresty.repo
    sudo yum install openresty

# webserver

https://eggggger.xyz/2016/10/17/why-http2/

http://wiki.jikexueyuan.com/project/openresty/test/performance_test.html

https://www.jianshu.com/p/e5aef185f0c6

# 可用性判断
cd ~/nginx_test
curl http://localhost:1080
curl -k https://localhost:1443
curl -k --cert ./conf/client/client.crt --key ./conf/client/client.key https://localhost:2443

# 性能测试

# 测试工具 wrk

[root@centos ~]#  cd /usr/local/src
[root@centos ~]#  yum install git -y
[root@centos ~]#  git clone https://github.com/wg/wrk.git
[root@centos ~]# cd wrk
[root@centos ~]# make
[root@centos ~]# ln -s /usr/local/src/wrk/wrk /usr/local/bin

url=114.116.75.83
wrk -t 2 -c 50 -d 20 --latency http://$url:1080

wrk -t 2 -c 50 -d 20 --latency http://$url:1443

wrk -t 2 -c 50 -d 20 -C ./conf/client/client.pem -K ./conf/client/client.key --latency http://$url:2443
wrk -t1 -c1 -d3 -C ./conf/client/client.pem -K ./conf/client/client.key --latency https://$url:2443

参数说明：
-t 需要模拟的线程数
-c 需要模拟的连接数
-d 测试的持续时间
--timeout 超时的时间
--latency 显示延迟统计
结果显示说明：
Latency：响应时间
Req/Sec：每个线程每秒钟的完成的请求数
Avg：平均
Max：最大
Stdev：标准差
HTTPS WebServer测试性能数据
连接类型	Session cache	包体(bytes)	加密套件	性能(qps)
长	打开	230	ECDHE-RSA-AES128-GCM-SHA256	296241
短	关闭	230	ECDHE-RSA-AES128-GCM-SHA256	65630


This changes allows one to apply a client SSL certificate in the PEM format:
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
Please use the '-C' parameter as follow:
wrk -t 1 -c 1 -C cert.pem ...

来自 <https://github.com/wg/wrk/pull/222> 

来自 <https://cloud.tencent.com/document/product/214/5983> 


# 证书生成
## server证书
name=server
openssl genrsa -des3 -out $name.key 1024

openssl req -new -key $name.key -out $name.csr 

cp $name.key $name.key.org
openssl rsa -in $name.key.org -out $name.key

openssl x509 -req -days 365 -in $name.csr -signkey $name.key -out $name.crt
## client证书
name=client
……

证书编码的转换
PEM转为DER openssl x509 -in cert.crt -outform der -out cert.der
DER转为PEM openssl x509 -in cert.crt -inform der -outform pem -out cert.pem
openssl x509 -in mycert.crt -out mycert.pem -outform PEM

(提示:要转换KEY文件也类似,只不过把x509换成rsa,要转CSR的话,把x509换成req...)


# Nginx管理

nginx -p ~/nginxconf/
nginx -c ~/conf/nginx.conf

kill -QUIT 主进程号
kill -信号类型 '/usr/nginx/logs/nginx.pid'

nginx -c ~/conf/nginx.conf -s reload  --重启
nginx -t -c /usr/nginx/conf/nginx.conf --检查

# Nginx配置

worker_processes  1;        #nginx worker 数量
error_log logs/error.log;   #指定错误日志文件路径
events {
    worker_connections 1024;
}

http {
    server {
        listen 127.0.0.1:1080;

        error_log /home/lhc/wrktest/logs/error_http.log warn;
        access_log /home/lhc/wrktest/logs/access_http.log main;

        location / {
            default_type text/html;

            content_by_lua_block {
                ngx.say("Http: HelloWorld")
            }
        }
    }

	server {
		listen 127.0.0.1:1443 ssl;
		server_name testssl.com;

        error_log /home/lhc/wrktest/logs/error_https.log warn;
        access_log /home/lhc/wrktest/logs/access_https.log main;



		ssl_certificate /home/lhc/wrktest/conf/server.crt;
		ssl_certificate_key /home/lhc/wrktest/conf/server.key;

        location / {
            default_type text/html;

            content_by_lua_block {
                ngx.say("Https: HelloWorld")
            }
        }
	}

    server {
        listen 127.0.0.1:2443 ssl;
        server_name testssl2.com;

        error_log /home/lhc/wrktest/logs/error_https2.log warn;
        access_log /home/lhc/wrktest/logs/access_https2.log main;

        ssl_certificate /home/lhc/wrktest/conf/server.crt;
        ssl_certificate_key /home/lhc/wrktest/conf/server.key;

        ssl_client_certificate /home/lhc/wrktest/conf/client/client.pem;
        ssl_verify_client on;

        location / {
            default_type text/html;

            content_by_lua_block {
                ngx.say("Https2: HelloWorld")
            }
        }
    }    
}
