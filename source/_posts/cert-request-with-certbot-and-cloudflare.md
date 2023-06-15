# 前提
1. 域名被cloudflare管理
2. docker环境

# 使用docker申请 
1. 在[Cloud Flare](https://dash.cloudflare.com/profile/api-tokens)上去获取Global API Key;并写入到cloudflare.ini配置文件中去
```
mkdir certbot

echo "dns_cloudflare_email = your.email@xxx.com
dns_cloudflare_api_key = cf-global-token" > certbot/cloudflare.ini
```

2. 申请证书
```
docker run -it --rm --name certbot \
            -v ./certbot/etc:/etc/letsencrypt \
            -v ./certbot/lib:/var/lib/letsencrypt \
            -v ./certbot:/.secrets \
            certbot/dns-cloudflare certonly \
            --non-interactive \
            --dns-cloudflare \
            --dns-cloudflare-credentials /.secrets/cloudflare.ini \
            --dns-cloudflare-propagation-seconds 60 \
            -m your.email@xxx.com \
            --agree-tos \
            --no-eff-email \
            -d '*.your.domain'
```

3. renew证书
```
docker run -it --rm --name certbot \
    -v "./certbot/etc:/etc/letsencrypt" \
    -v "./certbot/cloudflare.ini:/cloudflare.ini" \
    certbot/dns-cloudflare renew \
    --dns-cloudflare --dns-cloudflare-credentials /cloudflare.ini
```



