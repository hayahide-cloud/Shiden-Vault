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

## 接続コード（合言葉で自動接続）

各アプリの設定に **合言葉を1回入れるだけ** で Hayahide-Log に接続できる仕組み。トークンの貼り付けは不要で、iPhone のホーム画面アプリ（保存領域がアプリごとに分かれる）でも使える。

1. Vault の「紫電アプリ接続 → 接続コード（合言葉）」で合言葉を決めて（「生成」推奨）発行する
2. Vault が接続情報（owner / repo / branch / token）を合言葉から導出した鍵（PBKDF2-HMAC-SHA256 600,000回 → AES-256-GCM）で暗号化し、このリポジトリの `connect.json` に書き込む（Pages で公開されるが中身は暗号文のみ）
3. 各アプリの設定欄「接続コード（合言葉）」に入力 → `https://hayahide-cloud.github.io/Shiden-Vault/connect.json` を取得・復号して設定に保存
4. 合言葉は同一オリジン共有キー `shiden_connect_code` に保存されるので、同じブラウザの他の紫電アプリは入力なしで自動接続する
5. **トークン更新**: Vault で配布トークンを切り替えて「配布設定を保存」すると同じ合言葉で自動再発行される。各アプリは同期が 401/403 で失敗した時点で合言葉から取り直し、新しいトークンに自動で切り替わる
6. **無効化**: Vault の「無効化」で `connect.json` を失効状態にする（接続済みのアプリはそのまま使える。トークン自体を止めたい場合は GitHub でトークンを revoke する）

注意:
- 合言葉は大文字小文字・空白・ハイフンを区別しない（NFKC 正規化して大文字化）。英数12文字以上のランダムなもの（「生成」= 約60bit）を使う。規則的な語は総当たりされうる
- **トークンは2本に分ける**（対象リポジトリだけが違い、どちらも Contents: Read and write）

  | トークン | 対象リポジトリ | 置き場所 | 用途 |
  | --- | --- | --- | --- |
  | 接続用 | Hayahide-Log のみ | Vault の右上「設定」・各アプリ（配布） | 保管庫と各アプリのデータの読み書き |
  | 発行用 | Shiden-Vault のみ | 保管庫の中（「トークン・APIキー」として登録） | connect.json の発行・再発行・無効化 |

  発行用トークンは「紫電アプリ接続 → 発行用トークン」で選ぶ。暗号化された保管庫の中にだけ置かれ、発行・無効化の時だけメモリ上で使われる。端末の設定にも各アプリにも渡らないため、配布したトークンや合言葉が漏れても connect.json は書き換えられない
- 発行用トークンを選ばない場合は設定のトークンで書き込む（旧方式。そのトークンに Shiden-Vault の権限が必要）

### アプリ側の対応状況

| アプリ | 合言葉入力 | 共有キー自動接続 | 401 自動復旧 |
| --- | --- | --- | --- |
| Shiden Assets | ✅ | ✅ | ✅ |
| Shiden Dashboard | ✅ | ✅ | ✅ |
| Shiden Vault（2台目以降） | ✅ | ✅ | — |
| Shiden LifeLogger | ✅ | ✅ | ✅ |
| Shiden Tasks | ✅ | ✅ | ✅ |
| Shiden Expense | ✅ | ✅ | ✅（GitHub API の全リクエストで自動再試行） |

新しいアプリに組み込むときの参照実装は Shiden-Assets の `index.html` の「紫電 接続コード」ブロック（`fetchConnect` / `applyConnectCode` / `recoverByCode`、約45行）。設定パネルに入力欄を1つ足し、同期の 401/403 で `recoverByCode()` を呼んで true なら1回だけ再試行する。

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

1. GitHub で Fine-grained token を2本発行（どちらも Contents 読み書き）: 接続用 = Hayahide-Log のみ、発行用 = Shiden-Vault のみ
2. 右上「設定」に owner / repo / path（`vault.json`）/ 接続用トークンを入力して保存
3. 「保管庫を作成」でマスターパスワードを決める（12文字以上推奨）
4. 「紫電アプリ接続 → このアプリの接続トークンを保管庫に登録」で、そのトークン自体も保管庫に記録（期限は GitHub の設定画面で確認して記入）
5. 発行用トークンを「＋ 追加 → トークン・APIキー」で保管庫に登録し、「紫電アプリ接続 → 発行用トークン」で選ぶ
6. 「接続コード（合言葉）」を発行する
7. 他の端末・アプリでは設定の「接続コード」に合言葉を入れるだけ。同じブラウザなら「この端末の全アプリに書き込む」でも可

## ファイル形式（vault.json）

```json
{ "format": "shiden-vault", "v": 1,
  "kdf": { "name": "PBKDF2", "hash": "SHA-256", "iter": 600000, "salt": "<base64>" },
  "iv": "<base64>", "ct": "<base64>", "updated": "<ISO>" }
```

復号後: `{ v, entries:[{ id, type:"login"|"token"|"card"|"note", name, …, ct, mt }], apps:[{ id, name, url, storageKey, path }], hub:{ tokenId, owner, repo, branch } }`
