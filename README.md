# Test_AgileOps

For ssh jump, first cat ~/.ssh/id_rsa.pub on local machine. copy it and paste to db,app instance ~/.ssh/authorized_keys

Testing the ssh to app and db using
```
ssh db
ssh app
```
## 1. MySQL Setup on DB Instance:
Install and configure MySQL on the DB instance ( 10.100.0.101 ).

Because i can't edit the ip tables or add a NAT Gateway so i think about making db instance can access apt using ssh tunnel:
```
ssh -i ~/.ssh/agileops.pem -D 8080 -f -C -q -N ubuntu@10.100.4.50
```

Then add those line to /etc/apt/apt.conf.d/95proxies
```
Acquire::http::Proxy "socks5h://127.0.0.1:8080";
Acquire::https::Proxy "socks5h://127.0.0.1:8080";
```

```
ansible-playbook -i inventory.ini db.yml -v
```
![image](https://github.com/user-attachments/assets/67a8a801-0d60-4adb-bf60-f4aca5dfab0e)

![image](https://github.com/user-attachments/assets/77de97d5-2206-4848-9e70-8abcc6a8cb9f)

## 2. FastAPI Setup on App Instance:

Install and configure FastAPI on the App instance ( 10.100.0.100 ).

First make the app instance can access apt, then pip3 install through proxy
```
sudo apt install python3 python3-pip uvicorn python3-socks 
export ALL_PROXY=socks5://127.0.0.1:8080
pip3 install fastapi uvicorn
```

Prepare a basic API endpoint for verification and expose it securely.

Create a service for app

```
[Unit]
Description=Uvicorn instance for FastAPI app
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/app
ExecStart=uvicorn main:app --host 0.0.0.0 --port 8000 --reload
Restart=always

[Install]
WantedBy=multi-user.target
```
Run the service
```
sudo systemctl enable uvicorn
sudo systemctl start uvicorn
```
![image](https://github.com/user-attachments/assets/deec7158-e0e0-43ca-92a0-a49856231757)


## 3. Private DNS Server on Bastion:
Set up a private DNS server on the Bastion host to resolve:
- app.instance.local to the app instance’s private IP ( 10.100.0.100 ).
- db.instance.local to the db instance’s private IP ( 10.100.0.101 ).
```
ansible-playbook -i inventory.ini privatedns.yml -v
```

Ensure the private DNS server is accessible and functional across the bastion, app, and db instances (but not the external instance).

## 4. NGINX Reverse Proxy with SSL:

Configure NGINX on the Bastion host to act as a reverse proxy for FastAPI, using the domain https://api.lab.aandd.io .

```
ansible-playbook -i inventory.ini reverseproxy.yml -v
sudo certbot --nginx -d api.lab.aandd.io --non-interactive --agree-tos --email kietngo255@gmail.com
sudo crontab -e
0 0 1 * * certbot renew --quiet && systemctl reload nginx
```

![image](https://github.com/user-attachments/assets/072b994e-c3d7-4d63-9c4a-f7c97d8a0852)

## 5. Database Access for Analytics:
- Configure the External instance to connect securely to the MySQL database on the db instance.
- Verify database connectivity using phpMyAdmin or a MySQL client from the external instance.

On db instance, Edit file /etc/mysql/mysql.conf.d/mysqld.cnf

```
bind-address = 10.100.0.101
```

```
mysql -u root -p
CREATE USER 'analytics_user'@'10.100.4.50' IDENTIFIED BY '';
GRANT ALL PRIVILEGES ON *.* TO 'analytics_user'@'10.100.4.50';
FLUSH PRIVILEGES;
```

On external instance
```
ssh -f -i ~/.ssh/agileops.pem -N -L 3306:10.100.0.101:3306 ubuntu@52.76.217.36
```
Testing the connection
![image](https://github.com/user-attachments/assets/ae730aa1-bb71-4c7b-8c6b-1c0e84e81afa)

```
ansible-playbook -i inventory.ini external.yml -v
```
Access through http://18.142.15.207/phpmyadmin

![image](https://github.com/user-attachments/assets/243b4b8c-b2fb-41cc-80a1-5b6a8a6820ca)


