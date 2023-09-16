# Nginx-notes
This readme file will be updated regularly.

## 1. Installing Nginx in Rocky Linux OS Environment

```
sudo dnf install nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```
If you get an error when starting, try following the step below.

```
sudo apachectl stop
sudo systemctl restart nginx
```

now nginx is enabled and started.

## 2. Configure Nginx

```
vim /etc/ngingx/nginx.conf
```
If you want to customize your log directory, you can create new one with selected permissions for nginx but I'm gonna use /var/log directory for this time.
```
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    access_log /var/log/nginx/access.log main;
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    server_tokens off;

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}

```

Restart nginx service again

```
sudo systemctl restart nginx.service
```
# 3. Create a basic UI to access with nginx service
```
sudo mkdir -p /var/www/html/tarikspageburada.com
sudo vim /var/www/html/tarikspageburada.com/index.html
```

```
<!DOCTYPE html>
<html>
<head>
    <title>Tarik's Page Burada</title>
</head>
<body>
    <h1>Hello, World!</h1>
</body>
</html>

```
```
sudo nano /etc/nginx/sites-available/tarikspageburada.com
```

```
server {
    listen 80;
    server_name tarikspageburada.com;

    root /var/www/html/tarikspageburada.com;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

```
sudo ln -s /etc/nginx/sites-available/tarikspageburada.com /etc/nginx/sites-enabled/
```

```
sudo nginx -t
```
If the process is successfull, then

```
systemctl restart nginx.service
```

```
curl http://tarikburada.com
```
# 4. Create a UI with HTTPS (Certificated Connection)
First of all we need .pem and .key files to read the certificates by nginx, but in my case, only a password and pfx file exist. I'll start by converting this file to .pem as requested by nginx. Openssl has very efficient commands for this. Then, I'm gonna move these new files to /etc/ssl/certs  to prevent permission problems.

```
openssl pkcs12 -nodes -in vertificate.pfx -clcerts -nokeys -out certificate.pem
openssl rsa -in certificate.pem -out certificate.key
```

```
vim /etc/nginx/sites-available/tarikworld.lockumtech.com
```

```
server {
    listen 443 ssl;
    server_name tarikworld.lockumtech.com;
    root /var/www/html/tarikworld.lockumtech.com;
    ssl_certificate /etc/ssl/certs/certificate.pem;
    ssl_certificate_key /etc/ssl/certs/certificate.key;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        
	try_files $uri $uri/ =404;
    }
}

```

```
sudo ln -s /etc/nginx/sites-available/tarikworld.lockumtech.com /etc/nginx/sites-enabled/
sudo nginx -t  
sudo systemctl restart nginx
```

It's time to test the secured site we created 
```
curl https://tarikworld.lockumtech.com
```

# 5. Run a Python Flask Application with Nginx

Here, we will make a request with the domain we created in Nginx, using the Flask API, which is a Python library.
```
dnf -y update
sudo dnf install vim wget openssl-devel bzip2-devel libffi-devel -y
sudo dnf -y groupinstall "Development Tools"
VERSION=3.11.4
wget https://www.python.org/ftp/python/$VERSION/Python-$VERSION.tgz
tar xvf Python-$VERSION.tgz
cd Python-$VERSION
./configure --enable-optimizations
sudo make altinstall
python3.11 --version
```


```
mkdir flask_app
cd flask_app
python3 -m venv venv
source venv/bin/activate
pip install flask
```

```
vim app.py
```

```
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello World'

@app.route('/space')
def hello_uzay():
    return 'Hello alliens'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)

```

To keep our application always up, we install gunicorn.

```
pip install gunicorn
gunicorn -w 4 -b 0.0.0.0:5000 app:app --daemon --pid /var/run/myapp.pid

```

you can test the api by sending request to localhost:5000 and its extensions.

Now we must configure our nginx service for this flask api.


```
vim /etc/nginx/sites-available/flask_app
```

```
server {
    listen 80;
    server_name yuzucukursuapi.com www.yuzucukursuapi.com;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /uzay {
        proxy_pass http://127.0.0.1:5000/space;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

```

```
nginx -t
systemctl restart nginx.service
```
connect() to 127.0.0.1:5000 failed (13: Permission denied) while connecting to upstream, client: 127.0.0.1, server: yuzucukursuapi.com,
In my case, Although the nginx -t command was successful, the system could not start.

After a little research, I realized that the problem was caused by internal port forwarding.




But for HA projects we need to reach all the extensions that we created in code as functions. That's why we need to reconfigure our nginx conf as below;

```
server {
    listen 80;
    server_name yuzucukursuapi.com www.yuzucukursuapi.com;

    location / {
        rewrite ^/(.*)$ /$1 break;
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

```
```
sudo setsebool -P httpd_can_network_connect on
systemctl restart nginx.service
```

I will detail these topics with more functional examples. see you


