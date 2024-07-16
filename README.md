# deploy_ctfd


## 1. Install Dependencies
```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip nginx git
```

## 2. Clone CTFd Repository
Clone the CTFd repository from GitHub.
```bash
cd /opt
sudo git clone https://github.com/CTFd/CTFd.git
cd CTFd
```

## 3. Set Up Python Virtual Environment
Create and activate a Python virtual environment.

```bash
sudo python3 -m venv env
source env/bin/activate
```

## 4. Install CTFd Requirements
Install the required Python packages.

```bash
sudo pip install -r requirements.txt
```

## 5. Configure Gunicorn
Create a Gunicorn systemd service file.

```bash
sudo nano /etc/systemd/system/ctfd.service
```
```ini
[Unit]
Description=CTFd
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/opt/CTFd
Environment="PATH=/opt/CTFd/env/bin"
ExecStart=/opt/CTFd/env/bin/gunicorn --workers 3 --bind unix:/opt/CTFd/ctfd.sock "CTFd:create_app()"

[Install]
WantedBy=multi-user.target

```


## 6. Set Permissions for the CTFd Directory
Ensure the permissions for the CTFd directory and the Gunicorn socket file are correct.
```bash
sudo chown -R www-data:www-data /opt/CTFd
sudo chmod -R 755 /opt/CTFd
```

## 7. Set Up Python Virtual Environment
Create and activate a Python virtual environment.

```bash
python3 -m venv env
source env/bin/activate
```
Reload systemd and start the Gunicorn service.


```bash
sudo systemctl daemon-reload
sudo systemctl start ctfd
sudo systemctl enable ctfd
```

## 8. Create a Self-Signed SSL Certificate

Generate a self-signed SSL certificate inside the CTFd directory.
```bash
cd /opt/CTFd
sudo mkdir ssl
cd ssl

sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ctfd-selfsigned.key -out ctfd-selfsigned.crt
sudo openssl dhparam -out dhparam.pem 2048
```

## 9. Configure Nginx
Create an Nginx configuration file for CTFd.

```sh
sudo nano /etc/nginx/sites-available/ctfd
```

Add the following content to the file:
```ini
server {
    listen 80;
    server_name {ip address};
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name {ip address};

    ssl_certificate /opt/CTFd/ssl/ctfd-selfsigned.crt;
    ssl_certificate_key /opt/CTFd/ssl/ctfd-selfsigned.key;
    ssl_dhparam /opt/CTFd/ssl/dhparam.pem;

    ssl_protocols TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers EECDH+AESGCM:EDH+AESGCM;
    ssl_ecdh_curve secp384r1;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;

    location / {
        proxy_pass http://unix:/opt/CTFd/ctfd.sock;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```

Enable the new configuration
```bash 
sudo ln -s /etc/nginx/sites-available/ctfd /etc/nginx/sites-enabled
```

## 10. Verify Nginx Configuration
Test the Nginx configuration to ensure there are no syntax errors:

```bash 
sudo nginx -t
```

If the test is successful, reload Nginx:
```bash 
sudo systemctl reload nginx
```

## 11. Access CTFd
Now you can access CTFd via https://{ip address} in your web browser. You should be prompted with a warning about the self-signed certificate. Accept the warning to proceed to the site.

## License
[MIT](https://choosealicense.com/licenses/mit/)
