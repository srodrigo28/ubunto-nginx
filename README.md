## Projeto Nginx

* > editar o arquivo na pasta
```
root@srv1313919:~/infra-proxy# nano conf/default.conf
```

* > Versão 1
```
  # ==========================================
              # SITE 1 - PORTAL (AGORA COM SSL/CADEADO) 🔒
              # ==========================================
              server {
                  listen 443 ssl;
                  server_name 99dev.pro www.99dev.pro;

                  # Certificado do SITE PRINCIPAL (que acabamos de gerar)
                  ssl_certificate /etc/letsencrypt/live/99dev.pro/fullchain.pem;
                  ssl_certificate_key /etc/letsencrypt/live/99dev.pro/privkey.pem;

                  location / {
                      proxy_pass http://site1:80;
                      proxy_set_header Host $host;
                      proxy_set_header X-Real-IP $remote_addr;
                      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                      proxy_set_header X-Forwarded-Proto $scheme;
                  }
              }

              # ==========================================
              # JAVA APP (COM SSL/CADEADO) 🔒
              # ==========================================
              server {
                  listen 443 ssl;
                  server_name java.99dev.pro;

                  # Certificado do JAVA
                  ssl_certificate /etc/letsencrypt/live/java.99dev.pro/fullchain.pem;
                  ssl_certificate_key /etc/letsencrypt/live/java.99dev.pro/privkey.pem;

                  location / {
                      proxy_pass http://app-java:8080;
                      proxy_set_header Host $host;
                      proxy_set_header X-Real-IP $remote_addr;
                      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                      proxy_set_header X-Forwarded-Proto $scheme;
                  }
              }

              # ==========================================
              # SITE 2 (API - Mantido na porta 80 por enquanto)
              # ==========================================
              server {
                  listen 80;
                  server_name api.99dev.pro;
                  location /.well-known/acme-challenge/ { root /var/www/certbot; }
                  location / {
                      proxy_pass http://site2:80;
                      proxy_set_header Host $host;
                      proxy_set_header X-Real-IP $remote_addr;
                  }
              }

              # ==========================================
              # REDIRECIONAMENTO GERAL (Força HTTPS) 🔄
              # Joga todo mundo da porta 80 para a 443 (exceto API)
              # ==========================================
              server {
                  listen 80;
                  server_name 99dev.pro www.99dev.pro java.99dev.pro;

                  location /.well-known/acme-challenge/ {
                      root /var/www/certbot;
                  }

                  location / {
                      return 301 https://$host$request_uri;
                  }
              }

```

* > Versão 2
```
# ==========================================
# SITE 1 - PORTAL (COM SSL/CADEADO) 🔒
# ==========================================
server {
    listen 443 ssl;
    server_name 99dev.pro www.99dev.pro;

    # Certificado do SITE PRINCIPAL
    ssl_certificate /etc/letsencrypt/live/99dev.pro/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/99dev.pro/privkey.pem;

    location / {
        proxy_pass http://site1:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# ==========================================
# JAVA APP (COM SSL/CADEADO) 🔒
# ==========================================
server {
    listen 443 ssl;
    server_name java.99dev.pro;

    # Certificado do JAVA
    ssl_certificate /etc/letsencrypt/live/java.99dev.pro/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/java.99dev.pro/privkey.pem;

    location / {
        proxy_pass http://app-java:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# ==========================================
# API PRECIFEX / ATAS (Nova Configuração) 🚀
# ==========================================
server {
    listen 80;
    server_name api.99dev.pro;

    # Pasta para validação do Certbot (Não remova até gerar o SSL)
    location /.well-known/acme-challenge/ { 
        root /var/www/certbot; 
    }

    location / {
        # AQUI ESTÁ A MUDANÇA: Aponta para o Gunicorn no Host (Porta 5000)
        # O IP 172.17.0.1 é o Gateway do Docker para falar com o Host
        proxy_pass http://172.17.0.1:5000;
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# ==========================================
# REDIRECIONAMENTO GERAL (Força HTTPS) 🔄
# Joga sites principais para HTTPS. 
# (A API ainda não está aqui para não travar antes de gerar o certificado)
# ==========================================
server {
    listen 80;
    server_name 99dev.pro www.99dev.pro java.99dev.pro;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}
```

> * depois de editado
```
docker exec porteiro-nginx nginx -t
docker exec porteiro-nginx nginx -s reload
```