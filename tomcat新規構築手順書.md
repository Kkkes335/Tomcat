# Tomcat 新規構築手順書（Amazon Linux 2023）

> 対象: Amazon Linux 2023 上で Tomcat + Apache httpd + Knowledge を新規構築し、`/knowledge` で表示する
>
> この手順書は **新規構築向け** です。既存の Tomcat / Knowledge 環境を修正する手順ではなく、
> **まっさらな EC2 に初めて構築する** ことを前提にしています。

---

## 0. この手順のポイント

今回の手順では、以下の構成で構築します。

- Java: Amazon Corretto 8
- Tomcat: Apache Tomcat 9.0.116
- Web サーバ: Apache httpd
- アプリ: Knowledge v1.13.1

### この構成にする理由

- Knowledge の公開版は比較的古く、演習では **Tomcat 9 + Java 8** の組み合わせが安全です。
- Apache httpd を前段に置き、利用者は `http://グローバルIP/knowledge` でアクセスします。
- Knowledge の保存先を明示するため、`KNOWLEDGE_HOME=/home/tomcat/.knowledge` を設定します。

---

## 1. 前提条件

- Amazon Linux 2023 の EC2 インスタンスであること
- インターネットへ接続できること
- セキュリティグループで少なくとも以下が許可されていること
  - SSH: `22/tcp`
  - HTTP: `80/tcp`
- 作業は root 権限で実施すること

> 補足:
> この手順では Apache httpd が 80 番ポートで待ち受け、Tomcat は 8080 番ポートでローカル待受します。
> そのため通常は **8080 をセキュリティグループで開ける必要はありません**。

---

## 2. サーバへログインして root に切り替える

```bash
sudo su -
cd /root
```

---

## 3. 必要パッケージをインストールする

```bash
dnf install -y wget tar httpd java-1.8.0-amazon-corretto-devel
```

確認します。

```bash
java -version
javac -version
readlink -f /etc/alternatives/java_sdk
```

---

## 4. tomcat ユーザを作成する

> Knowledge は Tomcat 起動ユーザのホーム配下に `.knowledge` を作成するため、
> ホームディレクトリを `/home/tomcat` として作成します。

すでに `tomcat` ユーザが存在しないことを確認します。

```bash
id tomcat
getent passwd tomcat
```

存在しない場合は、以下で作成します。

```bash
useradd -r -m -d /home/tomcat -s /sbin/nologin tomcat
```

確認します。

```bash
getent passwd tomcat
ls -ld /home/tomcat
```

最後の項目が `/home/tomcat` になっていることを確認します。

> もし `tomcat` ユーザがすでに存在する場合は、
> この手順書どおりの新規構築前提から外れるため、既存環境の影響有無を確認してから進めてください。

---

## 5. Knowledge の保存先を作成する

```bash
mkdir -p /home/tomcat/.knowledge
chown -R tomcat:tomcat /home/tomcat
chmod 750 /home/tomcat
chmod 750 /home/tomcat/.knowledge
```

確認します。

```bash
ls -ld /home/tomcat /home/tomcat/.knowledge
```

---

## 6. Tomcat 9.0.116 をインストールする

```bash
mkdir -p /usr/local/src
cd /usr/local/src
wget https://downloads.apache.org/tomcat/tomcat-9/v9.0.116/bin/apache-tomcat-9.0.116.tar.gz
ls -l apache-tomcat-9.0.116.tar.gz
```

展開・配置します。

```bash
tar -xzf apache-tomcat-9.0.116.tar.gz
mv apache-tomcat-9.0.116 /usr/local/
ln -s /usr/local/apache-tomcat-9.0.116 /usr/local/tomcat
```

所有者・権限を設定します。

```bash
chown -R tomcat:tomcat /usr/local/apache-tomcat-9.0.116
chown -h tomcat:tomcat /usr/local/tomcat
chmod +x /usr/local/tomcat/bin/*.sh
```

確認します。

```bash
ls -ld /usr/local/apache-tomcat-9.0.116 /usr/local/tomcat
```

---

## 7. setenv.sh を作成する

> ここで **`KNOWLEDGE_HOME`** を明示します。

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
export KNOWLEDGE_HOME=/home/tomcat/.knowledge
export CATALINA_OPTS="-Xms128m -Xmx512m -Djava.security.egd=file:/dev/urandom -Dfile.encoding=UTF-8 -Dnet.sf.ehcache.skipUpdateCheck=true"
```

権限を付与します。

```bash
chmod 755 /usr/local/tomcat/bin/setenv.sh
chown tomcat:tomcat /usr/local/tomcat/bin/setenv.sh
```

内容確認。

```bash
cat /usr/local/tomcat/bin/setenv.sh
```

---

## 8. Tomcat の systemd サービスを作成する

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
Environment=CATALINA_HOME=/usr/local/tomcat
Environment=CATALINA_BASE=/usr/local/tomcat
Environment=CATALINA_PID=/usr/local/tomcat/temp/tomcat.pid
Environment=KNOWLEDGE_HOME=/home/tomcat/.knowledge
ExecStart=/usr/local/tomcat/bin/startup.sh
ExecStop=/usr/local/tomcat/bin/shutdown.sh
SuccessExitStatus=143
Restart=on-failure
RestartSec=5
UMask=0027

[Install]
WantedBy=multi-user.target
```

反映します。

```bash
systemctl daemon-reload
systemctl enable tomcat
```

---

## 9. Knowledge を配備する

```bash
cd /usr/local/tomcat/webapps
wget https://github.com/support-project/knowledge/releases/download/v1.13.1/knowledge.war -O knowledge.war
chown tomcat:tomcat knowledge.war
ls -l knowledge.war
```

---

## 10. Tomcat を起動して確認する

Tomcat を起動します。

```bash
systemctl start tomcat
systemctl status tomcat --no-pager
```

アプリの展開に少し時間がかかるため、少し待ってから確認します。

```bash
sleep 10
```

### 10-1. 8080 の疎通確認

```bash
curl -I http://127.0.0.1:8080/
```

期待値: `HTTP/1.1 200`

### 10-2. Knowledge の確認

```bash
curl -I http://127.0.0.1:8080/knowledge
curl -I http://127.0.0.1:8080/knowledge/
```

期待値:

- `200`
- または `302`

### 10-3. 展開確認

```bash
ls -l /usr/local/tomcat/webapps | grep knowledge
```

期待値:

- `knowledge.war`
- `knowledge/`

の両方が見えること。

### 10-4. `.knowledge` ディレクトリ確認

```bash
ls -la /home/tomcat
ls -la /home/tomcat/.knowledge
```

---

## 11. エラーがある場合はログを確認する

```bash
journalctl -u tomcat -n 100 --no-pager
```

```bash
tail -n 100 /usr/local/tomcat/logs/catalina.out
```

```bash
tail -n 100 /usr/local/tomcat/logs/localhost*.log
```

エラーだけ絞って見る場合:

```bash
grep -Ei "knowledge|exception|error|severe" /usr/local/tomcat/logs/catalina.out /usr/local/tomcat/logs/localhost*.log 2>/dev/null
```

---

## 12. Apache httpd にリバースプロキシ設定を入れる

```bash
vi /etc/httpd/conf.d/knowledge-proxy.conf
```

以下を記載します。

```apache
ProxyRequests Off
ProxyPreserveHost On

ProxyPass        /knowledge  http://127.0.0.1:8080/knowledge
ProxyPassReverse /knowledge  http://127.0.0.1:8080/knowledge
ProxyPass        /knowledge/ http://127.0.0.1:8080/knowledge/
ProxyPassReverse /knowledge/ http://127.0.0.1:8080/knowledge/
```

> `/knowledge` と `/knowledge/` の両方を書いています。
> 末尾スラッシュあり・なしの不整合を避けるためです。

構文チェックします。

```bash
httpd -t
```

問題なければ起動します。

```bash
systemctl enable httpd
systemctl restart httpd
systemctl status httpd --no-pager
```

> `Invalid command 'ProxyPass'` のようなエラーが出る場合は、
> proxy 関連モジュールの読み込み状態を確認してください。

---

## 13. Apache 経由で動作確認する

ローカル確認:

```bash
curl -I http://127.0.0.1/knowledge
curl -I http://127.0.0.1/knowledge/
```

ブラウザ確認:

```text
http://グローバルIP/knowledge
http://グローバルIP/knowledge/
```

必要なら 8080 直アクセスも確認:

```text
http://グローバルIP:8080/knowledge
http://グローバルIP:8080/knowledge/
```

---

## 14. うまくいかない場合の確認ポイント

### 14-1. Tomcat が起動していない

```bash
systemctl status tomcat --no-pager
journalctl -u tomcat -n 100 --no-pager
```

### 14-2. Knowledge が展開されていない

```bash
ls -l /usr/local/tomcat/webapps | grep knowledge
```

### 14-3. `KNOWLEDGE_HOME` が反映されていない

```bash
cat /usr/local/tomcat/bin/setenv.sh
systemctl cat tomcat
```

### 14-4. Apache 設定に誤りがある

```bash
httpd -t
systemctl status httpd --no-pager
```

### 14-5. セキュリティグループで 80 番が開いていない

ブラウザでアクセスできない場合は、EC2 のセキュリティグループで `80/tcp` を確認してください。

---

## 15. まとめ

この手順書では、Knowledge を動かすために必要な以下の条件を最初から組み込んでいます。

- `tomcat` ユーザのホームディレクトリを `/home/tomcat` として作成する
- `KNOWLEDGE_HOME=/home/tomcat/.knowledge` を設定する
- Tomcat 9 + Java 8 の組み合わせで構築する
- Apache httpd から `/knowledge` を Tomcat へプロキシする

そのため、`http://グローバルIP/knowledge` で Knowledge のログイン画面が表示されれば、新規構築は完了です。
