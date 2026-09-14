# コンテナ化デプロイ

GitHub Pages を使用せずにブックコレクションをセルフホストするための任意（オプトイン）の方法です。ファイアウォール内や内部ネットワークでの利用に便利です。

**これはローカル開発用ではありません。** 執筆やプレビューには `cargo xtask serve` を使用してください。コンテナを使用せずに再ビルドし、<http://localhost:3000> で配信します。

## 使い方

リポジトリのルートから以下を実行します:

```bash
docker compose -f docker/compose.yaml up --build
```

起動後、<http://localhost:3000> を開きます。ホスト側のポートは `PORT` で上書きできます:

```bash
PORT=8080 docker compose -f docker/compose.yaml up --build
```

Compose を使用しない場合:

```bash
docker build -f docker/Dockerfile -t rust-training .
docker run --rm -p 3000:8080 rust-training
```

いずれの場合もビルドコンテキストはリポジトリルートである点に注意してください。ビルドにはブックのソースコードと `xtask` クレートが必要です。

## 動作の仕組み

2つのステージで構成されています:

1. **builder** (`rust:1-slim-bookworm`): `mdbook` と `mdbook-mermaid` をインストールし、`cargo xtask build` を実行して、生成されたランディングページとともに7冊すべてのブックを `site/` にビルドします。
2. **runtime** (`nginxinc/nginx-unprivileged:alpine`): ポート 8080 で `site/` を配信します。最終イメージには Rust ツールチェーン、mdbook、ブックのソースコードは含まれません。

`xtask deploy` ではなく `xtask build` を使用しているのは、両者が生成するコンテンツが同一であるためです。`deploy` の違いは、`docs/` への出力と、コンテナ環境では不要な GitHub Pages 向けの手順を出力することのみです。

## バージョンの固定（ピン留め）

`MDBOOK_VERSION` と `MDBOOK_MERMAID_VERSION` は Dockerfile 内のビルド引数（build args）です。
CI（`pages.yml`）は現在 `cargo install` を介して両方をバージョン未指定でインストールしているため、アップストリームの mdbook リリース後、コンテナ側のバージョンが進んでいたり遅れていたりする可能性があります。必要に応じて引数の値を更新してください。

アップストリームで配布されている場合はビルド済みリリースバイナリを使用し、そうでない場合は `cargo install` にフォールバックします。固定されているバージョン時点では、`mdbook-mermaid` には arm64 Linux 向けの公開バイナリが存在しないため、arm64 ビルドではソースからコンパイルが行われ、所要時間が著しく長くなります。

## 注意事項

- コンテナは UID 101 として実行され、非特権ポートにバインドされるため、root 権限や追加のケーパビリティは不要です。
- サービスに `read_only: true` を追加することは可能ですが、nginx のキャッシュおよび PID パス用に tmpfs マウントが必要になります。未テストのまま提供するのを避けるため、デフォルトでは無効になっています。
- コンテンツはビルド時にイメージ内に組み込まれます。ブックの変更を反映するには、イメージを再ビルドしてください。
