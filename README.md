# renew-checker

td-agent（Fluentd）から OpenSearch へログを送信できることを、Docker Compose で確認するためのサンプル環境です。

## 構成

- OpenSearch: `http://localhost:9200`
- OpenSearch Dashboards: `http://localhost:5601`
- Fluentd: Forward protocol `24224/tcp` / `24224/udp`
- Fluentd から OpenSearch に送信されたログは `td-agent-test-YYYY.MM.DD` 形式のインデックスに保存されます。

## 前提条件

Docker と Docker Compose が利用できることを確認してください。

```bash
docker --version
docker compose version
```

## 1. 起動

リポジトリを clone して、Docker Compose を起動します。

```bash
git clone https://github.com/rchaser53/renew-checker.git
cd renew-checker

docker compose up -d --build
```

コンテナの状態を確認します。

```bash
docker compose ps
```

`opensearch`、`opensearch-dashboards`、`fluentd` が起動していれば次に進みます。

起動時に問題がある場合はログを確認します。

```bash
docker compose logs
```

特定のサービスだけ確認する場合:

```bash
docker compose logs opensearch
docker compose logs fluentd
```

## 2. OpenSearch の動作確認

OpenSearch にアクセスします。

```bash
curl http://localhost:9200
```

OpenSearch のバージョン情報などを含む JSON が返れば正常です。

クラスタの状態も確認できます。

```bash
curl 'http://localhost:9200/_cluster/health?pretty'
```

## 3. Fluentd からテストログを送信

Fluentd コンテナ内の `fluent-cat` を使ってテストログを送ります。

```bash
echo '{"message":"Hello OpenSearch"}' | \
  docker exec -i fluentd fluent-cat test.log
```

Fluentd は受信したログを OpenSearch に送信します。設定上、バッファは約1秒で flush されます。

## 4. OpenSearch にインデックスが作成されたことを確認

数秒待ってからインデックス一覧を確認します。

```bash
curl 'http://localhost:9200/_cat/indices?v'
```

次のような名前のインデックスが表示されれば、Fluentd から OpenSearch への送信に成功しています。

```text
td-agent-test-YYYY.MM.DD
```

## 5. 送信したログを確認

OpenSearch の Search API でログを確認します。

```bash
curl 'http://localhost:9200/td-agent-test-*/_search?pretty'
```

検索結果の `_source` に、送信したログが含まれていることを確認してください。

例:

```json
{
  "message": "Hello OpenSearch",
  "fluentd_tag": "test.log"
}
```

これが確認できれば、

```text
fluent-cat
    ↓
Fluentd
    ↓
fluent-plugin-opensearch
    ↓
OpenSearch
```

という一連の経路が正常に動作しています。

## 6. OpenSearch Dashboards へのアクセス

ブラウザで次のアドレスを開きます。

```text
http://localhost:5601
```

OpenSearch Dashboards が表示されれば、Dashboards から OpenSearch への接続も正常です。

ログを Dashboards から検索する場合は、`td-agent-test-*` を対象とした index pattern / data view を作成してください。

## 停止

コンテナを停止・削除します。

```bash
docker compose down
```

OpenSearch のデータも含めて完全に削除する場合は、volume も削除します。

```bash
docker compose down -v
```

## 最短の動作確認手順

一度環境を構築した後は、以下だけでも疎通確認できます。

```bash
docker compose up -d --build

curl http://localhost:9200

echo '{"message":"Hello OpenSearch"}' | \
  docker exec -i fluentd fluent-cat test.log

sleep 2

curl 'http://localhost:9200/_cat/indices?v'
curl 'http://localhost:9200/td-agent-test-*/_search?pretty'
```

`td-agent-test-*` インデックス内に `Hello OpenSearch` が確認できれば動作確認完了です。
