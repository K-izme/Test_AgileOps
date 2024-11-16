# Test_AgileOps

![agileops drawio](https://github.com/user-attachments/assets/f452f3cd-8f0a-44e9-a1d7-a160c29bfa3c)

For testing ssh jump, first cat ~/.ssh/id_rsa.pub of personal machine. copy it and paste to db,app instance ~/.ssh/authorized_keys

Configure ~/.ssh/config like in the ssh/config file

Testing the ssh to app and db using
```
ssh db
ssh app
```
If it work, now ansible on our local machine can access to db, app instance through bastion instance

## 1. MySQL Setup on DB Instance:
Install and configure MySQL on the DB instance ( 10.100.0.101 ).

Because i can't edit the iptables, make it a NAT gateway or add an AWS NAT Gateway so i think about making db instance can access apt using ssh tunnel:
```
ssh -i ~/.ssh/agileops.pem -D 8080 -f -C -q -N ubuntu@10.100.4.50
```

Then add those line to /etc/apt/apt.conf.d/95proxies
```
Acquire::http::Proxy "socks5h://127.0.0.1:8080";
Acquire::https::Proxy "socks5h://127.0.0.1:8080";
```

Now can execute ansible playbook and set up mysql on db instance

```
ansible-playbook -i inventory.ini db.yml -v
```
![image](https://github.com/user-attachments/assets/67a8a801-0d60-4adb-bf60-f4aca5dfab0e)

![image](https://github.com/user-attachments/assets/77de97d5-2206-4848-9e70-8abcc6a8cb9f)

## 2. FastAPI Setup on App Instance:

Install and configure FastAPI on the App instance ( 10.100.0.100 ).

First make the app instance can access apt by doing the same thing with we have done with db instance, then pip3 install through proxy

```
sudo apt install python3 python3-pip uvicorn python3-socks 
export ALL_PROXY=socks5://127.0.0.1:8080
pip3 install fastapi uvicorn
```

Prepare a basic API endpoint (in app folder) for verification and expose it securely.

Then ssh into db instance and prepare database for app
```
CREATE DATABASE app_db;
CREATE USER 'app_user'@'10.100.0.100' IDENTIFIED BY '...';
GRANT ALL PRIVILEGES ON app_db.* TO 'app_user'@'10.100.0.100';
FLUSH PRIVILEGES;
```

Create a service for app on app instance

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
pip install -r requirements.txt
sudo systemctl enable uvicorn
sudo systemctl start uvicorn
```

![image](https://github.com/user-attachments/assets/deec7158-e0e0-43ca-92a0-a49856231757)


## 3. Private DNS Server on Bastion:
Set up a private DNS server on the Bastion host to resolve:
- app.instance.local to the app instance’s private IP ( 10.100.0.100 ).
- db.instance.local to the db instance’s private IP ( 10.100.0.101 ).
- 
```
ansible-playbook -i inventory.ini privatedns.yml -v
```

Ensure the private DNS server is accessible and functional across the bastion, app, and db instances (but not the external instance).

From Bastion:

![image](https://github.com/user-attachments/assets/7236a367-2da0-4cb9-8408-08843a116099)

From App:

![image](https://github.com/user-attachments/assets/2b5b281a-7b40-48e2-87d3-6bef6df63ada)

From DB:

![image](https://github.com/user-attachments/assets/40bb0b19-5d1b-4fe7-8b7a-da0fcdcd81b4)

From external:

![image](https://github.com/user-attachments/assets/1a9d3c19-8845-445e-b75e-bee79f0130b8)

## 4. NGINX Reverse Proxy with SSL:

Configure NGINX on the Bastion host to act as a reverse proxy for FastAPI, using the domain https://api.lab.aandd.io .

```
ansible-playbook -i inventory.ini reverseproxy.yml -v
sudo certbot --nginx -d api.lab.aandd.io --non-interactive --agree-tos --email kietngo255@gmail.com
sudo crontab -e
0 0 1 * * certbot renew --quiet && systemctl reload nginx
```
This also set up ssl using certbot and a cron job to renew cert every month (because it expires every 3 months)

![image](https://github.com/user-attachments/assets/072b994e-c3d7-4d63-9c4a-f7c97d8a0852)

https://api.lab.aandd.io/health

https://api.lab.aandd.io/signup

https://api.lab.aandd.io/login

https://api.lab.aandd.io/logout

## 5. Database Access for Analytics:
- Configure the External instance to connect securely to the MySQL database on the db instance.
- Verify database connectivity using phpMyAdmin or a MySQL client from the external instance.

On db instance, Edit file /etc/mysql/mysql.conf.d/mysqld.cnf

```
bind-address = 10.100.0.101
```

Create user for analytics and grant privilege

```
mysql -u root -p
CREATE USER 'analytics_user'@'10.100.4.50' IDENTIFIED BY '';
GRANT ALL PRIVILEGES ON *.* TO 'analytics_user'@'10.100.4.50';
FLUSH PRIVILEGES;
```

On external instance, set up an ssh tunnel to secure traffic between the External Instance and the Bastion Host. Uses the Bastion Host to forward requests to the DB Instance.

```
ssh -f -i ~/.ssh/agileops.pem -N -L 3306:10.100.0.101:3306 ubuntu@52.76.217.36
```

Test the connection 

![image](https://github.com/user-attachments/assets/ae730aa1-bb71-4c7b-8c6b-1c0e84e81afa)

```
ansible-playbook -i inventory.ini external.yml -v
```
Access through http://18.142.15.207/phpmyadmin

![image](https://github.com/user-attachments/assets/243b4b8c-b2fb-41cc-80a1-5b6a8a6820ca)

![image](https://github.com/user-attachments/assets/4c003784-c7cd-4fcf-8344-8f161f7b42c7)

