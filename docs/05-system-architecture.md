# 05. システム構成図

ステージ0（自宅 1 か所・SwitchBot 前提）の構成。設計の詳細は [03-system-design.md](03-system-design.md)、鍵の選定は [04-lock-selection.md](04-lock-selection.md)。

## 1. 全体構成

```mermaid
flowchart TB
  subgraph users["利用者のスマホ"]
    guest["ゲスト<br>LINE アプリ"]
    host["ホスト<br>LINE アプリ"]
  end

  subgraph line["LINE プラットフォーム"]
    liff["LIFF<br>LINE 内ブラウザ"]
    login["LINE Login<br>ID トークンの検証"]
    msg["Messaging API<br>公式アカウント"]
  end

  subgraph vps["VPS 1 台（Kamal で配置）"]
    proxy["Kamal Proxy<br>HTTPS 終端"]
    subgraph app["Rails 8 アプリ（1 コンテナ）"]
      web["Web<br>画面・Webhook 受付"]
      jobs["Solid Queue<br>通知・定時処理"]
      lockad["鍵アダプター"]
    end
    db[("SQLite<br>利用・同意・監査ログ")]
    files[("ディスク<br>ガイドと記録の写真")]
  end

  backup[("バックアップ先<br>オブジェクトストレージ")]

  subgraph sb["SwitchBot"]
    sbapi["SwitchBot クラウド API"]
  end

  subgraph home["ホスト宅"]
    wifi["Wi-Fi ルーター"]
    hub["SwitchBot ハブ"]
    lock["SwitchBot ロック<br>玄関の内側"]
    keypad["SwitchBot キーパッド<br>玄関の外側"]
  end

  guest --> liff
  host --> liff
  liff -->|HTTPS| proxy
  proxy --> web
  web -->|ID トークン検証| login
  web --> db
  web --> files
  jobs --> db
  jobs -->|プッシュ通知| msg
  msg -->|通知| guest
  msg -->|通知と承認ボタン| host
  msg -->|Webhook| proxy
  web --> lockad
  jobs --> lockad
  lockad -->|番号の発行・解錠・施錠・状態取得| sbapi
  sbapi -->|インターネット| wifi
  wifi --> hub
  hub -->|Bluetooth| lock
  hub -->|Bluetooth| keypad
  keypad -->|Bluetooth| lock
  guest -.->|暗証番号を入力| keypad
  db -->|1 日 1 回| backup
  files -->|1 日 1 回| backup
```

## 2. 構成要素

| 区分 | 要素 | 役割 |
|---|---|---|
| 利用者 | ゲスト・ホストのスマホ | LINE アプリだけで使う。専用アプリは入れない |
| LINE | LIFF | LINE の中で Web 画面を開く。ログイン操作が要らない |
| LINE | LINE Login | 利用者の特定（LINE のユーザー識別子） |
| LINE | Messaging API | 通知の送信、承認・解錠ボタンの操作の受け取り（Webhook） |
| サーバー | Kamal Proxy | HTTPS の終端、無停止での入れ替え |
| サーバー | Rails アプリ（Web） | 画面、申込・承認・自動承認の判定、ガイドの編集、権限の確認 |
| サーバー | Solid Queue | 通知の送信、朝 9:30・10:00・正午の定時処理、番号の発行と確認 |
| サーバー | 鍵アダプター | 鍵の操作を 1 か所にまとめる部品。機種を替えるときはここだけ差し替える |
| サーバー | SQLite | 利用者、開放、利用、ガイドの版と同意、監査ログ |
| サーバー | ディスク | ガイドとトラブル記録の写真 |
| 外部 | バックアップ先 | DB と写真の複製を 1 日 1 回保管 |
| 鍵 | SwitchBot クラウド API | 番号の発行、解錠・施錠、状態取得の窓口 |
| ホスト宅 | Wi-Fi ルーター・ハブ | クラウドからの命令を鍵とキーパッドに中継 |
| ホスト宅 | ロック・キーパッド | 玄関の施錠・解錠、暗証番号の入力 |

## 3. 主な流れ

### 申込から入室まで

```mermaid
sequenceDiagram
  actor G as ゲスト
  participant A as Rails アプリ
  participant L as LINE Messaging API
  actor H as ホスト
  participant S as SwitchBot API
  participant K as ハブとキーパッドとロック

  G->>A: 申込（到着予定、希望、代行の見込み）
  alt 信頼済みで条件を満たす
    A->>A: 自動承認
    A->>L: 自動承認の通知（音なし）
    L->>H: 通知
  else それ以外
    A->>L: 申込の通知
    L->>H: 通知（承認と辞退のボタン）
    H->>L: 承認
    L->>A: Webhook
  end
  A->>L: 承認の通知（喫煙ルール、案内へのリンク）
  L->>G: 通知
  G->>A: 「着きました」
  A->>S: 30 分有効の暗証番号を発行
  S->>K: 番号を登録
  A->>G: 暗証番号を表示
  G->>K: 暗証番号を入力して解錠
  G->>A: 退出
  A->>L: 退出の通知
  L->>H: 通知
```

### ホストによる取消・退去要請

```mermaid
sequenceDiagram
  actor H as ホスト
  participant A as Rails アプリ
  participant S as SwitchBot API
  participant L as LINE Messaging API
  actor G as ゲスト

  H->>A: 承認取消、退去要請、利用停止
  A->>A: 利用の状態を変更し、監査ログに記録
  A->>A: 以後の暗証番号の発行を止める
  A->>S: 発行済みの番号を削除
  A->>S: 番号一覧を読み直して確認
  opt 番号が残っている
    A->>L: ホストに通知（最大 30 分で失効）
  end
  A->>L: ゲストに通知
  L->>G: 利用が終了したこと
  G->>A: 画面を開く
  A->>G: 「利用終了」画面（住所と番号は非表示）
```

## 4. 障害時の動き

| 止まったもの | 影響 | 代替 |
|---|---|---|
| サーバー（VPS） | 申込・承認・番号の発行ができない | LINE で直接やり取りし、ホストが SwitchBot アプリで解錠する |
| LINE | 画面を開けず、通知も届かない | 電話などで連絡し、ホストが SwitchBot アプリで解錠する |
| SwitchBot クラウド・ハブ・宅内 Wi-Fi | 新しい番号の発行と遠隔の解錠ができない | ホストが玄関で直接開ける。発行済みの番号が使えるかは実機で確認する（04 の 4 章） |
| キーパッド・ロックの電池切れ | 暗証番号でも遠隔でも開かない | 物理の鍵で開ける。電池残量を定期的に確認する |
