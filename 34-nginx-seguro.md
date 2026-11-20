---
id: ART-34
titulo: Nginx seguro
version: 1.0.0
estado: vigente
aplica_a: [infraestructura]
depende_de: [ART-02, ART-09, ART-33]
---

# ART-34 · Nginx seguro

> **Instrucción para agentes IA**
> Usa esta configuración como base y ajusta dominios, CSP y límites según el proyecto. Cualquier relajación de cabeceras o límites requiere ADR. Principios: P-SN-01..04, P-VA-05, P-SB-02.
> **Trampa conocida:** en Nginx, `add_header` dentro de un `location` **anula** todos los `add_header` heredados del `server`. Por eso las cabeceras de seguridad viven en un snippet que se incluye en cada `location` que define sus propias cabeceras.

---

## 1. Estructura de archivos

```
infra/nginx/
├── nginx.conf
├── conf.d/
│   └── app.conf
└── snippets/
    ├── ssl.conf
    ├── security-headers.conf
    └── proxy-common.conf
```

## 2. `nginx.conf`

```nginx
worker_processes auto;
pid /tmp/nginx.pid;
events { worker_connections 4096; }

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    server_tokens off;
    charset utf-8;

    # Rutas temporales escribibles (imagen sin privilegios, read_only)
    client_body_temp_path /tmp/client_temp;
    proxy_temp_path       /tmp/proxy_temp;
    fastcgi_temp_path     /tmp/fastcgi_temp;
    uwsgi_temp_path       /tmp/uwsgi_temp;
    scgi_temp_path        /tmp/scgi_temp;

    sendfile on;
    tcp_nopush on;
    keepalive_timeout 30s;

    # Límites anti abuso (P-VA-05, API4)
    client_max_body_size      1m;      # Se amplía por location para cargas de archivos
    client_body_timeout       15s;
    client_header_timeout     15s;
    send_timeout              30s;
    large_client_header_buffers 4 8k;

    # Rate limiting (P-SB-02)
    limit_req_zone  $binary_remote_addr zone=api_ip:10m   rate=20r/s;
    limit_req_zone  $binary_remote_addr zone=auth_ip:10m  rate=5r/m;
    limit_conn_zone $binary_remote_addr zone=conn_ip:10m;
    limit_req_status  429;
    limit_conn_status 429;

    # WebSocket
    map $http_upgrade $connection_upgrade { default upgrade; '' close; }

    # Logs JSON sin query string (puede contener tickets) ni cabeceras sensibles
    log_format json escape=json '{"time":"$time_iso8601","remote_addr":"$remote_addr",'
        '"request_id":"$request_id","method":"$request_method","uri":"$uri",'
        '"status":$status,"bytes":$body_bytes_sent,"rt":$request_time,'
        '"upstream_rt":"$upstream_response_time","ua":"$http_user_agent"}';
    access_log /dev/stdout json;
    error_log  /dev/stderr warn;

    gzip on;
    gzip_types text/css application/javascript application/json image/svg+xml;
    gzip_min_length 1024;

    upstream api_upstream {
        server api:8000;
        keepalive 32;
    }

    include /etc/nginx/conf.d/*.conf;
}
```

> Se registra `$uri` (sin query string) en lugar de `$request_uri`, para no guardar tickets WS ni parámetros con datos personales.

## 3. `snippets/ssl.conf` (perfil Mozilla "intermediate")

```nginx
ssl_certificate     /etc/nginx/certs/fullchain.pem;
ssl_certificate_key /etc/nginx/certs/privkey.pem;
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
ssl_prefer_server_ciphers off;
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:10m;
ssl_session_tickets off;
```

## 4. `snippets/security-headers.conf`

```nginx
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=(), usb=()" always;
add_header Cross-Origin-Opener-Policy "same-origin" always;
add_header Cross-Origin-Resource-Policy "same-origin" always;
add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data: blob:; font-src 'self'; connect-src 'self' wss://$host; frame-ancestors 'none'; base-uri 'self'; form-action 'self'; object-src 'none'; upgrade-insecure-requests" always;
```

- Agrega `preload` a HSTS solo cuando el dominio y todos sus subdominios estén listos para HTTPS permanente.
- Si el design system necesita estilos inline (algunas librerías), preferir hashes/nonce; `'unsafe-inline'` en `style-src` requiere ADR. Nunca `'unsafe-inline'` ni `'unsafe-eval'` en `script-src`.
- Para empezar con una CSP nueva en un sistema existente: `Content-Security-Policy-Report-Only` durante un sprint.

## 5. `snippets/proxy-common.conf`

```nginx
proxy_http_version 1.1;
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Request-ID      $request_id;
proxy_hide_header X-Powered-By;
proxy_hide_header Server;
proxy_connect_timeout 5s;
proxy_read_timeout    30s;
proxy_send_timeout    30s;
```

`X-Forwarded-For` se **reemplaza** con `$proxy_add_x_forwarded_for` desde el borde; si Nginx está detrás de otro balanceador, configurar `set_real_ip_from` solo con sus IPs.

## 6. `conf.d/app.conf`

```nginx
# HTTP → HTTPS
server {
    listen 8080 default_server;
    server_name _;
    location /.well-known/acme-challenge/ { root /var/www/certbot; }
    location / { return 301 https://$host$request_uri; }
}

# Rechaza hosts desconocidos en HTTPS
server {
    listen 8443 ssl default_server;
    http2 on;
    include /etc/nginx/snippets/ssl.conf;
    return 444;
}

server {
    listen 8443 ssl;
    http2 on;
    server_name app.ejemplo.com;          # ← dominio real
    include /etc/nginx/snippets/ssl.conf;
    include /etc/nginx/snippets/security-headers.conf;

    limit_conn conn_ip 50;

    # Solo métodos esperados
    if ($request_method !~ ^(GET|POST|PUT|PATCH|DELETE|OPTIONS|HEAD)$) { return 405; }

    # Bloquear archivos ocultos y de respaldo
    location ~ /\.(?!well-known) { deny all; return 404; }
    location ~* \.(bak|old|orig|sql|env|log|ini|sh|swp)$ { deny all; return 404; }

    # Autenticación: límites más estrictos
    location ~ ^/api/v1/auth/(login|refresh|mobile/refresh|password) {
        limit_req zone=auth_ip burst=5 nodelay;
        include /etc/nginx/snippets/proxy-common.conf;
        proxy_pass http://api_upstream;
    }

    # Cargas de archivos (ajustar por proyecto)
    location ^~ /api/v1/archivos {
        client_max_body_size 10m;
        limit_req zone=api_ip burst=10 nodelay;
        include /etc/nginx/snippets/proxy-common.conf;
        proxy_request_buffering on;
        proxy_pass http://api_upstream;
    }

    # API
    location /api/ {
        limit_req zone=api_ip burst=40 nodelay;
        include /etc/nginx/snippets/proxy-common.conf;
        proxy_pass http://api_upstream;
    }

    # Documentación del API: deshabilitada en producción (P-SB-08)
    location ~ ^/(docs|redoc|openapi\.json) { return 404; }

    # WebSocket
    location /ws/ {
        limit_req zone=api_ip burst=10 nodelay;
        include /etc/nginx/snippets/proxy-common.conf;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_read_timeout 75s;              # > intervalo de heartbeat (ART-10 §6.9)
        proxy_send_timeout 75s;
        proxy_pass http://api_upstream;
    }

    # Health solo interno
    location /health/ { deny all; return 404; }

    # Estáticos con hash: caché larga
    location /assets/ {
        root /usr/share/nginx/html;
        include /etc/nginx/snippets/security-headers.conf;
        add_header Cache-Control "public, max-age=31536000, immutable" always;
        try_files $uri =404;
    }

    # SPA
    location / {
        root /usr/share/nginx/html;
        include /etc/nginx/snippets/security-headers.conf;
        add_header Cache-Control "no-store" always;     # index.html nunca se cachea
        try_files $uri /index.html;
    }

    # Mapas de fuentes nunca públicos (P-SF-06)
    location ~* \.map$ { deny all; return 404; }
}
```

## 7. Certificados
- Let's Encrypt con `certbot` (contenedor aparte o en el host) y renovación automática con recarga de Nginx.
- Llave privada con permisos `0600`, montada en solo lectura.
- Verificar el resultado con SSL Labs (objetivo A o A+) y `testssl.sh` en el pipeline de staging.

## 8. Pruebas de la configuración (ART-42)

| Prueba | Esperado |
|---|---|
| `nginx -t` en CI | Sin errores |
| `curl -I http://` | 301 a HTTPS |
| Host desconocido | Conexión cerrada (444) |
| Cabeceras en `/`, `/assets/x.js`, `/api/...` | Todas las de §4 presentes |
| `Server` en respuestas | Sin versión |
| `/.env`, `/.git/config`, `/backup.sql`, `/index.js.map` | 404 |
| `/docs`, `/openapi.json`, `/health/ready` desde fuera | 404 |
| Ráfaga de 20 intentos de login en 1 min | 429 |
| Body de 2 MB a `/api/` | 413 |
| Método `TRACE` | 405 |
| TLS 1.0 / 1.1 | Rechazado |
| WebSocket sin `Upgrade` | No conecta |
