# Tomcat 演習手順書（修正版 / Amazon Linux 2023）

> 対象: Amazon Linux 2023 上で Tomcat + Apache httpd + Knowledge を構築し、`/knowledge` で表示する
>
> この版は、前回の手順で発生した `http://127.0.0.1:8080/knowledge/` の `404` を踏まえて、
> **Knowledge の保存先設定**、**Tomcat サービス定義**、**再配備手順**、**切り戻し手順** を追加した差し替え版です。

---

## 0. この手順のポイント

今回の手順では、以下の構成で構築します。

- Java: Amazon Corretto 8
- Tomcat: Apache Tomcat 9.0.116
- Web サーバ: Apache httpd
- アプリ: Knowledge v1.13.1

### この構成にする理由

- Knowledge の公式案内は **Java 8 以降 / Tomcat 8.0 以降** です。
- Tomcat 10 以降は Jakarta EE 系へ移行しているため、古い Java EE 系 WAR はそのままでは動かない場合があります。
- Knowledge 公開版は古いため、演習では **Tomcat 9 + Java 8** が安全です。

---

## 1. サーバへログインして root に切り替える

```bash
sudo su -
cd /root
```

---

## 2. 事前バックアップを取得する（切り戻し用）

> **必ず最初に実施してください。**
> 後で修正前の状態に戻したい場合、このバックアップを使います。

```bash
BK="/root/tomcat_fix_backup_$(date +%Y%m%d%H%M%S)"
mkdir -p "$BK"/{systemd,tomcat_bin,tomcat_conf,httpd_conf,webapps}

echo "Backup dir: $BK"

cp -a /etc/systemd/system/tomcat.service "$BK/systemd/" 2>/dev/null || true
cp -a /usr/local/tomcat/bin/setenv.sh "$BK/tomcat_bin/" 2>/dev/null || true
cp -a /usr/local/tomcat/conf/server.xml "$BK/tomcat_conf/" 2>/dev/null || true
cp -a /etc/httpd/conf.d/knowledge-proxy.conf "$BK/httpd_conf/" 2>/dev/null || true
cp -a /usr/local/tomcat/webapps/knowledge* "$BK/webapps/" 2>/dev/null || true
getent passwd tomcat > "$BK/tomcat_user.txt" 2>/dev/null || true
systemctl is-enabled tomcat > "$BK/tomcat_enabled.txt" 2>/dev/null || true
systemctl is-enabled httpd > "$BK/httpd_enabled.txt" 2>/dev/null || true
```

バックアップ先をメモしておきます。

```bash
echo "$BK"
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

## 4. tomcat ユーザを正しく作成する

> 前回版では `tomcat` ユーザにホームディレクトリを明示していませんでした。
> Knowledge は Tomcat 起動ユーザのホーム配下に `.knowledge` を作るため、ここを明示します。

既存の `tomcat` ユーザがあるか確認します。

```bash
id tomcat
getent passwd tomcat
```

### 4-1. tomcat ユーザが存在しない場合

```bash
useradd -r -m -d /home/tomcat -s /sbin/nologin tomcat
```

### 4-2. tomcat ユーザが既に存在する場合

> ホームディレクトリが `/home/tomcat` でない場合は修正します。

```bash
usermod -d /home/tomcat tomcat
mkdir -p /home/tomcat
chown tomcat:tomcat /home/tomcat
chmod 750 /home/tomcat
```

確認します。

```bash
getent passwd tomcat
```

最後の項目が `/home/tomcat` になっていることを確認します。

---

## 5. Knowledge の保存先を作成する

```bash
mkdir -p /home/tomcat/.knowledge
chown -R tomcat:tomcat /home/tomcat/.knowledge
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
ln -sfn /usr/local/apache-tomcat-9.0.116 /usr/local/tomcat
```

所有者・権限を設定します。

```bash
chown -R tomcat:tomcat /usr/local/apache-tomcat-9.0.116
chown -h tomcat:tomcat /usr/local/tomcat
chmod +x /usr/local/tomcat/bin/*.sh
```

---

## 7. setenv.sh を作成する

> ここで **`KNOWLEDGE_HOME`** を明示します。
> これが今回の差し替え版の重要修正点です。

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

## 9. 旧配備物を削除してから Knowledge を再配備する

> すでに `knowledge.war` や展開済み `knowledge/` がある場合、
> そのまま上書きせず一度消してから再配備します。

まず停止します。

```bash
systemctl stop tomcat
```

旧配備物・作業キャッシュを削除します。

```bash
rm -rf /usr/local/tomcat/webapps/knowledge
rm -f /usr/local/tomcat/webapps/knowledge.war
rm -rf /usr/local/tomcat/work/Catalina/localhost/knowledge*
rm -rf /usr/local/tomcat/temp/*
```

WAR を再取得して配置します。

```bash
cd /usr/local/tomcat/webapps
wget https://github.com/support-project/knowledge/releases/download/v1.13.1/knowledge.war -O knowledge.war
chown tomcat:tomcat knowledge.war
ls -l knowledge.war
```

Tomcat を起動します。

```bash
systemctl start tomcat
systemctl status tomcat --no-pager
```

---

## 10. Tomcat 単体で動作確認する

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

### 10-4. `.knowledge` ディレクトリ作成確認

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
grep -Ei "knowledge|exception|error|severe" /usr/local/tomcat/logs/catalina.out /usr/local/tomcat/logs/localhost*.log
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

## 14. この修正版で直した点

前回版から、次を修正しています。

1. `tomcat` ユーザのホームディレクトリを明示した
2. `KNOWLEDGE_HOME=/home/tomcat/.knowledge` を設定した
3. `knowledge.war` を置く前に旧配備物とキャッシュを削除するようにした
4. Apache の `ProxyPass` を末尾スラッシュあり・なし両対応にした
5. 切り戻し用のバックアップ取得手順を先頭に追加した

---

## 15. 切り戻し手順

> この手順は、**この修正版を試したあとに修正前の状態へ戻したい場合**に使います。
> 手順 2 で控えたバックアップディレクトリを使います。

以下では例としてバックアップディレクトリを変数に入れています。

```bash
BK="/root/tomcat_fix_backup_YYYYMMDDHHMMSS"
```

実際には、あなたが手順 2 で作成したディレクトリ名に置き換えてください。

### 15-1. サービス停止

```bash
systemctl stop httpd 2>/dev/null || true
systemctl stop tomcat 2>/dev/null || true
```

### 15-2. 今回追加・変更した Knowledge データを退避する

> 修正版で新たに `.knowledge` が作成された場合、念のため退避します。

```bash
if [ -d /home/tomcat/.knowledge ]; then
  mv /home/tomcat/.knowledge "/home/tomcat/.knowledge.rollback.$(date +%Y%m%d%H%M%S)"
fi
```

### 15-3. 設定ファイルを元に戻す

```bash
cp -af "$BK/systemd/tomcat.service" /etc/systemd/system/ 2>/dev/null || true
cp -af "$BK/tomcat_bin/setenv.sh" /usr/local/tomcat/bin/ 2>/dev/null || true
cp -af "$BK/tomcat_conf/server.xml" /usr/local/tomcat/conf/ 2>/dev/null || true
cp -af "$BK/httpd_conf/knowledge-proxy.conf" /etc/httpd/conf.d/ 2>/dev/null || true
```

### 15-4. 配備物を元に戻す

```bash
rm -rf /usr/local/tomcat/webapps/knowledge
rm -f /usr/local/tomcat/webapps/knowledge.war

cp -af "$BK/webapps/knowledge" /usr/local/tomcat/webapps/ 2>/dev/null || true
cp -af "$BK/webapps/knowledge.war" /usr/local/tomcat/webapps/ 2>/dev/null || true
```

### 15-5. Tomcat の作業キャッシュを掃除する

```bash
rm -rf /usr/local/tomcat/work/Catalina/localhost/knowledge*
rm -rf /usr/local/tomcat/temp/*
```

### 15-6. tomcat ユーザのホームを元に戻したい場合

> ここは **必要な場合だけ** 実施してください。
> 手順 2 の `tomcat_user.txt` に修正前の情報があります。

確認:

```bash
cat "$BK/tomcat_user.txt"
```

もし修正前のホームディレクトリへ戻したいなら、`usermod -d` で戻します。

例:

```bash
# 例: 修正前が /var/lib/tomcat だった場合
usermod -d /var/lib/tomcat tomcat
```

### 15-7. systemd 再読込とサービス起動

```bash
systemctl daemon-reload
systemctl restart tomcat 2>/dev/null || systemctl start tomcat
systemctl restart httpd 2>/dev/null || systemctl start httpd
```

### 15-8. 切り戻し確認

```bash
systemctl status tomcat --no-pager
systemctl status httpd --no-pager
```

```bash
curl -I http://127.0.0.1:8080/
curl -I http://127.0.0.1:8080/knowledge/
curl -I http://127.0.0.1/knowledge/
```

---

## 16. まとめ

今回の 404 は、単純な Tomcat 起動失敗ではなく、
**Knowledge 用の実行条件や再配備条件を明示していなかったこと**が原因候補でした。

そのため、この修正版では以下を必須化しています。

- `tomcat` ユーザのホームディレクトリ明示
- `KNOWLEDGE_HOME` 明示
- 旧配備物削除後の再配備
- Apache のプロキシ定義見直し
- 切り戻し用バックアップの先行取得

