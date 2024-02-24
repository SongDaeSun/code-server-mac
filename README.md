# code-server-mac
MAC OS에서 https로 보안이 된 code-server를 띄워서 언제 어디서나 web에 접속해 MAC에서 코딩을 해보자.


## code-server 설치 및 설정
다음의 명령어로 code-server 설치
```
brew install code-server
```

### ubuntu code-server install
```
curl -fsSL https://code-server.dev/install.sh | sh
```

code-server 명령어로 code-server 실행여부 파악.  
이를 해줘야 code-server config 파일이 생성된다.
```
code-server
```
localhost:8080으로 접속하여 잘 작동되는지를 우선 확인한다.  


code-server의 config파일 수정
```
vim .config/code-server
```
다음과 같이 수정한다.
```
bind-addr: 0.0.0.0:8080
auth: password
password: {your password}
cert: false
```
bind-addr에 0.0.0.0을 넣어줌으로서 원격 어디에서도 문제없이 접속할 수 있도록 해준다.  
cert는 nginx와 letsencrypt로 설정할 것이기에 false로 남겨둔다.


## nginx 설치 및 설정
다음의 명령어로 nginx
```
brew uninstall --force nginx
brew install nginx
```

### ubuntu nginx tjfcl
```
sudo apt update
sudo apt install nginx
```

아래의 명령어로 nginx.conf를 열고,  
server가 활성화 된 부분에 다음의 코드를 이식할 것.
### MAC 기준
```
vim /usr/local/etc/nginx/nginx.conf  
```

### Ubuntu 기준
```
sudo vim /etc/nginx/nginx.conf  
```


```
server {
    listen 80;
    listen [::]:80;
    server_name your.domain; # 발급받은 도메인

    location / {
      proxy_pass http://localhost:8080/; # config.yaml로 설정한 서버의 주소
      proxy_set_header Host $host;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection upgrade;
      proxy_set_header Accept-Encoding gzip;
    }
}
```
server_name에 해당하는 부분의 domain만 수정해두면 된다.  

nginx를 재실행하여서 방금의 수정내용을 적용시킨다.
```
brew services restart nginx
```
localhost:80으로 접속하여 잘 작동되는지를 우선 확인한다.  


## letsencrypt로 인증서 발급받고 nginx에 적용하기
우선 해당하는 PC의 80포트가 개방되었는지 여부를 먼저 보아야 한다.  
80포트가 닫혀있으면 정상적으로 인증서가 발급되지 않을 수 있다.

다음의 명령어로 certbot 설치하기  
```
brew uninstall --force certbot
brew install certbot
```

nginx에 ssl적용하기
```
sudo certbot certonly --nginx
```

콘솔창에서 당신이 설정한 domain이 뜨면 성공이다.  
(여기서는 your.domain)이었다.  
1을 선택해서 엔터를 누르자.  

정상적으로 끝났다면, nginx.conf는 다음과 같이 변경되었을 것이다.
```
#user  nobody;
worker_processes  5;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
    server_name your.domain; # 발급받은 도메인

    location / {
      proxy_pass http://localhost:8080/; # config.yaml로 설정한 서버의 주소
      proxy_set_header Host $host;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection upgrade;
      proxy_set_header Accept-Encoding gzip;
    }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    
    listen [::]:443 ssl ipv6only=on; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/your.domain/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/your.domain/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}
    include servers/*;


    server {
    if ($host = your.domain) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


        listen 80;
         listen [::]:80;
    server_name your.domain;
    return 404; # managed by Certbot


}}

```

여기까지 완료되었다면, 다시 nginx를 재기동시키자.
```
brew services restart nginx
```

https로 접속하여 정상적으로 작동되는지를 확인하자.  

## crontab을 활성화시켜 인증서 자동 갱신하기


