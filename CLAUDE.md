# 開発方針＆開発環境ルール(rs-xorriso)

全リポジトリ共通の開発ルール(自動継続・検証徹底等)は
[`open-raid-z`](https://github.com/aon-co-jp/open-raid-z)の`CLAUDE.md`を
正本として参照すること。この節では本リポジトリ固有の事項のみ記す。

## リポジトリの役割

GNU xorrisoへの敬意を込めたRustでのリスペクト版。`RS-Blog`(WordPressの
Rust版)・`RS-EC`(EC-CUBEのRust版)と同じ位置づけの長期プロジェクトで、
本家との機能差は現時点で大きい。[`make-disk`](https://github.com/aon-co-jp/make-disk)
は現状xorriso本体をラップしているが、本リポジトリが実用段階に達したら
置き換え候補となる。

## アーキテクチャ

- `src/iso9660.rs` — ISO9660(Level 1)イメージのバイナリ生成・単体テスト
- `src/main.rs` — xorriso風の簡易CLI(`-as mkisofs -V <label> -o <out> <src>`)

## 実装済み(2026-09-12時点)

- フラットな(サブディレクトリ無しの)ディレクトリからのISO9660 Level 1
  イメージ生成。ファイル名は8.3形式へサニタイズ。
- 単体テスト4件(CD001マジックバイト・セクタ境界整合・ボリューム
  ラベル埋め込み・ファイル名サニタイズ)全green。
- 実機検証: Windows `Mount-DiskImage`で実際にマウントし、複数ファイルの
  内容が正しく読み出せることを確認済み。

## 未実装・既知の制限

- サブディレクトリ非対応(ルートディレクトリが1セクタに収まる件数まで)
- Rock Ridge/Joliet拡張非対応(長いファイル名・非ASCII名は8.3へ切り詰め)
- ディスクへの実書き込み機能は無い(ISOイメージ生成のみ)
- ファイル名衝突(切り詰め後の重複)の検出・回避は無い

## make-diskとの互換性検証結果(2026-09-12)

`make-disk`が実際に生成する引数(`-as mkisofs -iso-level 3 -J -R -V <label>
-o <out> <src>`)で実機検証した結果:

- **ISO生成は動作する**が、`-J -R`(Joliet/Rock Ridge)は効果が無く、
  長いファイル名は8.3形式へ切り詰められる(警告は出すが処理は継続)。
- **`-as cdrecord`(書き込み)・`-devices`(デバイス列挙)は未実装**。
  make-diskの`burn.rs`/`list_devices()`からは呼び出せない
  (呼び出すと明示的なエラーで失敗する、旧版のような黙った誤動作は無い)。
- 発見時の旧版には、未知のフラグの値(例: `-iso-level`の`3`)を誤って
  ソースディレクトリとして拾ってしまう脆いパーサーの不具合があった
  (引数の並び順に依存する偶然の成功)。修正済み。

結論: 現状は「make-diskのISO生成部分の限定的な代替」であり、
書き込み機能を含む完全な代替ではない。README/webpage側の案内文でも
この限界を明記すること。

## 次にやること候補

- ルートディレクトリの複数セクタ対応(ファイル数の多いケース)
- サブディレクトリ対応(パステーブルの複数エントリ化)
- Rock Ridge拡張(長いファイル名の保持)

## API 方針: REST 全廃・gRPC / Connect へ (2026-11-04)

### 日本語
- **REST は全廃する。新しい API は REST で作らない。** ブラウザや `curl` から呼ぶ API は **Connect の単純呼び出し**(`POST /<パッケージ>.<Service>/<Method>`、`content-type: application/json`、本文は JSON)、サービス間は **gRPC / gRPC-web**(RPoem の管理API `open_runo.admin.v1.Admin` と同じ流儀)。ストリーミングは、必要になってから。
- **既存の REST は、触るときに順に移す。** 移したら旧 REST の経路は消す。移行が終わるまで残るが、**新規には追加しない**。
- **JSON は RS-JSON に統一**する(読み込みは `from_str_secure` / `parse_secure` でサイズ・深さ・重複キーの上限つき、書き出しは `rust_json::fast`)。ブラウザの WASM に `prost` は入れない(サイズが増えるため)。
- **例外(自分たちの API ではなく、相手の規格が決めているもの)**: ACME、OAuth/OIDC のコールバック、第三者(GitHub など)からの Webhook、ヘルスチェック、静的ファイル配信、OTLP など。これらは規格どおりの HTTP を受ける。
- 参考実装: `RPoem`(`open_runo.admin.v1.Admin`)、`open-easy-web`(`server/src/connect.rs`、`proto/open_easy_web/admin/v1/admin.proto`)。
- ※ このリポジトリのコードが既に方針どおりかどうかは、各リポジトリの現状による。この節は方針の記録であり、コードの変更を意味しない。

### English
- **REST is being abolished. New APIs are not built as REST.** APIs called from browsers or `curl` use **Connect unary calls** (`POST /<package>.<Service>/<Method>`, `content-type: application/json`, JSON body); service-to-service calls use **gRPC / gRPC-web** (the same style as RPoem's admin API `open_runo.admin.v1.Admin`). Streaming only when it is actually needed.
- **Existing REST endpoints are migrated one by one, when they are touched.** The old REST route is removed once migrated. Until migration is finished they remain, but **no new ones are added**.
- **JSON is unified on RS-JSON**: reading with `from_str_secure` / `parse_secure` (size, depth and duplicate-key limits), writing with `rust_json::fast`. `prost` is not put into browser WASM (it would grow the binary).
- **Exceptions (shapes dictated by someone else's standard, not our own API)**: ACME, OAuth/OIDC callbacks, webhooks from third parties (e.g. GitHub), health checks, static file serving, OTLP, and so on. These accept plain HTTP as the standard requires.
- Reference implementations: `RPoem` (`open_runo.admin.v1.Admin`) and `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Note: whether the code in a given repository already follows this policy depends on that repository. This section records the policy; it does not mean the code has been changed.

### Deutsch (KI-Übersetzung, noch nicht von Muttersprachlern geprüft)
- **REST wird abgeschafft. Neue APIs werden nicht als REST gebaut.** APIs, die aus dem Browser oder mit `curl` aufgerufen werden, nutzen **Connect-Unary-Aufrufe** (`POST /<Paket>.<Service>/<Methode>`, `content-type: application/json`, JSON-Body); zwischen Diensten gilt **gRPC / gRPC-web** (wie bei der Admin-API von RPoem, `open_runo.admin.v1.Admin`). Streaming erst, wenn es wirklich nötig ist.
- **Bestehende REST-Endpunkte werden einzeln migriert, sobald man sie anfasst.** Nach der Migration wird die alte REST-Route entfernt. Bis dahin bleiben sie bestehen, es werden aber **keine neuen hinzugefügt**.
- **JSON wird auf RS-JSON vereinheitlicht**: Lesen mit `from_str_secure` / `parse_secure` (Grenzen für Größe, Tiefe und doppelte Schlüssel), Schreiben mit `rust_json::fast`. `prost` kommt nicht in Browser-WASM (die Datei würde größer).
- **Ausnahmen (Formate, die ein fremder Standard vorgibt, nicht unsere eigene API)**: ACME, OAuth/OIDC-Callbacks, Webhooks von Dritten (z. B. GitHub), Health-Checks, statische Dateien, OTLP usw. Diese nehmen schlichtes HTTP entgegen, wie es der Standard verlangt.
- Referenzimplementierungen: `RPoem` (`open_runo.admin.v1.Admin`) und `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Hinweis: Ob der Code eines Repositorys diese Richtlinie schon befolgt, hängt vom jeweiligen Repository ab. Dieser Abschnitt hält die Richtlinie fest; er bedeutet nicht, dass der Code bereits geändert wurde.

### Русский (машинный перевод, носителями языка ещё не проверен)
- **REST полностью отменяется. Новые API не создаются как REST.** API, вызываемые из браузера или через `curl`, используют **унарные вызовы Connect** (`POST /<пакет>.<Service>/<Method>`, `content-type: application/json`, тело — JSON); между сервисами — **gRPC / gRPC-web** (так же, как административный API RPoem, `open_runo.admin.v1.Admin`). Потоковая передача — только когда она действительно нужна.
- **Существующие REST-эндпоинты переносятся по одному, когда до них доходят руки.** После переноса старый REST-маршрут удаляется. До завершения переноса они остаются, но **новые не добавляются**.
- **JSON унифицируется на RS-JSON**: чтение через `from_str_secure` / `parse_secure` (ограничения на размер, глубину и повторяющиеся ключи), запись через `rust_json::fast`. `prost` не добавляется в браузерный WASM (размер вырос бы).
- **Исключения (форматы, заданные чужим стандартом, а не наш собственный API)**: ACME, обратные вызовы OAuth/OIDC, вебхуки сторонних сервисов (например, GitHub), проверки работоспособности, раздача статических файлов, OTLP и т. п. Они принимают обычный HTTP, как требует стандарт.
- Эталонные реализации: `RPoem` (`open_runo.admin.v1.Admin`) и `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Примечание: соответствует ли код конкретного репозитория этой политике, зависит от самого репозитория. Этот раздел фиксирует политику и не означает, что код уже изменён.
