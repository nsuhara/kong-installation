# Kong API Gatewayを構築する

## はじめに

`Mac環境の記事ですが、Windows環境も同じ手順になります。環境依存の部分は読み替えてお試しください。`

### 目的

この記事を最後まで読むと、次のことができるようになります。

| No.  | 概要         | キーワード          |
| :--- | :----------- | :------------------ |
| 1    | 仮装環境構築 | VirtualBox, Vagrant |
| 2    | Kong構築     | Kong, PostgreSQL    |

### 実行環境

| 環境           | Ver.    |
| :------------- | :------ |
| macOS Catalina | 10.15.3 |
| CentOS         | 7.0     |
| VirtualBox     | 6.0     |
| Vagrant        | 2.2.8   |
| Kong           | 2.0.4   |
| PostgreSQL     | 11.7    |

### 関連する記事

- [Kong CentOS Installation](https://docs.konghq.com/install/centos/?_ga=2.204812184.982123663.1559529933-582008484.1559346208)
- [PostgreSQL Yum Repository](https://www.postgresql.org/download/linux/redhat/)

## 全体の流れ

1. VirtualBoxをインストールする
2. Vagrantをインストールする
3. CentOSを立ち上げる
4. Kongをインストールする
5. PostgreSQLをインストールする
6. Kong DBを設定する
7. Kongサービスを起動する
8. ルーティングを登録する
9. ルーティングを確認する
10. まとめ

## 1. VirtualBoxをインストールする

### VirtualBoxインストール

```command.sh
~$ brew cask install virtualbox
```

## 2. Vagrantをインストールする

### Vagrantインストール

```command.sh
~$ brew cask install vagrant
```

## 3. CentOSを立ち上げる

### Vagrant初期化

```command.sh
~$ vagrant init centos/7
```

### Vagrant環境設定

```command.sh
~$ vi Vagrantfile

~$ - # config.vm.network "private_network", ip: "192.168.33.10"
~$ + config.vm.network "private_network", ip: "192.168.33.77"
```

### Vagrant起動

```command.sh
~$ vagrant up
```

### Vagrantログイン

```command.sh
~$ vagrant ssh
```

## 4. Kongをインストールする

### Kongリポジトリインストール

```command.sh
~$ sudo yum update -y
~$ sudo yum install -y wget
~$ wget https://bintray.com/kong/kong-rpm/rpm -O bintray-kong-kong-rpm.repo
~$ export major_version=`grep -oE '[0-9]+\.[0-9]+' /etc/redhat-release | cut -d "." -f1`
~$ sed -i -e 's/baseurl.*/&\/centos\/'$major_version''/ bintray-kong-kong-rpm.repo
~$ sudo mv bintray-kong-kong-rpm.repo /etc/yum.repos.d/
~$ sudo yum update -y
```

### Kongインストール

```command.sh
~$ sudo yum install -y kong
~$ kong version
2.0.4
```

## 5. PostgreSQLをインストールする

### PostgreSQLリポジトリ確認

- [PostgreSQL Yum Repository](https://www.postgresql.org/download/linux/redhat/)

### PostgreSQLリポジトリインストール

```command.sh
~$ sudo yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

### PostgreSQLインストール

```command.sh
~$ sudo yum install postgresql11
~$ sudo yum install postgresql11-server
```

### PostgreSQLサービス起動

```command.sh
~$ sudo /usr/pgsql-11/bin/postgresql-11-setup initdb
~$ sudo systemctl enable postgresql-11
~$ sudo systemctl start postgresql-11
```

### postgresパスワード設定

```command.sh
~$ sudo passwd postgres
新しいパスワード:
新しいパスワードを再入力してください:
```

## 6. Kong DBを設定する

### ユーザ/テーブル作成

```command.sh
~$ su - postgres
パスワード:

~$ psql
~# CREATE USER kong; CREATE DATABASE kong OWNER kong;
CREATE ROLE
CREATE DATABASE

~# \q
~$ exit
```

### クライアント認証設定

```command.sh
~$ sudo vi /var/lib/pgsql/11/data/pg_hba.conf

# IPv4 local connections:
- host    all             all             127.0.0.1/32            ident
+ host    all             all             127.0.0.1/32            trust

~$ sudo systemctl restart postgresql-11.service
```

### DBマイグレーション

```command.sh
~$ sudo chmod -R 777 /usr/local/kong
~$ sudo chmod -R 777 /usr/local/share/lua/5.1
~$ kong migrations bootstrap
```

### DB確認

```command.sh
~$ psql -h 127.0.0.1 -p 5432 -d kong -U kong
~> \dt
                   List of relations
 Schema |             Name              | Type  | Owner
--------+-------------------------------+-------+-------
 public | acls                          | table | kong
 public | acme_storage                  | table | kong
 public | basicauth_credentials         | table | kong
 public | ca_certificates               | table | kong
 public | certificates                  | table | kong
 public | cluster_events                | table | kong
 public | consumers                     | table | kong
 public | hmacauth_credentials          | table | kong
 public | jwt_secrets                   | table | kong
 public | keyauth_credentials           | table | kong
 public | locks                         | table | kong
 public | oauth2_authorization_codes    | table | kong
 public | oauth2_credentials            | table | kong
 public | oauth2_tokens                 | table | kong
 public | plugins                       | table | kong
 public | ratelimiting_metrics          | table | kong
 public | response_ratelimiting_metrics | table | kong
 public | routes                        | table | kong
 public | schema_meta                   | table | kong
 public | services                      | table | kong
 public | sessions                      | table | kong
 public | snis                          | table | kong
 public | tags                          | table | kong
 public | targets                       | table | kong
 public | ttls                          | table | kong
 public | upstreams                     | table | kong
(26 rows)

~> \q
```

## 7. Kongサービスを起動する

### Kongサービス起動

```command.sh
~$ kong start
```

### 動作起動

```command.sh
~$ curl -i http://localhost:8001/
```

## 8. ルーティングを登録する

### サービス登録

1. 登録は`8001`ポートとなる
2. `https://news.yahoo.co.jp/pickup/rss.xml`を`service-yahoo-news-rss`として保存する

```command.sh
~$ curl -i -X POST \
--url http://localhost:8001/services/ \
--data 'name=service-yahoo-news-rss' \
--data 'url=https://news.yahoo.co.jp/pickup/rss.xml'
```

### ルート登録

1. 登録は`8001`ポートとなる
2. `service-yahoo-news-rss`を`route-yahoo-news-rss`として保存する

```command.sh
~$ curl -i -X POST \
--url http://localhost:8001/services/service-yahoo-news-rss/routes \
--data 'hosts[]=route-yahoo-news-rss'
```

## 9. ルーティングを確認する

### Kong使用

1. 接続は`8000`ポートとなる
2. `http://192.168.33.77:8000/`にヘッダー`Host: route-yahoo-news-rss`を設定してコールする

```command.sh
~$ curl -i -X GET \
--url http://192.168.33.77:8000/ \
--header 'Host: route-yahoo-news-rss'
```

### Kong未使用

1. `https://news.yahoo.co.jp/pickup/rss.xml`を直接コールする

```command.sh
~$ curl -i -X GET \
--url https://news.yahoo.co.jp/pickup/rss.xml
```

## 10. まとめ

1. コンテンツに関しては、同じ結果であることを確認
2. ルーティングの違いに伴い、一部のヘッダーに差異があることを確認

| Kong使用                     | Kong未使用                                                                |
| :--------------------------- | :------------------------------------------------------------------------ |
| Via: kong/2.0.4              | Via: http/1.1 edge2879.img.kth.yahoo.co.jp (ApacheTrafficServer [c sSf ]) |
| X-Kong-Upstream-Latency: 260 |                                                                           |
| X-Kong-Proxy-Latency: 30     |                                                                           |
