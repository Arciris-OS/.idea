# ArcirisOS Init検証 / Update

## 1. Init システム署名と検証

### 署名フロー

* 作成した秘密鍵を用いて、initramfsとカーネルを署名
* initramfsはGPGでdetached signature作成

  * 署名により、改ざんされたinitramfsは起動前に検知可能
* カーネルはSecure Boot用にsbsigで署名

  * GRUBやshimからSecure Boot検証チェーンで確認される

### GRUB での検証

* GRUBに公開鍵を埋め込み、起動時にinitramfsの署名検証を行う
* 検証に成功した場合のみブートを続行
* 失敗した場合はブートを停止し、ログやコンソールに警告を出す
* これにより、BIOS/UEFIが改ざんされていなくても、initramfsの改ざんは阻止可能

### セキュリティ補足

* initramfs の署名を検証することで、rootfsマウント前に改ざんを検知できる
* 秘密鍵は誰かが保有し配布サーバーには公開鍵のみ配置

## 2. noaによるチェック

### 更新チェックのタイミング

* noa起動後、ネットワーク準備完了直後に自動でチェック
* 最新のmanifest.jsonを公式サーバーから取得
* manifest.jsonにはOSコアのバージョン、署名情報、必要ファイルリストを含む。↓の感じ

```json
{
  "version": "0.1.1",
  "release_date": "2025/09/02",
  "description": "Linux kernel update (X.X.X -> Y.Y.Y)",
  "signature_algorithm": "gpg",
  "files": {
    "kernel": {
      "name": "vmlinux-0.1.2.signed",
      "hash": "sha256:abcd1234...5678",
      "signed": true
    },
    "initramfs": {
      "name": "initramfs-0.1.2.img",
      "signature": "initramfs-0.1.2.img.asc",
      "hash": "sha256:1234abcd...5678",
      "signed": true
    }
  },
  "packages": [
    {
      "name": "coreutils",
      "version": "9.5",
      "hash": "sha256:efgh5678...abcd",
      "signed": true
    },
    {
      "name": "noa",
      "version": "256",
      "hash": "sha256:ijkl9012...mnop",
      "signed": true
    }
  ],
  "slots": {
    "active": "A",
    "inactive": "B"
  },
  "mandatory_update": false,
  "changelog": [
    "Fixed kernel panic on rare hardware",
    "Updated initramfs with new security modules",
    "Minor system library updates"
  ]
}
```
* `version` : OSバージョン番号
* `release_date` : 更新リリースの日付
* `description` : 更新内容の簡単な説明
* `signature_algorithm` : 使用している署名方式（例：GPG）
* `files` : OSコア必須ファイルの情報

  * `kernel` : カーネルファイルの名前、ハッシュ、署名有無
  * `initramfs` : initramfs ファイルの名前、署名ファイル名、ハッシュ、署名有無
* `packages` : 更新対象パッケージの情報（名前、バージョン、ハッシュ、署名有無）
* `slots` : A/B スロットの情報

  * `active` : 現在使用中スロット
  * `inactive` : 更新対象スロット
* `mandatory_update` : 強制的に更新するかどうか（重大な脆弱性の修正とかはこっち？）
* `changelog` : 更新内容の履歴や変更点の詳細な説明


### CLI でのユーザー確認

* CLI 版では以下の流れでユーザーに通知

  * 「There is a new update. Would you like to update now? (y/n)」と表示
  * y → `iris upgrade` 実行
  * n → 通常起動

### GraphicalEdition(仮) の通知
* Gnome の通知機能を利用
* 更新がある場合、通知を表示してユーザーにクリックさせる

  * 例: 「A new update is available. Click here to update」
* 通知クリックでウィンドウを表示し、アップデートするを選択されたら `iris upgrade` を実行
* 更新進行中も通知で `% 完了` を表示可能（irisに作る必要あり）

### エラー処理

* ダウンロード失敗 → 再試行や次回更新で対応
* 署名検証失敗 → 更新を適用せず警告を表示
* 更新適用中の障害 → A/B スロット方式でロールバック

## 3. irisUpdate設計

### 配布サーバー構成

* サーバーには署名付きファイルを置く

  * vmlinux-<version>.signed
  * initramfs-<version>.img + .asc
  * manifest.json（署名付き）

### A/B スロット方式

* /boot/slotA と /boot/slotB にそれぞれ別バージョンを保持
* 更新は非アクティブスロットに展開
* GRUB 設定を書き換え、次回起動で新スロットを使用
* 起動成功 → 新スロットをメインに確定
* 起動失敗 → 自動で旧スロットにロールバック

### セキュリティ設計

* manifest.jsonはGPG署名検証必須
* initramfsは必ず署名確認後に適用
* カーネルはSecure Boot用に署名済み
* 失敗時には自動ロールバック
* 更新チェックと署名検証はnoaが制御、ユーザーが勝手に変更できない

## 4. フロー

1. noa起動 → ネットワーク準備完了
2. 更新チェック (manifest.json 取得 & 署名検証)
3. CLI / GUIに通知
4. ユーザー同意後、または自動モードで`iris upgrade`実行
5. 更新ファイルを非アクティブスロットに展開
6. GRUB設定を書き換え、再起動
7. 起動成功 → 新スロット確定、失敗 → ロールバック

### 追加ポイント

* CLIはユーザーに y/n で確認
* GUIはクリック操作
* 更新進行状況を可視化
* 安全性を最優先に設計しつつ、ユーザー体験は自然に維持
