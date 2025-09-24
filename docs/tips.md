# Tips

## Ollama

### GPUの読み込みに失敗する

```
time=2025-09-15T05:27:47.102+09:00 level=WARN source=gpu.go:616 msg="unknown error initializing cuda driver library /usr/lib/x86_64-linux-gnu/libcuda.so.575.64.03: cuda driver library init failure: 999. see https://github.com/ollama/ollama/blob/main/docs/troubleshooting.md for more information"
```

対応策: 

```
sudo systemctl stop ollama
sudo rmmod nvidia_uvm
sudo modprobe nvidia_uvm
sudo systemctl start ollama
```


## Nginx

/etc/nginx/sites-availableに、`myapp.conf`を作成する。

```
server {
  listen 80;
  server_name localhost;

  location / {
    proxy_pass http://localhost:5173;
    proxy_set_header Host $host; 
    proxy_set_header X-Real-IP $remote_addr; 
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 
    proxy_set_header X-Forwarded-Proto $scheme; 
  }
}
```

`sites-available/default`へのシンボリックリンクを削除する。

設定の反映

```
$ sudo ln -s /etc/nginx/sites-available/myapp.conf /etc/nginx/sites-enabled/
$ sudo nginx -t # 設定テスト
$ sudo systemctl reloead nginx
```


### HTTPS対応

https化にはLet's encryptを使用する。

sites-available/myapp.conf
```
server {
    listen 80;
    server_name example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com;
    
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_sertificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    
    location / {
        proxy_pass http://localhost:5173;
    }
}
```

## HTTPS対応(特殊ケース)

Google Cloud上で展開するサービスを、ローカルPCから参照したい。
社内のサブドメインを取得するにはすこし手間がかかるから、VMのグローバルIPに紐づけて
httpsで公開できるようにしたい。

**自己証明書の作成**

秘密鍵の作成
```
sudo mkdir -p /etc/ssl/private
sudo chmod 700 /etc/ssl/private
openssl genpkey -algorithm RSA -out /etc/ssl/private/server.key
```

```
openssl req -new -x509 -day 365 -key /etc/nginx/ssl/server.key -out /etc/nginx/ssl/server.crt \
-subj "/CN=localhost" \
-extensions v3_req -extfile /etc/nginx/ssl/server.ext
```


server.ext
```
[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
```


CSRの作成
```
sudo openssl req -new \
  -key /etc/nginx/ssl/server.key \
  -out /etc/nginx/ssl/server.csr \
  -subj "/CN=localhost"
```

証明書を発行
```
sudo openssl x509 -req -days 365 \
  -in /etc/nginx/ssl/server.csr \
  -signkey /etc/nginx/ssl/server.key \
  -out /etc/nginx/ssl/server.crt \
  -extensions v3_req -extfile /etc/nginx/ssl/server.ext
```

```
sudo systemctl reload nginx
```




