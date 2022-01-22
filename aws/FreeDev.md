## AWS Ec2 Amazon linux 2

## Python3.8 

### install python
```
sudo amazon-linux-extras install -y python3.8
```

### set alias
```
```

## Postgresql 13

### 1. Add PostgreSQL Yum Repository
```
sudo tee /etc/yum.repos.d/pgdg.repo<<EOF
[pgdg13]
name=PostgreSQL 13 for RHEL/CentOS 7 - x86_64
baseurl=https://download.postgresql.org/pub/repos/yum/13/redhat/rhel-7-x86_64
enabled=1
gpgcheck=0
EOF
```
### 2. Command to install PostgreSQL
```
sudo yum install postgresql13 postgresql13-server
```

### 3. Initial database configurations

```
sudo /usr/pgsql-13/bin/postgresql-13-setup initdb
```

### 4. Enable and Start PostgreSQL Service

```
sudo systemctl start postgresql-13
sudo systemctl enable postgresql-13
sudo systemctl status postgresql-13
```

### 5. Secure PostgreSQL default Database
```
sudo passwd postgres
```  

```
su - postgres
```

```
psql -c "ALTER USER postgres WITH PASSWORD 'your-password';"
```


## Remote - SSH

### Setting file

```
Host remotessh
  Hostname <サーバーのIP>
  User <ログインユーザー名>
  Port 22
  IdentityFile <秘密鍵のパス>
```