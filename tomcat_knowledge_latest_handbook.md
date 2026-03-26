# Tomcat 演習手順書（Amazon Linux 2023 / 2026-03 時点）

> 対象: Amazon Linux 2023 上に Tomcat と Apache httpd を構築し、Knowledge を `/knowledge` で公開する
>
> この手順書は、**古い固定バージョン指定**と**誤った systemd 設定**を見直し、**現在でも動かしやすい構成**に修正したものです。

---

## 先に結論

今回の手順では、以下の組み合わせで構築します。

- Java: **Amazon Corretto 8**
- Tomcat: **Apache Tomcat 9.0.116**
- Web サーバ: **Apache httpd**
- アプリ: **Knowledge v1.13.1**

### この構成にした理由

- Knowledge の公開最新版は **v1.13.1** のままです。
- Knowledge の公式インストール説明は「Java 8 以降」「Tomcat 8.0 以降」となっていますが、公開版が古いため、Tomcat 10/11 や Java 17/21 へそのまま上げると互換性で詰まりやすいです。
- そのため、**現行の入手元から取得できる範囲で、Java EE 系アプリと相性がよい Tomcat 9 系 + Java 8** で構成します。

---

## 1. サーバにログインして root に切り替える

```bash
sudo su -
cd /root
```

必要な基本コマンドを入れておきます。

```bash
dnf install -y wget tar httpd
```

---

## 2. JDK をインストールする

Amazon Linux 2023 では、古い tar.gz を手動展開するより、**パッケージで入れる方法**のほうが安全です。

```bash
dnf install -y java-1.8.0-amazon-corretto-devel
```

### インストール確認

```bash
java -version
javac -version
readlink -f /etc/alternatives/java_sdk
```

### 確認ポイント

- `java -version` に `Corretto` が含まれていること
- `/etc/alternatives/java_sdk` が存在すること

> 以降の `JAVA_HOME` は `/etc/alternatives/java_sdk` を使います。
> これにより、細かいビルド番号付きのディレクトリ名を手で書かずに済みます。

---

## 3. Tomcat ユーザを作成する

```bash
id tomcat 2>/dev/null || useradd -r -s /sbin/nologin tomcat
```

---

## 4. Tomcat をインストールする

作業用ディレクトリへ移動します。

```bash
mkdir -p /usr/local/src
cd /usr/local/src
```

Tomcat 9.0.116 をダウンロードします。

```bash
wget https://downloads.apache.org/tomcat/tomcat-9/v9.0.116/bin/apache-tomcat-9.0.116.tar.gz
ls -l apache-tomcat-9.0.116.tar.gz
```

展開して配置します。

```bash
tar -xzf apache-tomcat-9.0.116.tar.gz
mv apache-tomcat-9.0.116 /usr/local/
ln -sfn /usr/local/apache-tomcat-9.0.116 /usr/local/tomcat
```

所有者を変更します。

```bash
chown -R tomcat:tomcat /usr/local/apache-tomcat-9.0.116
chown -h tomcat:tomcat /usr/local/tomcat
chmod +x /usr/local/tomcat/bin/*.sh
```

---

## 5. setenv.sh を作成する

Tomcat の環境変数は `setenv.sh` にまとめます。

```bash
vi /usr/local/tomcat/bin/setenv.sh
```

以下を記載します。

```sh
#!/bin/sh
export JAVA_HOME=/etc/alternatives/java_sdk
export CATALINA_HOME=/usr/local/tomcat
export CATALINA_BASE=/usr/local/tomcat
export CATALINA_PID=/usr/local/tomcat/temp/tomcat.pid
export CATALINA_OPTS="-Xms128m -Xmx512m -Djava.security.egd=file:/dev/urandom"
```

保存後、権限を付与します。

```bash
chmod 755 /usr/local/tomcat/bin/setenv.sh
```

---

## 6. Tomcat の systemd サービスを作成する

> 旧手順の `Type=oneshot` や `ExecReStart` は不適切です。
> Tomcat は通常 `startup.sh` / `shutdown.sh` を使うため、`Type=forking` で定義します。

```bash
vi /etc/systemd/system/tomcat.service
```

以下を記載します。

```ini
[Unit]
Description=Apache Tomcat 9
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat
Environment=JAVA_HOME=/etc/alternatives/java_sdk
Environment=CATALINA_PID=/usr/local/tomcat/temp/tomcat.pid
Environment=CATALINA_HOME=/usr/local/tomcat
Environment=CATALINA_BASE=/usr/local/tomcat
ExecStart=/usr/local/tomcat/bin/startup.sh
ExecStop=/usr/local/tomcat/bin/shutdown.sh
SuccessExitStatus=143
Restart=on-failure
RestartSec=5
UMask=0027

[Install]
WantedBy=multi-user.target
```

systemd に読み込ませます。

```bash
systemctl daemon-reload
systemctl enable tomcat
```

---

## 7. Knowledge アプリを配備する

> 旧手順では `jar xf` で手動展開していましたが、Knowledge の公式説明は **`knowledge.war` を webapps に置く** 方法です。
> 今回はその方法で配備します。

```bash
cd /tmp
wget https://github.com/support-project/knowledge/releases/download/v1.13.1/knowledge.war
ls -l knowledge.war
```

Tomcat の `webapps` 配下へ配置します。

```bash
cp knowledge.war /usr/local/tomcat/webapps/
chown tomcat:tomcat /usr/local/tomcat/webapps/knowledge.war
```

Tomcat を起動します。

```bash
systemctl start tomcat
systemctl status tomcat --no-pager
```

### Tomcat 単体の動作確認

```bash
curl -I http://127.0.0.1:8080/
curl -I http://127.0.0.1:8080/knowledge/
```

`HTTP/1.1 200` や `302` が返ればひとまず正常です。

---

## 8. Apache httpd にリバースプロキシ設定を入れる

> 旧手順では `httpd.conf` の末尾へ直接追記していましたが、`conf.d` に分離したほうが管理しやすいです。

```bash
vi /etc/httpd/conf.d/knowledge-proxy.conf
```

以下を記載します。

```apache
ProxyPreserveHost On
ProxyPass /knowledge http://127.0.0.1:8080/knowledge
ProxyPassReverse /knowledge http://127.0.0.1:8080/knowledge
```

構文チェックを行います。

```bash
httpd -t
```

問題がなければ httpd を起動します。

```bash
systemctl enable httpd
systemctl start httpd
systemctl status httpd --no-pager
```

---

## 9. 動作確認

### 9-1. Apache 経由のローカル確認

```bash
curl -I http://127.0.0.1/knowledge/
```

### 9-2. ブラウザ確認

ブラウザから以下へアクセスします。

```text
http://グローバルIP/knowledge/
```

### 9-3. 8080 直アクセスを外部から確認したい場合

ブラウザから以下へアクセスします。

```text
http://グローバルIP:8080/knowledge/
```

> ただし、外部から 8080 を確認するには、EC2 のセキュリティグループで TCP/8080 を許可している必要があります。
> 通常は Apache 経由の 80 番だけ確認できれば十分です。

---

## 10. うまくいかない時の確認ポイント

### 10-1. Tomcat ログ確認

```bash
journalctl -u tomcat -e --no-pager
```

```bash
tail -n 100 /usr/local/tomcat/logs/catalina.out
```

### 10-2. Apache ログ確認

```bash
journalctl -u httpd -e --no-pager
```

```bash
tail -n 100 /var/log/httpd/error_log
```

### 10-3. ポート確認

```bash
ss -lntp | egrep ':80|:8080'
```

想定例:

- `:8080` を Tomcat が待ち受けている
- `:80` を httpd が待ち受けている

---

## 11. この手順書で修正した主な問題点

旧手順から、以下を修正しています。

1. **古い固定バージョンの tar.gz 展開をやめた**  
   - Corretto はパッケージで導入
   - `JAVA_HOME` は `/etc/alternatives/java_sdk` を使用

2. **Tomcat のメジャーバージョン選定を見直した**  
   - Tomcat 10/11 ではなく、Knowledge と相性のよい Tomcat 9 を採用

3. **`server.xml` の不要変更をやめた**  
   - `autoDeploy` / `unpackWARs` を無理に変更しない
   - `knowledge.war` を `webapps` に置く標準手順へ変更

4. **誤った systemd 設定を修正した**  
   - `Type=oneshot` → `Type=forking`
   - `ExecReStart` の誤記を除去
   - `systemctl daemon-reload` を追加

5. **Apache の設定場所を整理した**  
   - `httpd.conf` 直編集ではなく `conf.d/knowledge-proxy.conf` に分離

---

## 12. 完成コマンド一覧（まとめて見たい場合）

```bash
sudo su -
cd /root

dnf install -y wget tar httpd java-1.8.0-amazon-corretto-devel

id tomcat 2>/dev/null || useradd -r -s /sbin/nologin tomcat

mkdir -p /usr/local/src
cd /usr/local/src
wget https://downloads.apache.org/tomcat/tomcat-9/v9.0.116/bin/apache-tomcat-9.0.116.tar.gz
tar -xzf apache-tomcat-9.0.116.tar.gz
mv apache-tomcat-9.0.116 /usr/local/
ln -sfn /usr/local/apache-tomcat-9.0.116 /usr/local/tomcat
chown -R tomcat:tomcat /usr/local/apache-tomcat-9.0.116
chown -h tomcat:tomcat /usr/local/tomcat
chmod +x /usr/local/tomcat/bin/*.sh

cat > /usr/local/tomcat/bin/setenv.sh <<'EOS'
#!/bin/sh
export JAVA_HOME=/etc/alternatives/java_sdk
export CATALINA_HOME=/usr/local/tomcat
export CATALINA_BASE=/usr/local/tomcat
export CATALINA_PID=/usr/local/tomcat/temp/tomcat.pid
export CATALINA_OPTS="-Xms128m -Xmx512m -Djava.security.egd=file:/dev/urandom"
EOS
chmod 755 /usr/local/tomcat/bin/setenv.sh

cat > /etc/systemd/system/tomcat.service <<'EOS'
[Unit]
Description=Apache Tomcat 9
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat
Environment=JAVA_HOME=/etc/alternatives/java_sdk
Environment=CATALINA_PID=/usr/local/tomcat/temp/tomcat.pid
Environment=CATALINA_HOME=/usr/local/tomcat
Environment=CATALINA_BASE=/usr/local/tomcat
ExecStart=/usr/local/tomcat/bin/startup.sh
ExecStop=/usr/local/tomcat/bin/shutdown.sh
SuccessExitStatus=143
Restart=on-failure
RestartSec=5
UMask=0027

[Install]
WantedBy=multi-user.target
EOS

systemctl daemon-reload
systemctl enable tomcat

cd /tmp
wget https://github.com/support-project/knowledge/releases/download/v1.13.1/knowledge.war
cp knowledge.war /usr/local/tomcat/webapps/
chown tomcat:tomcat /usr/local/tomcat/webapps/knowledge.war
systemctl start tomcat

cat > /etc/httpd/conf.d/knowledge-proxy.conf <<'EOS'
ProxyPreserveHost On
ProxyPass /knowledge http://127.0.0.1:8080/knowledge
ProxyPassReverse /knowledge http://127.0.0.1:8080/knowledge
EOS

httpd -t
systemctl enable httpd
systemctl start httpd

curl -I http://127.0.0.1:8080/knowledge/
curl -I http://127.0.0.1/knowledge/
```

---

## 13. 補足

この手順は **「まず演習で動かす」** ことを優先した最小構成です。
本番運用を意識する場合は、少なくとも以下は別途検討してください。

- HTTPS 化
- Apache 側のアクセス制御
- Tomcat マネージャアプリの削除または制限
- ログローテーション
- バックアップ
- 外部 DB 利用
- SELinux / Firewall / Security Group の見直し

