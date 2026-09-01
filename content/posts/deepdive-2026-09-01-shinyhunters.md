---
title: "ShinyHunters — 「脆弱性」ではなく「ログイン」を突く、2026年最大級の恐喝マシン"
date: 2026-09-01T07:00:00+09:00
tags: ["security", "intelligence", "深掘り", "ShinyHunters"]
draft: false
---

## 30秒サマリ

- 2026年4月〜9月、ShinyHunters（Google/Mandiantの呼称 UNC6240）は暗号化を伴わない「pay or leak（払うか、公開されるか）」型の恐喝を、教育・医療・小売・製造・国際機関にまたがって連続実行した。
- 侵入経路は大きく3系統。①ビッシング／ヘルプデスクなりすましによるSSO（Okta・Entra・Google）乗っ取り、②サードパーティSaaS・ベンダー（Anodot、Context.ai、ShipMonk等）経由の認証トークン窃取、③Oracle PeopleSoftゼロデイ CVE-2026-35273（CVSS 9.8）の大量悪用。
- 標的の中心は「大量のPIIが集まるデータ層」——Salesforce、Snowflake、BigQuery、Databricks。マルウェアではなく正規の認証情報でログインして持ち出すため、境界防御・EDRでは検知しづらい。
- 公表ベースの被害規模は、Instructure/Canvas 2億7,500万ユーザー・約9,000教育機関、DentaQuest 最大2,340万人、Carhartt 1,290万アカウント、McGraw Hill 1,350万アカウント、Medtronic 380万人など。9月1日時点でMcKesson（2.84億件主張・5,500万ドル要求）が期限を迎えている。
- 恐喝は「一発勝負」ではない。Instructureのように交渉合意に至る例、Charter・7-Eleven・Carharttのように支払い拒否で公開される例、さらに公開データが数ヶ月後に第三者のセクストーション詐欺に再利用される例まで、被害は時間軸に沿って伸び続ける。

## 背景 — 何者か / 何が起きているか

ShinyHuntersは、記事データ上で「恐喝グループ」「データ窃取グループ」と一貫して記述される脅威アクターです。Googleは本キャンペーンに関連してUNC6240の指定を用いており（2026年6月のPeopleSoft報道）、MandiantおよびGoogle Threat Intelligence Groupが追跡しています。

重要なのは、彼らが**ランサムウェア（暗号化）を主武器としていない**点です。Health-ISACが2026年7月25日に出した警告は、この構造を端的にまとめています——ランサムウェアではなく、ビッシング（音声ソーシャルエンジニアリング）で従業員やヘルプデスクを騙し、MFAのリセットやデバイス再登録を実施させ、Microsoft Entra・Okta・Google SSOのアカウントを乗っ取り、SaaSプラットフォームへ横展開してデータを窃取する。暗号化がないため業務は止まらず、結果として**検知が遅れ、公表も遅れる**。

SecurityWeekが2026年6月23日に分析した通り、これは「エクスプロイトやマルウェアからIDベースの攻撃への構造的シフト」です。窃取クレデンシャル、OAuthトークン悪用、MFA疲弊攻撃、ビッシング、ヘルプデスクなりすまし——いずれも「ログインするだけ」の攻撃であり、従来の境界防御の想定外にあります。

もう一つの文脈として、Grafanaの侵害報道（5月19日）ではShinyHunters・Scattered Spider・Lapsus$に関連する「Coinbase Cartel」という括りが登場します。7月3日にはScattered Spiderメンバーとされる19歳が米国へ送還・起訴されており、これらのグループが緩やかなネットワークとして相互に人材・手口を共有している構図がうかがえます。ただし各グループ間の正確な関係は、報道時点では確定的に整理されていません。

## 時系列 — インシデントの連鎖

### 第1波（4月）：サプライチェーンとSalesforce設定不備

**4月9日 — 供給網からの燃料補給。** SANS ISCが報じたTeamPCP／UNC6780キャンペーンでは、Trivyのサプライチェーン侵害（CVE-2026-33634）と悪意あるGitHub Actionを介してCiscoの開発環境が侵害され、300以上のプライベートリポジトリとAWSキーが窃取されました。注目すべきは末尾の一文——**ShinyHuntersがその盗取データを利用してFBI・DHS等を標的に恐喝活動を展開中**。つまりShinyHuntersは自ら侵入するだけでなく、他アクターが得たデータの「恐喝フロントエンド」としても機能しています。

**4月14〜15日 — 二正面の同時展開。** Rockstar Gamesがアナリティクスベンダー**Anodot**経由の漏洩を確認（ベンダー起点）。同日翌日、McGraw-Hillが**Salesforceのホスト型ウェブページ設定ミス**を突かれたと確認（設定不備起点）。ここで後の全キャンペーンを貫く2本の柱が出揃います。

**4月17日 — 数字が化ける。** McGraw Hillの被害は当初「限定的で非機密」との同社説明から、**1,350万アカウント・100GB超**へ拡大。攻撃者の主張（4,500万件）と被害企業の初期説明の乖離は、以降ほぼ全事案で繰り返されるパターンです。

**4月20〜26日 — ビッシング×Okta SSOの完成。** ADTで4月20日に不正アクセスを検知。攻撃ベクターは**ビッシングによる従業員Okta SSOアカウント乗っ取り → ADTのSalesforceインスタンスへのアクセス**。ShinyHuntersは1,000万件を主張しましたが、4月28日の確認値は約550万人・11GB。ADTにとっては2024年に続く再侵害です。

**4月21日 — トークン窃取の教科書的事例。** VercelがContext.ai経由の侵害を認める。社員の**個人デバイス**がRobloxチートに偽装したLumma Stealerに感染 → OAuthトークン窃取 → Vercel内部のAIツールへ不正アクセス。ShinyHuntersを名乗る攻撃者が200万ドルでの売却を試みました。BYOD、インフォスティーラー、過剰権限OAuth統合が一直線につながった構図です。

**4月28日 — 医療セクターへ。** Medtronicが900万件主張を受けて企業ITシステムの侵害を確認（実際の侵入期間は4月13〜19日と後に判明）。ADTと同一の「ビッシング＋SSO認証情報窃取」手口であることが報じられ、**単発事件ではなくキャンペーンである**ことが明確になりました。

**4月29日 — Anodotの再登場。** VimeoがAnodotの認証情報窃取に起因し、**SnowflakeとBigQuery**インスタンスへ不正アクセスされたと確認。Rockstar（4月14日）と同じベンダーです。1社のアナリティクスベンダー侵害が複数顧客に波及する典型例。

### 第2波（5月）：教育セクターの大量恐喝

**5月5〜9日 — 過去最大級。** InstructureがCanvas LMSの侵害を公表。ShinyHuntersは**3.65TB・9,000以上の教育機関・2億7,500万人**を主張し、HarvardやMITを含むとしました。CMCの後日分析（6月27日）によれば、不正活動の検出は**4月29日**。

**5月6日 — 点と点がつながる。** Vimeoの被害が11万9,200人と確定。同日のTechCrunch報道は、InstructureとVimeoが**同一グループ・類似手口**であると明示しました。

**5月7日 — 二つ目の脆弱性。** CMCの記録では、5月7日に別の脆弱性を悪用して**約330サイトのログインページが改ざん**されています。

**5月12日 — 恐喝の「小売化」。** ShinyHuntersは数百校のログインページを改ざんし、**教育機関ごとに個別の身代金交渉**を迫るキャンペーンに移行。同日、ZaraでもAnodot経由・BigQuery/Snowflake侵害による約19万7,000人の漏洩が判明。Infosecurity Magazineはこれを「pay or leak」キャンペーンの一環と位置づけました。期末試験期間を狙ったタイミングは意図的と見られています。

**5月13〜14日 — 交渉の成立。** InstructureがShinyHuntersと**データ削除の合意**に達したと報道。同社は各教育機関に対し「攻撃者との直接交渉は不要」と通知。合意条件は非公開で、削除の実効性は保証されません。この「合意」は業界に強い議論を呼びました。

**5月19〜30日 — 小売・通信・レジャーへの拡大。** 7-Eleven（4月8日検知、Salesforceから9.4GB／約60万レコード、25万ドル要求→拒否→リーク、最終的に約18万5,300人）、Grafana、Carnival（4月14日判明、ソーシャルエンジニアリングで従業員アカウント侵害、約600万人、後に870万レコード公開、政府発行ID番号を含む）、Charter Communications（4,200万レコードのリーク、HIBP分析で490万件のユニークメール＋8万5,000件の従業員レコード、支払い拒否と見られる）。

### 第3波（6月）：ゼロデイという新兵器

**5月27日〜6月9日 — 静かな大量侵害。** ShinyHuntersがOracle PeopleSoft PeopleToolsの**未パッチゼロデイ CVE-2026-35273**（CVSS 9.8、PeopleTools 8.61/8.62、認証不要・ユーザー操作不要のRCE）を悪用し、**300以上のPeopleSoftインスタンス**を侵害。これはIDベース攻撃一辺倒だったこのグループが、初めて大規模な技術的エクスプロイトに手を伸ばした瞬間です。

**6月11〜13日 — 表面化。** Mandiantが**100以上の組織**に通知、うち**68%が米国の高等教育機関**。ノッティンガム大学では45万5,000件の固有メールアドレスと40GB超の文書が公開され、パスポート番号・民族・障害情報・成績・支払い記録まで含まれていました。6月12日にCISAがKEVカタログへ追加、**是正期限は6月15日——実質3日間**。この時点でOracleからパッチは提供されていませんでした。

**6月16日 — 国際機関へ。** 欧州評議会（46加盟国・7億人以上を代表）が調査を開始。ShinyHuntersは42万9,000件超の文書（2011〜2026年の給与明細40万9,000件、職員1万人以上、人事ファイル3,700件、履歴書1万4,000件、銀行口座・社会保険番号・医療記録を含む）を主張し、6月16日の公開を予告。同日、Infinite Campus（全米K-12学区向け学生情報システム）でも3月のSalesforce窃取により13万7,000件の教職員アカウントが流出したと判明。

**6月18〜19日 — Kodak。** 220万件超（顧客PII＋内部企業データ）の窃取主張と6月18日の公開予告。

**6月30日〜7月1日 — PeopleSoft波の実害。** Nissanが米国・カナダ・メキシコ・ブラジルの現職／元従業員の**社会保障番号・銀行口座詳細・税務情報・扶養家族情報**の流出を公表。Oracleが緊急アドバイザリを出したのは攻撃開始後でした。同日、NAIC（全米保険監督官協会）も6月11日に侵害を検出したと発表——ただしNAICは「公開情報・古いログ・設定ファイルのみ」と主張し、ShinyHuntersは3.1TB・10万5,000ファイル（保険会社提出書類やAWS設定を含む）と主張。**主張の食い違いが未解消のまま**です。

### 第4波（7月〜9月）：医療セクターへの収斂と、被害の二次利用

**7月3〜4日 — Medtronic確定値。** 380万人以上。氏名・連絡先・生年月日・社会保障番号・健康関連情報。当初主張の900万件から縮小した一方、**ShinyHuntersはリークサイトからMedtronicを削除**しており、身代金支払いの可能性が示唆されています（未確認）。

**7月25日 — 業界警告。** Health-ISACがヘルスケアセクター向けに手口と緩和策を公表。同日、Abbottのがん診断事業への侵害も報じられました。

**7月26日 — 被害の「残響」。** ShinyHuntersがリークしたメールアドレスを、**無関係な第三者**が再利用し、ShinyHuntersを名乗って2,000ドルのBitcoinを要求するセクストーションメールを送信。標的にはAmtrak、Hallmark、Substack、Betterment、CarGurus、ADT、Panera Bread、McGraw Hillの過去漏洩データが使われていました。実際の端末侵害の証拠はなく、純粋な心理的圧迫です。**一度の漏洩は、その後何年も攻撃資源として流通し続ける**という事実を示します。

**7月28日 — 二正面。** DentaQuest（Sun Life傘下）の被害が最大2,340万人へ拡大（5月17〜20日の侵入、234GB、社会保障番号・Medicaid/Medicare番号・診断治療情報を含む。8月5日のHealth-ISAC続報では通知ベースで1,500万人、当初主張の5倍）。同日、Ernst & Youngへの**サプライチェーン経由**の侵害も犯行声明。

**8月4〜28日 — 息切れなし。** Brinks Home（Salesforceから490万件超主張、支払い拒否で41GB公開）、Trezor顧客1万4,000人（配送業者ShipMonkがMetabaseのSQLインジェクションゼロデイで侵害）、**ReliaQuest**（セキュリティ企業自身がフィッシング被害、影響は限定的と発表）、Carhartt（**Databricks**分析基盤の侵害、50GB超・1,290万アカウント＋1万5,000件の従業員メール、330万ドル要求を拒否され公開）。

**9月1日 — 現在進行形。** McKesson（北米の病院・薬局に処方薬の約3分の1を供給）が8月25日発覚の侵害を確認。ShinyHuntersは**2.84億件のレコード**窃取と**5,500万ドル**の要求を主張し、期限は9月1日。TechCrunchによれば侵入経路はフィッシング／ソーシャルエンジニアリングによる**SnowflakeおよびSalesforce**環境への侵入。診断名・薬剤・アレルギー・診療メモまで含むとされます。

**この流れが示すもの:** 4月のSalesforce設定不備 → 5月の教育セクター大量恐喝 → 6月のゼロデイ大量悪用 → 7月以降の医療への収斂。手口は入れ替わっても、**「大量PIIが集約されたデータ層に、正規の認証で到達する」**という目的関数は5ヶ月間まったくブレていません。

## 手口・技術詳細 (TTP)

### 初期侵入（3系統）

**① ID／ソーシャルエンジニアリング系（主力）**

- ビッシング（音声フィッシング）で従業員またはヘルプデスクを騙し、**MFAリセット・デバイス再登録**を実行させる（Health-ISAC）
- 対象IdP：Okta（ADT）、Microsoft Entra、Google SSO
- MFA疲弊攻撃、ヘルプデスクなりすまし（SecurityWeek 6/23）
- 通常のフィッシング（ReliaQuest、McKesson、7-Eleven）

**② サードパーティ／サプライチェーン系**

- アナリティクスベンダー **Anodot** の認証情報／トークン窃取 → 顧客のクラウドDWHへ（Rockstar、Vimeo、Zara）
- **Context.ai** 経由 → Vercel内部AIツール（起点は個人デバイスのLumma Stealer感染とOAuthトークン窃取）
- 配送業者 **ShipMonk**（Metabase SQLインジェクションゼロデイ）→ Trezor顧客
- 地域パートナー **GFN.am** → NVIDIA GeForce NOWユーザー
- **Ernst & Young** へのサプライチェーン経由の認証情報入手（本人主張）

**③ 技術的エクスプロイト系**

- **CVE-2026-35273**（Oracle PeopleSoft PeopleTools 8.61/8.62、CWE-306 重要機能の認証欠如、CVSS 9.8）。未認証・ユーザー操作不要のRCEでインスタンス完全乗っ取りが可能
- 悪用時期：2026年5月27日〜6月9日、300以上のインスタンス／100以上の組織
- CISA KEV追加：6月12日、是正期限6月15日（BOD 26-04）。追加時点でOracleパッチ未提供

### 侵害後の動き

- Python SimpleHTTPサーバによるツール配送
- **MeshCentral エージェントをAzureバイナリに偽装**してC2として使用（C2ドメイン：`azurenetfiles.net`）
- SSHによる水平移動
- 偵察とクレデンシャルスプレー
- カスタムクレデンシャルスティーラー **SANDCLOCK**（TeamPCP/UNC6780関連キャンペーン）

### 標的データ層

Salesforce（McGraw Hill、ADT、Instructure、7-Eleven、Infinite Campus、Brinks Home、McKesson）／Snowflake（Vimeo、Zara、McKesson）／BigQuery（Vimeo、Zara）／Databricks（Carhartt）／Oracle PeopleSoft（大学群、Nissan、NAIC）。悪用された設定不備として、**Salesforce Experience Cloudのゲストユーザー設定**と**SnowflakeのMFA未設定**が名指しされています。

### 恐喝モデル

暗号化なし。窃取 → 期限付き要求 → 拒否ならリークサイト公開。金額は判明分で7-Eleven 25万ドル、Vercel関連データ売却200万ドル、Carhartt 330万ドル、McKesson 5,500万ドルと、**標的の規模とデータ機微性に応じて2桁のレンジ**を持ちます。Canvas事案では**被害ベンダーではなく個々の教育機関へ個別交渉**を仕掛ける手法、ログインページ改ざんによる公然の圧力という新機軸も見られました。

## 影響と被害範囲

**規模上位（公表または攻撃者主張、いずれか明記）**

| 組織 | 規模 | 出所 |
|---|---|---|
| McKesson | 2.84億件（主張）、期限9/1 | 攻撃者主張 |
| Instructure/Canvas | 2億7,500万ユーザー・9,000機関・3.65TB | 攻撃者主張、Instructure公表 |
| Charter Communications | 4,200万レコード漏洩／490万ユニークメール | HIBP分析 |
| DentaQuest | 最大2,340万人（通知ベース1,500万人） | SecurityWeek／Health-ISAC |
| McGraw Hill | 1,350万アカウント | 確認値 |
| Carhartt | 1,290万アカウント | 確認値 |
| Carnival | 約600万人（870万レコード公開） | 確認値 |
| ADT | 約550万人 | 確認値 |
| Medtronic | 380万人 | 確認値 |
| 欧州評議会 | 職員1万人以上・42.9万文書 | 攻撃者主張、調査中 |
| ノッティンガム大学 | 45万5,000件 | 確認値 |

**データの質が悪化している。** 4月の初期事案は氏名・メール・電話中心でしたが、6月以降は**社会保障番号・銀行口座・税務情報**（Nissan）、**パスポート番号・障害情報・成績**（ノッティンガム大学）、**Medicaid/Medicare番号・診断治療情報**（DentaQuest）、**診断名・薬剤・アレルギー・診療メモ**（McKesson主張）へと機微度が上がっています。氏名とメールは変更できますが、SSNとパスポート番号と診断履歴は変更できません。

**セクター別の構造的リスク**

- **教育**：Dark Readingが指摘した通り、Canvas／Infinite Campusのような寡占LMS・SISへのベンダー集中リスク。英国では約160の高等教育機関が影響（CMC）。試験期間を狙われた運用インパクトも実害でした。
- **医療**：Health-ISACが2度警告（7/25、8/5）。Medtronic、DentaQuest、Abbott、McKesson。別途RunSafeの調査では、ヘルスケア組織の24%が医療機器標的の攻撃を受け、80%が患者ケアへの支障を報告、44%が既知の未パッチ脆弱性を持つ機器を運用中。ただしMedtronicもMcKessonも**医療機器の機能・患者安全への直接影響は否定**しています。
- **セキュリティ業界自身**：ReliaQuestの事例は、防御を売る側も同じフィッシングで抜かれることを示しました。

**「終わり」がない被害。** 7月26日のセクストーション事案が示すとおり、リークされたデータは第三者に再利用され、企業側の対応が完了した後も被害者を襲い続けます。

## 防御・推奨アクション

### 優先度1 — 今週中に着手（IDの入口を塞ぐ）

1. **ヘルプデスクの本人確認プロセスを書き換える。** MFAリセット／デバイス再登録の依頼に対し、**アウトオブバンドでの確認**（登録済みの上長経由、別チャネルでのコールバック等）を必須化。これが本キャンペーンの最大の入口です（Health-ISAC明示）。
2. **フィッシング耐性のあるMFA（FIDO2/WebAuthn）へ移行。** 少なくとも特権アカウント、IdP管理者、SaaS管理者から。SMS／プッシュ通知型はMFA疲弊とビッシングに耐えません。
3. **SnowflakeのMFA未設定アカウントを今日棚卸しする。** SecurityWeekが名指しした悪用ポイントです。Databricks、BigQueryも同様に。
4. **Salesforce Experience Cloudのゲストユーザー権限と公開ページ設定を監査する。** McGraw Hillの起点であり、複数組織に共通する構成不備です。

### 優先度2 — 今月中（データ層への到達を絶つ）

5. **OAuthトークン／Connected Apps／APIキーの全棚卸しとローテーション。** 特にアナリティクス・BI・AI系のサードパーティ統合。Anodot型の侵害では、**あなたの環境は無傷でもベンダー側のトークンで入られます**。スコープの最小化と有効期限の短縮を。
6. **CVE-2026-35273 の確認。** PeopleTools 8.61/8.62を運用しているなら、パッチ適用状況、EMHub／PSEMHUBエンドポイントのインターネット露出、CISA緩和策の適用を確認。既に侵害されている前提での遡及調査も。
7. **C2痕跡のハンティング。** `azurenetfiles.net` への通信、不審な**MeshCentral**接続、Azureサービスを装ったアウトバウンド、想定外のSSH水平移動。SANDCLOCK関連IOCのSIEM登録。
8. **クラウドDWHの異常データエクスポート検知。** 「認証は正常、ボリュームが異常」というパターンを検知できるか。ShinyHuntersはログインしてダウンロードするだけなので、**認証イベントの成否ではなくデータ移動量とアクセス元の異常**を見る必要があります。

### 優先度3 — 四半期内（構造の是正）

9. **サードパーティのデータアクセス台帳を作る。** どのベンダーが、どのデータストアに、どの権限で、どのトークンで繋がっているか。台帳がなければAnodot型侵害の影響範囲は特定できません。
10. **BYOD／個人デバイスからのSaaS接続ポリシーを見直す。** Vercel事案は個人デバイスのインフォスティーラー感染が起点でした。EDR配備とデバイス信頼性の要求を。
11. **ビッシング訓練を、メールフィッシング訓練と分けて実施する。** 特にヘルプデスク、IT運用、人事、経理。
12. **CSPM／SaaSセキュリティ態勢管理ツールの導入検討**と、CMCが推奨する**アプリケーション層とデータ層の分離**。
13. **恐喝対応のプレイブックを事前に作る。** Instructureは合意し、Charter・7-Eleven・Carhartt・Brinks Homeは拒否しました。方針を事案発生後に決めるのは最悪です。法務・広報・規制当局報告要件を含めて事前整備を。

### 個人／利用者向け

- 該当企業から提供されるクレジットモニタリング／ダークウェブ監視への登録
- Have I Been Pwned での確認
- ShinyHuntersを名乗る**セクストーションメールは支払わない**。実際の端末侵害の証拠はなく、過去の漏洩データを使った欺瞞です

## 今後の見通し

**① 医療セクターへの圧力は続く。** Health-ISACが2度警告を出し、Medtronic・DentaQuest・Abbott・McKessonと標的が連続していること、そしてPHIが最も高値で交渉できるデータであることを考えれば、この方向性は当面継続すると見るのが妥当です。McKessonの5,500万ドルという要求額は、これまでの2桁上のレンジであり、**医療データの「値付け」が上振れしている**シグナルかもしれません。

**② 「合意」の実効性は今後検証される。** InstructureはShinyHuntersとデータ削除で合意しましたが、条件は非公開です。Medtronicはリークサイトから削除されており支払いが示唆されています（未確認）。一方でCharter・7-Eleven・Carharttは拒否して公開されました。**支払った側のデータが後日流通しないかどうか**が、今後数ヶ月〜数年の観察点です。合意が守られなかった事例が出れば、業界の対応方針は一変します。

**③ ゼロデイ利用は例外か、新常態か。** PeopleSoftゼロデイの大量悪用は、IDベース攻撃を主軸としてきたこのグループにとって手口の拡張でした。ShipMonk事案のMetabase SQLインジェクションゼロデイも同様です。ただしこれが自前開発なのか、購入なのか、他アクターとの連携なのかは**報道時点では未確認**です。今後もERP・HR基盤といった「PIIが集中する業務システム」のゼロデイが狙われる可能性は高いと見られます。

**④ 法執行の影響は限定的と見るべき。** 7月にScattered Spiderメンバーの送還・起訴がありましたが、ShinyHunters本体の活動は8月・9月と途切れていません。緩やかなネットワーク型の犯罪エコシステムに対して、個人の逮捕がキャンペーンを停止させる効果は限定的です。

**⑤ リークデータの二次流通は数年単位で続く。** 7月のセクストーション事案が示す通りです。2026年上半期に流出した数億件のPIIは、これから何年も標的型フィッシングとソーシャルエンジニアリングの燃料になります。**今日の防御は、過去の漏洩を前提に組み立てる必要があります。**

**実務者への一行まとめ:** 「境界に穴がないか」より「正規のログインで、誰がどのデータをどれだけ持ち出せる状態か」を問い直してください。ShinyHuntersの5ヶ月間は、その一点を突き続けた記録です。

## 参照記事

| 日付 | タイトル / URL |
|---|---|
| 2026-04-09 | TeamPCP Supply Chain Campaign: Update 007 — Cisco Source Code Stolen via Trivy-Linked Breach<br>https://isc.sans.edu/diary/rss/32880 |
| 2026-04-14 | Stolen Rockstar Games analytics data leaked by extortion gang<br>https://www.bleepingcomputer.com/news/security/stolen-rockstar-games-analytics-data-leaked-by-extortion-gang/ |
| 2026-04-15 | McGraw-Hill confirms data breach following extortion threat<br>https://www.bleepingcomputer.com/news/security/mcgraw-hill-confirms-data-breach-following-extortion-threat/ |
| 2026-04-17 | Data breach at edtech giant McGraw Hill affects 13.5 million accounts<br>https://www.bleepingcomputer.com/news/security/data-breach-at-edtech-giant-mcgraw-hill-affects-135-million-accounts/ |
| 2026-04-18 | In Other News: Satellite Cybersecurity Act, $90K Chrome Flaw, Teen Hacker Arrested<br>https://www.securityweek.com/in-other-news-satellite-cybersecurity-act-90k-chrome-flaw-teen-hacker-arrested/ |
| 2026-04-21 | Vercel Employees' AI Tool Access Leads to Data Breach<br>https://www.darkreading.com/application-security/vercel-employees-ai-tool-access-data-breach |
| 2026-04-26 | ADT confirms data breach after ShinyHunters leak threat<br>https://www.bleepingcomputer.com/news/security/adt-confirms-data-breach-after-shinyhunters-leak-threat/ |
| 2026-04-28 | Medtronic confirms breach after hackers claim 9 million records theft<br>https://www.bleepingcomputer.com/news/security/medtronic-confirms-breach-after-hackers-claim-9-million-records-theft/ |
| 2026-04-28 | Home security giant ADT data breach affects 5.5 million people<br>https://www.bleepingcomputer.com/news/security/home-security-giant-adt-data-breach-affects-55-million-people/ |
| 2026-04-29 | Vimeo Confirms User and Customer Data Breach<br>https://www.securityweek.com/vimeo-confirms-user-and-customer-data-breach/ |
| 2026-04-30 | Quarter of Healthcare Organizations Hit by Medical Device Attacks<br>https://www.infosecurity-magazine.com/news/quarter-healthcare-medical-device/ |
| 2026-05-05 | EdTech Firm Instructure Discloses Data Breach<br>https://www.securityweek.com/edtech-firm-instructure-discloses-data-breach/ |
| 2026-05-06 | Hackers steal students' data during breach at education tech giant Instructure<br>https://techcrunch.com/2026/05/05/hackers-steal-students-data-during-breach-at-education-tech-giant-instructure/ |
| 2026-05-06 | Vimeo data breach exposes personal information of 119,000 people<br>https://www.bleepingcomputer.com/news/security/vimeo-data-breach-exposes-personal-information-of-119-000-people/ |
| 2026-05-07 | Instructure Breach Exposes Schools' Vendor Dependence<br>https://www.darkreading.com/cyberattacks-data-breaches/instructure-breach-exposes-schools-vendor-dependence |
| 2026-05-09 | ShinyHunters claims nearly 9,000 schools affected by Canvas data breach<br>https://edscoop.com/shinyhunters-claims-nearly-9000-schools-affected-by-canvas-data-breach/ |
| 2026-05-09 | NVIDIA confirms GeForce NOW data breach affecting Armenian users<br>https://www.bleepingcomputer.com/news/security/nvidia-confirms-geforce-now-data-breach-affecting-armenian-users/ |
| 2026-05-12 | Zara Data Breach Impacts 200,000<br>https://www.infosecurity-magazine.com/news/zara-data-breach-impacts-200000/ |
| 2026-05-12 | ShinyHunters Escalates Canvas Extortion Campaign<br>https://www.infosecurity-magazine.com/news/shinyhunters-escalates-canvas/ |
| 2026-05-13 | Deal Reached With Hackers to Delete Data Stolen From the Canvas Educational Platform<br>https://www.securityweek.com/deal-reached-with-hackers-to-delete-data-stolen-from-the-canvas-educational-platform/ |
| 2026-05-14 | Canvas Owner Reaches Agreement With Cybercriminals After Ransomware Attack<br>https://www.infosecurity-magazine.com/news/canvas-cybercriminals-agreement/ |
| 2026-05-19 | 7-Eleven Data Breach Confirmed After ShinyHunters Ransom Demand<br>https://www.securityweek.com/7-eleven-data-breach-confirmed-after-shinyhunters-ransom-demand/ |
| 2026-05-19 | Grafana Confirms Breach After Hackers Claim They Stole Data<br>https://www.securityweek.com/grafana-confirms-breach-after-hackers-claim-they-stole-data/ |
| 2026-05-20 | 7-Eleven confirms data breach claimed by the ShinyHunters gang<br>https://www.bleepingcomputer.com/news/security/7-eleven-confirms-data-breach-claimed-by-the-shinyhunters-gang/ |
| 2026-05-27 | 185,000 Likely Impacted by 7-Eleven Data Breach<br>https://www.securityweek.com/185000-likely-impacted-by-7-eleven-data-breach/ |
| 2026-05-29 | Carnival Data Breach Exposed 6 Million People<br>https://www.securityweek.com/carnival-data-breach-exposed-6-million-people/ |
| 2026-05-30 | Charter Communications Data Breach Could Impact Nearly 5 Million<br>https://www.securityweek.com/charter-communications-data-breach-could-impact-near-5-million/ |
| 2026-06-05 | DentaQuest data breach exposed info of 2.6 million accounts<br>https://www.bleepingcomputer.com/news/security/dentaquest-data-breach-exposed-info-of-26-million-accounts/ |
| 2026-06-11 | Oracle PeopleSoft servers hacked in ShinyHunters data theft attacks<br>https://www.bleepingcomputer.com/news/security/oracle-peoplesoft-servers-hacked-in-shinyhunters-data-theft-attacks/ |
| 2026-06-12 | ShinyHunters Exploits Oracle PeopleSoft Zero-Day (CVE-2026-35273) to Breach Universities<br>https://thehackernews.com/2026/06/shinyhunters-exploits-oracle-peoplesoft.html |
| 2026-06-12 | Nottingham University Data Breach Affects Over 450,000 Students<br>https://www.bleepingcomputer.com/news/security/nottingham-university-data-breach-affects-over-450-000-students/ |
| 2026-06-13 | CVE-2026-35273 — CISA Known Exploited Vulnerabilities Catalog<br>https://www.cisa.gov/known-exploited-vulnerabilities-catalog?search_api_fulltext=CVE-2026-35273 |
| 2026-06-13 | ShinyHunters is actively extorting universities after exploiting an unpatched Oracle flaw<br>https://cyberscoop.com/oracle-peoplesoft-zero-day-vulnerability-shinyhunters-extortion/ |
| 2026-06-13 | ShinyHunters Uses Oracle Zero-Day to Rampage Higher Ed<br>https://www.darkreading.com/vulnerabilities-threats/shinyhunters-oracle-zero-day-higher-ed |
| 2026-06-16 | Council of Europe investigates ShinyHunters data breach claims<br>https://www.bleepingcomputer.com/news/security/council-of-europe-investigates-shinyhunters-data-breach-claims/ |
| 2026-06-16 | Infinite Campus data breach affects 137,000 school staff accounts<br>https://www.bleepingcomputer.com/news/security/infinite-campus-data-breach-affects-137-000-school-staff-accounts/ |
| 2026-06-18 | Kodak confirms data breach claimed by ShinyHunters extortion gang<br>https://www.bleepingcomputer.com/news/security/kodak-confirms-data-breach-claimed-by-shinyhunters-extortion-gang/ |
| 2026-06-19 | Kodak Admits Data Breach After ShinyHunters Hack Claims<br>https://www.securityweek.com/kodak-admits-data-breach-after-shinyhunters-hack-claims/ |
| 2026-06-23 | What the Latest ShinyHunters Breaches Reveal About Modern Cyberattacks<br>https://www.securityweek.com/what-the-latest-shinyhunters-breaches-reveal-about-modern-cyberattacks/ |
| 2026-06-27 | CMC Releases Analysis and Guidance for Education Sector After Canvas Data Breach<br>https://www.infosecurity-magazine.com/news/cmc-analysis-education-canvas-data/ |
| 2026-06-30 | Nissan discloses employee data breach linked to Oracle zero-day attacks<br>https://www.bleepingcomputer.com/news/security/nissan-discloses-employee-data-breach-linked-to-oracle-zero-day-attacks/ |
| 2026-06-30 | NAIC says public data stolen in ShinyHunters' PeopleSoft breach<br>https://www.bleepingcomputer.com/news/security/naic-says-public-data-stolen-in-shinyhunters-peoplesoft-breach/ |
| 2026-07-01 | Nissan Discloses Employee Data Breach Linked to Oracle Zero-Day<br>https://www.infosecurity-magazine.com/news/nissan-oracle-peoplesoft-zero-day/ |
| 2026-07-03 | Medtronic notifies customers impacted by ShinyHunters data breach<br>https://www.bleepingcomputer.com/news/security/medtronic-notifies-customers-impacted-by-shinyhunters-data-breach/ |
| 2026-07-03 | Alleged Scattered Spider Member Extradited to US<br>https://www.infosecurity-magazine.com/news/scattered-spider-member-extradited/ |
| 2026-07-04 | Medtronic Data Breach Impacts 3.8 Million People<br>https://www.securityweek.com/medtronic-data-breach-impacts-3-8-million-people/ |
| 2026-07-25 | Shiny Hunters Impact to Health Sector and Recommended Mitigation Strategies<br>https://health-isac.org/shiny-hunters-impact-to-health-sector-and-recommended-mitigation-strategies/ |
| 2026-07-25 | In Other News: Dolphin X AI-Powered Malware, Car Anti-Theft Device Hack, 400 Linux Kernel Flaws<br>https://www.securityweek.com/in-other-news-dolphin-x-ai-powered-malware-car-anti-theft-device-hack-400-linux-kernel-flaws/ |
| 2026-07-26 | ShinyHunters data leaks fuel $2,000 sextortion email scam<br>https://www.bleepingcomputer.com/news/security/shinyhunters-data-leaks-fuel-2-000-sextortion-email-scam/ |
| 2026-07-28 | DentaQuest Data Breach Potentially Impacts Over 23 Million People<br>https://www.securityweek.com/dentaquest-data-breach-potentially-impacts-over-23-million-people/ |
| 2026-07-28 | Ernst & Young data breach claimed by ShinyHunters extortion gang<br>https://www.bleepingcomputer.com/news/security/ernst-and-young-data-breach-claimed-by-shinyhunters-extortion-gang/ |
| 2026-08-04 | Brinks Home Discloses Data Breach as Hackers Leak Files<br>https://www.securityweek.com/brinks-home-discloses-data-breach-as-hackers-leak-files/ |
| 2026-08-05 | DentaQuest Data Theft Hack Affects 15M Patients<br>https://health-isac.org/dentaquest-data-theft-hack-affects-15m-patients/ |
| 2026-08-05 | Health-ISAC warns of rising ShinyHunters data theft attacks on healthcare<br>https://health-isac.org/health-isac-warns-of-rising-shinyhunters-data-theft-attacks-on-healthcare/ |
| 2026-08-15 | 14,000 Trezor Customers Impacted by Data Breach at ShipMonk<br>https://www.securityweek.com/14000-trezor-customers-impacted-by-data-breach-at-shipmonk/ |
| 2026-08-25 | ReliaQuest Confirms ShinyHunters Hack, but Says Impact Was Limited<br>https://www.securityweek.com/reliaquest-confirms-shinyhunters-hack-but-says-impact-was-limited/ |
| 2026-08-28 | Carhartt data breach exposes information of 12.9 million accounts<br>https://www.bleepingcomputer.com/news/security/carhartt-data-breach-exposes-information-of-129-million-accounts/ |
| 2026-09-01 | McKesson Confirms Data Breach as Attacker Deadline Looms<br>https://www.securityweek.com/mckesson-confirms-data-breach-as-attacker-deadline-looms/ |
| 2026-09-01 | Hackers claim millions of patient records stolen during data breach at healthcare giant McKesson<br>https://techcrunch.com/2026/08/31/hackers-claim-millions-of-patient-records-stolen-during-data-breach-at-healthcare-giant-mckesson/ |
