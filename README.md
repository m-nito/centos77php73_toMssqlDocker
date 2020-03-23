# Centos7にphpを導入し、docker経由でmssqlを呼び出す
```
cat /etc/centos-release
>> CentOS Linux release 7.7.1908 (Core)
```

おおむね以下のような構造になります。

- Centos 7.7
  - PHP 7.3
  - Docker
    - MsSql

ここで`PHP7.3`が`MsSql`にアクセス可能な状態にします。

# php / mssql basic installations
出典：https://ptsv.jp/repository/rhel7-php71-sqlserver/
```
sudo yum -y update
sudo yum -y install epel-release
sudo rpm -Uvh http://rpms.remirepo.net/enterprise/remi-release-7.rpm
sudo su
curl https://packages.microsoft.com/config/rhel/7/prod.repo > /etc/yum.repos.d/mssql-release.repo
exit
sudo yum remove unixODBC-utf16 unitODBC-utf16-devel
sudo ACCEPT_EULA=Y yum -y install msodbcsql17
sudo ACCEPT_EULA=Y yum -y install mssql-tools
echo 'export PATH="$PATH:/opt/mssql-tools/bin"' >> ~/.bash_profile
echo 'export PATH="$PATH:/opt/mssql-tools/bin"' >> ~/.bashrc
source ~/.bashrc
sudo yum remove php-*
sudo yum -y --enablerepo=remi,remi-php73 install php php-devel php-pdo php-sqlsrv unixODBC-devel php-mbstring php-gd php-cli
```

# docker
```
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum -y install docker-ce
sudo su
systemctl start docker && systemctl enable $_
curl -o /usr/local/bin/docker-compose -L https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname -s`-`uname -m`
chmod +x /usr/local/bin/docker-compose
exit

sudo usermod -aG docker $USER
sudo reboot
```

# docker-compose
出典：https://qiita.com/bezeklik/items/972c6d1c331ec891d1e3
```
mkdir docker_mssql && cd docker_mssql

cat << "_EOF_" > docker-compose.yml && docker-compose up -d
version: '3'

services:
  db_mssql:
    image: microsoft/mssql-server-linux:2017-latest
    container_name: mssql
    hostname: mssql
    volumes:
      - ./.db:/var/opt/mssql/
      - /var/opt/mssql/data
    environment:
      - ACCEPT_EULA=Y
      - MSSQL_SA_PASSWORD=YourP@ssw0rd
    # OPTIONAL (this allows you to connect as 'localhost'.)
    # (Otherwise, you may simpy get network address by doing 'docker network ls' then 'docker network inspect mssql_default' stuff.)
    network_mode: "host"
    ports:
        - 1433:1433
_EOF_

docker-compose up -d
```

→`docker ps -a`などで起動できていることが確認できるなら次へ

# 確認を行う
## sqlcmdでmssqlへの疎通を確認
```
sqlcmd -S localhost,1433 -U sa -P YourP@ssw0rd -Q "SELECT @@version;"

>> Microsoft SQL Server 2017 (RTM-CU13) (KB4466404) - 14.0.3048.4 (X64)
>>        Nov 30 2018 12:57:58
>>        Copyright (C) 2017 Microsoft Corporation
>>        Developer Edition (64-bit) on Linux (Ubuntu 16.04.5 LTS)
```

## php経由で確認
```
mkdir php_test && cd php_test
cat << "_EOF_" > test.php
<?php
$pdo = new PDO('sqlsrv:Server=localhost,1433','sa','YourP@ssw0rd');
$res = $pdo->query('SELECT @@version;')->fetchColumn();
echo($res);
_EOF_

php test.php

>> Microsoft SQL Server 2017 (RTM-CU13) (KB4466404) - 14.0.3048.4 (X64)
>>        Nov 30 2018 12:57:58
>>        Copyright (C) 2017 Microsoft Corporation
>>        Developer Edition (64-bit) on Linux (Ubuntu 16.04.5 LTS)
```

# 備考
- MsSqlイメージについて
  - うまく起動しない場合は`docker-compose up --verbose`で詳細を確認する
  - `AWS`で構築を行う場合、`t2.micro`だとメモリが足りず`mssql`が起動しないため注意
  - パスワードが複雑性の要件を満たさない場合、`mssql`イメージが起動しないため注意
