# Test_AgileOps

For ssh jump, first cat ~/.ssh/id_rsa.pub on local machine. copy it and paste to db,app instance ~/.ssh/authorized_keys

Testing the ssh to app and db using
```
ssh db
ssh app
```

Make db, app instance can access internet using ssh tunnel:
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
