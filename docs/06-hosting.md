# 06. ホスティングの選定

調査日：2026-11-17。

価格は比較サイト・解説記事からの情報で、各社の公式ページでは確認できていない。記事どうしで食い違う価格もある。契約前に公式の料金表で確かめる。ドル建ての価格は為替で変わる。

## 1. 求める条件

| # | 条件 | 理由 |
|---|---|---|
| H1 | 常時起動している | LINE の Webhook にすぐ応答する必要がある。朝 9:30・10:00・正午の定時処理と、暗証番号の発行が、深夜でも動く必要がある |
| H2 | 消えないディスクがある | DB（SQLite）と写真をサーバーのディスクに置く設計のため |
| H3 | メモリ 1GB 以上 | Rails・ジョブ・SQLite を 1 台で動かす。512MB では動作が重くなったという報告がある |
| H4 | 日本国内（東京・大阪） | 利用者も LINE も SwitchBot の利用も国内。画面と解錠の応答を速くする |
| H5 | 月額 1,000 円程度以内 | 要件 N-10 |
| H6 | 最低利用期間が短い | 3 か月の試行で、やめる・乗り換える可能性がある |
| H7 | 運用の手間が小さい | 運用するのはホスト 1 人 |

## 2. 候補の比較

| 方式 | サービス | 月額の目安（1GB） | H1 常時起動 | H2 ディスク | H4 国内 | 評価 |
|---|---|---|---|---|---|---|
| 国内 VPS | KAGOYA CLOUD VPS | 日額 20 円、月の上限 550 円 | ○ | ○ | ○ | **第一候補** |
| 国内 VPS | さくらの VPS | 880 円 | ○ | ○ | ○ | 代替 |
| 国内 VPS | ConoHa VPS | 468〜763 円（記事により差） | ○ | ○ | ○ | 代替 |
| 国内 VPS | シン VPS | 451 円（36 か月契約）〜750 円 | ○ | ○ | ○ | 最低利用期間 3 か月 |
| 国内 VPS | WebARENA Indigo | 上限 449 円 | ○ | ○ | ○ | 新規受付を停止中との記載あり |
| 海外系 VPS | Vultr、Akamai（Linode） | 5 ドル | ○ | ○ | ○（東京・大阪） | ドル建て。価格の出所が弱い |
| PaaS | Fly.io | 約 6.9 ドル（1GB ＋ ディスク 1GB） | ○ | ○（ボリューム） | ○（東京） | **第二候補** |
| PaaS | Render | 7 ドル前後 ＋ ディスク 0.25 ドル/GB | ○（有料プラン） | ○ | ×（最寄りはシンガポール） | 不採用 |
| PaaS | Railway | 5 ドル〜 ＋ 従量 | ○ | ○ | 未確認 | 不採用 |
| 無料枠 | Oracle Cloud Always Free | 0 円 | △ | ○ | ○ | 不採用 |
| サーバーレス | Cloud Run など | 従量 | × | × | ○ | 不採用 |

不採用の理由：

- **Render・Railway**：国内リージョンがない、または確認できない。Render はディスクを付けると無停止での入れ替えができない。費用面の利点もない。
- **Oracle Cloud Always Free**：使用率が低いインスタンスは回収の対象になる。空きがなく作成できないこともある。夜間に鍵を開ける仕組みの土台にするには不確実。
- **サーバーレス**：アクセスがないと停止する方式で、定時処理と消えないディスクの前提に合わない。DB を外部に出す設計変更が要る。

## 3. 推奨

### 第一候補：国内 VPS（1GB）＋ Kamal

設計（03）の前提どおりで、変更が要らない。円建てで月 1,000 円以内に収まる。

その中では **KAGOYA CLOUD VPS** を第一に挙げる。日額課金で最低利用期間がなく、月の上限が 550 円と安い。試行をやめる・乗り換える場合の損が小さい。さくらの VPS・ConoHa VPS でも条件は満たすので、使い慣れたものがあればそちらでよい。

VPS に付いてくる手間：

- OS の更新、SSH の鍵認証、ファイアウォール（22・80・443 のみ開ける）の設定を自分で行う
- 独自ドメインが要る。LINE 内ブラウザと Webhook は HTTPS が必須で、Kamal Proxy が証明書を自動取得するにはドメインが必要

### 第二候補：Fly.io（東京リージョン）

サーバーの OS を管理したくない場合の選択肢。

- 良い点：OS の管理が不要。HTTPS つきのサブドメインが付くので独自ドメインが要らない。異常終了時に自動で再起動する
- 悪い点：ドル建てで、1GB の構成は月 1,000 円をやや超える見込み。2026 年 10 月にメモリの単価が値上げされた。配置の方法が Kamal ではなく Fly.io 独自になる。東京リージョンの単価は確認できていない
- ディスク（ボリューム）は 1 台のマシンに紐づくので、冗長化はされない。バックアップは VPS と同じく必要

### 判断の目安

| 状況 | 選ぶもの |
|---|---|
| Linux サーバーの基本的な設定ができる・費用を抑えたい | 国内 VPS |
| サーバー管理を避けたい・多少高くてもよい | Fly.io |

どちらを選んでも、アプリは Docker コンテナ 1 つなので、後から乗り換えられる。移すのは SQLite のファイルと写真だけ。

## 4. 周辺のサービス

| 用途 | 候補 | 費用の目安 | 備考 |
|---|---|---|---|
| バックアップ先 | Cloudflare R2 などのオブジェクトストレージ | ほぼ 0 円（データは数十 MB の見込み） | R2 は保存 0.015 ドル/GB・月、転送料なしとの記載。無料枠の有無は未確認 |
| 独自ドメイン（VPS の場合） | 任意のレジストラ | 年 1,000〜2,000 円程度（未確認） | Fly.io なら不要 |
| 通知 | LINE Messaging API | 無料枠の範囲を想定 | 月間の送信数上限を契約前に確認（03 の 2 章） |
| 死活監視 | 外部の監視サービス（無料枠のあるもの） | 0 円 | サーバーが止まったらホストに知らせる。止まると暗証番号を発行できないため、必ず入れる |

バックアップは 1 日 1 回、SQLite の複製と写真をオブジェクトストレージに送る。常時複製する方式（Litestream）は、メモリ 512MB の環境で動作が重くなったという報告があるので、まずは 1 日 1 回の方式にする。

## 5. 月額の見込み

| 項目 | 国内 VPS の場合 | Fly.io の場合 |
|---|---|---|
| サーバー | 550〜880 円 | 約 6.9 ドル |
| 独自ドメイン | 月あたり 100〜170 円程度 | 不要 |
| バックアップ先 | ほぼ 0 円 | ほぼ 0 円 |
| LINE・監視 | 0 円（無料枠） | 0 円（無料枠） |
| 合計 | 約 650〜1,050 円 | 約 6.9 ドル |

鍵の機器代（04）は含まない。

## 6. 契約前に確かめること

1. 選んだ VPS の、1GB プランの現在の月額・初期費用・最低利用期間
2. 国内 VPS で Docker と Kamal が動くこと（root での SSH 接続、IPv4 アドレスの付与）
3. Fly.io を選ぶ場合、東京リージョンの単価と、ボリュームつきマシンの月額
4. LINE Messaging API の無料枠の送信数上限
5. バックアップ先の無料枠と、復元の手順（実際に 1 度復元してみる）

## 7. 情報源

- 国内 VPS の料金比較：https://ryusei.io/en/article/vps-compare-prices ／ https://www.gr-co.jp/media/vps-osusume/ ／ https://i-fc.jp/column/vps-rentalserver/ ／ https://streamrental.com/linuxvps_hikaku/
- Fly.io の料金：https://fly.io/pricing/ ／ https://hostim.dev/blog/fly-io-pricing-2026/ ／ https://www.buildmvpfast.com/blog/fly-io-pricing-2026-ram-increase-vs-hetzner
- 東京リージョンの海外系 VPS：https://www.valebyte.com/en/blog/best-vps-in-tokyo-2026-review-and-prices/
- Render・Railway：https://temps.sh/blog/render-pricing-2026 ／ https://techsy.io/en/blog/railway-vs-render-vs-fly-io ／ https://selfhost.dev/railway-vs-render-pricing-2026-verdict/
- Oracle Cloud Always Free：https://docs.oracle.com/en-us/iaas/Content/FreeTier/resourceref.htm ／ https://oracommit.blogspot.com/2026/02/understanding-oci-always-free-compute.html
- Rails 8 ＋ SQLite の運用：https://scrapbox.io/miurahantorb/Rails8.1%E3%81%A8SQLite3%E3%81%A8Solid%E4%B8%89%E5%85%84%E5%BC%9F%E3%81%A8Litestream ／ https://akitaonrails.com/en/2026/02/20/sqlite-kamal-rails-deploy-without-drama-behind-the-m-akita-chronicles/
