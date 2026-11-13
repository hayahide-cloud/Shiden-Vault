# Shiden-Vault
Shiden Vault（紫電パスワードマネージャー）の公開ホスティング（GitHub Pages・秘密情報なし）

## Shiden Vault とは

パスワード・APIトークン・セキュアメモを **端末内で暗号化** して非公開リポジトリ `Hayahide-Log` の `vault.json` に保存する、スマホ用の単一HTMLアプリ。Shiden Assets と同じ技術（単一HTML・GitHub Contents API 同期・localStorage 設定・操作キュー・紫電ダークテーマ）で作られている。

- 公開URL: https://hayahide-cloud.github.io/Shiden-Vault/
- 実データ: 非公開リポジトリ `hayahide-cloud/Hayahide-Log` の `vault.json`（暗号文のみ・各端末のトークンで実行時に読み書き）
- 経緯: `Shiden-Dashboard/vault/` で開発し、2026-11-11 にこのリポジトリへ移設（Shiden Assets と同じ流れ）

## 安全設計

| 層 | 内容 |
| --- | --- |
| 保管庫の暗号化 | マスターパスワード → PBKDF2-HMAC-SHA256（600,000回・16byteソルト）→ AES-256-GCM。鍵は `extractable=false` でメモリにのみ保持し、ロック・自動ロック（既定5分）で破棄 |
| GitHub 上 | `vault.json` は `{ format, kdf, iv, ct }` の暗号文のみ。リポジトリが非公開でも平文は置かない |
| 端末上 | GitHub トークン（接続設定）は localStorage。保管庫の端末コピーと未同期の操作キューは保管庫の鍵で暗号化して保存 |
| この公開ページ | 秘密情報は一切含まない |

マスターパスワードを忘れると復元できない（意図的に復旧手段を持たない）。「保管庫の設定 → 平文JSONをエクスポート」でバックアップが取れる。

## できること

- ログイン（URL・ユーザー名・パスワード）／トークン・APIキー（サービス・権限・有効期限・管理ページ）／クレジットカード（ブランド・名義・番号・有効期限・セキュリティコード・暗証番号。一覧は末尾4桁のみ表示）／セキュアメモ の登録・検索・タグ
- パスワード生成（長さ・数字・記号）、コピー、表示/非表示
- トークンの期限警告（14日前から）・カードの有効期限警告（30日前から）。期限切れは赤
- カードはブランドのマーク（Visa / Mastercard / JCB / Amex / Diners の簡易図案）を一覧に表示。ブランド未選択なら番号の先頭桁から自動判定
- 他端末との同期（操作キュー方式・409 は最新を取り直して再適用）
- マスターパスワード変更（全体を再暗号化。他端末は次回解錠時に新パスワード）

## 紫電アプリ接続（トークンの一元管理）

Shiden Assets / Dashboard / Expense / LifeLogger / Tasks は Hayahide-Log 用のトークンを各自の設定に持っている。このアプリの「紫電アプリ接続」カードから接続情報（owner / repo / branch / token）を配布し、アプリ側でトークンを打ち直さずに済ませる。

受け渡し方法は 3 つ（上から優先）:

1. **この端末に書き込む** — 同一オリジン `hayahide-cloud.github.io` の localStorage を直接書く。既知アプリは固有キー（Assets: `shiden_asset_gh_cfg`、Dashboard: `mtworks_gh_cfg`）へ、全アプリ共通で `shiden_hub_connection` へ書く
2. **接続して開く** — `https://.../Shiden-XXX/#shiden-connect=<base64url JSON>` で開く。フラグメントはサーバーに送られず、受け取ったアプリは取り込み後すぐ URL から消す。iOS ホーム画面アプリ（ストレージが分離）はこの方法か、トークンをコピーして貼り付ける
3. **トークンをコピー** — 従来どおりアプリの設定に貼り付け

### アプリ側の対応（Shiden Assets / Dashboard は対応済）

起動時に自分の設定が無ければ `shiden_hub_connection` を取り込み、`#shiden-connect=` があれば明示操作として上書きする。各アプリの GitHub 設定コードの末尾に以下を足し、`render()` の前で `adoptHubConnection()` を呼ぶ（`getCfg / setCfg / cfgOk / b64dec` は各アプリ既存のもの）。

```js
function adoptHubConnection() {
  const HUB_KEY = "shiden_hub_connection";
  const m = location.hash.match(/[#&]shiden-connect=([A-Za-z0-9_-]+)/);
  let hub = null;
  if (m) {
    try { hub = JSON.parse(b64dec(m[1].replace(/-/g, "+").replace(/_/g, "/"))); } catch (e) {}
    history.replaceState(null, "", location.pathname + location.search);
    if (hub && hub.token) { try { localStorage.setItem(HUB_KEY, JSON.stringify(hub)); } catch (e) {} }
  }
  if (!hub) { try { hub = JSON.parse(localStorage.getItem(HUB_KEY)); } catch (e) {} }
  if (!hub || !hub.token || !hub.owner || !hub.repo) return false;
  const c = getCfg();
  if (cfgOk(c) && !m) return false; // 手動設定は上書きしない（リンク経由は明示操作なので更新）
  setCfg(Object.assign({}, c, { owner: hub.owner, repo: hub.repo, branch: hub.branch || "main", token: hub.token, path: c.path || "<そのアプリの既定path>" }));
  return true;
}
```

対応したアプリは、このアプリの「紫電アプリ接続 → 編集」で設定キーと既定 path を登録すると「この端末に書き込む」が直接効くようになる。

`shiden_hub_connection` の形:

```json
{ "v": 1, "owner": "hayahide-cloud", "repo": "Hayahide-Log", "branch": "main", "token": "github_pat_...", "updated": "ISO", "source": "shiden-vault" }
```

## 初期設定

1. GitHub で Fine-grained token を発行（対象: Hayahide-Log のみ・Contents 読み書き）
2. 右上「設定」に owner / repo / path（`vault.json`）/ token を入力して保存
3. 「保管庫を作成」でマスターパスワードを決める（12文字以上推奨）
4. 「紫電アプリ接続 → このアプリの接続トークンを保管庫に登録」で、そのトークン自体も保管庫に記録（期限は GitHub の設定画面で確認して記入）
5. 「この端末の全アプリに書き込む」で Assets / Dashboard へ配布

## ファイル形式（vault.json）

```json
{ "format": "shiden-vault", "v": 1,
  "kdf": { "name": "PBKDF2", "hash": "SHA-256", "iter": 600000, "salt": "<base64>" },
  "iv": "<base64>", "ct": "<base64>", "updated": "<ISO>" }
```

復号後: `{ v, entries:[{ id, type:"login"|"token"|"card"|"note", name, …, ct, mt }], apps:[{ id, name, url, storageKey, path }], hub:{ tokenId, owner, repo, branch } }`
