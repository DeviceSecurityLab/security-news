---
title: "ShinyHunters — クラウドSaaSの「設定」と「人」を狙い続けた恐喝マシンの2か月"
date: 2026-06-21T07:00:00+09:00
tags: ["security", "intelligence", "深掘り", "ShinyHunters"]
draft: false
---

## 30秒サマリ

- 2026年4〜6月、恐喝グループShinyHunters（Mandiantの呼称UNC6240）が、McGraw-Hill、ADT、Medtronic、Vimeo、Instructure(Canvas)、Zara、7-Eleven、Carnival、Charter、DentaQuest、Kodak、欧州評議会、多数の大学など数十組織への侵害を主張・確認させた。
- 攻撃の主軸は3系統。①Salesforce設定不備・OAuth連携の悪用、②ビッシング/ソーシャルエンジニアリングによる従業員SSO(Okta)乗っ取り、③サードパーティ分析ベンダー（Anodot等）のトークン窃取によるSnowflake/BigQuery侵入。
- 6月には新展開として、Oracle PeopleSoftのゼロデイ **CVE-2026-35273**（CVSS 9.8、認証不要RCE）を悪用し、300超のインスタンス・100超の組織を侵害。その68%が米国の高等教育機関。
- 共通する事業モデルは「pay or leak（払うか暴露か）」。期限を切って身代金を要求し、決裂するとダークウェブで公開する。
- 防御の要点は、SaaS（特にSalesforce）の公開設定とConnected App/OAuthトークンの棚卸し、フィッシング耐性のあるMFA、ビッシング訓練、そしてPeopleSoftの緊急緩和。

## 背景 — 何者か / 何が起きているか

ShinyHuntersは、データを窃取して暴露をちらつかせ金銭を要求する「恐喝（extortion）グループ」です。ランサムウェアのようにシステムを暗号化するのではなく、**データを盗んで公開を脅迫する**点が特徴です。本データ群では、Mandiant/Google Threat Intelligence Groupが同グループを **UNC6240** として追跡しており、報道によってはScattered SpiderやLapsus$、「Coinbase Cartel」といった他グループとの関連も指摘されています（ただし各グループの関係性の詳細は報道時点で確定的ではありません）。

この2か月で観測されたのは、単発の侵害ではなく**同一の手口を横展開する波状攻撃**です。特に「Salesforceを中心とする企業向けSaaS」と「従業員という人的経路」を繰り返し突いており、被害は教育・医療・小売・通信・エンタメ・国際機関と業種を問いません。

## 時系列 — インシデントの連鎖

**第1波：開発サプライチェーンとアナリティクスベンダー（4月上旬〜中旬）**

- **4/09** Cisco関連の侵害が報じられる。Trivyのサプライチェーン侵害（CVE-2026-33634）を起点とするTeamPCPキャンペーン（UNC6780）の盗取データを、ShinyHuntersが**恐喝に転用**してFBI・DHS等を標的にしたとされる。盗む者と脅す者が役割分担する「データの二次流通」を示す。
- **4/14** Rockstar Games。分析基盤ベンダー **Anodot** 経由の漏洩データが公開される。

**第2波：Salesforce設定不備の連鎖（4月中旬）**

- **4/15→4/17** McGraw-Hill。Salesforceホスト型ウェブページの**設定ミス**が原因。当初「限定的・非機密」とされたが、最終的に**1,350万アカウント**の氏名・住所・電話・メール（100GB超）が流布。攻撃者側は4,500万件のSalesforceレコードを主張。
- ここで「Salesforceの誤設定が複数組織に共通の弱点」というパターンが明確化。

**第3波：人を狙う（ビッシング＋SSO）（4月下旬）**

- **4/21** Vercel。社員の個人デバイスがLumma Stealer（Robloxチートに偽装）に感染しOAuthトークンが盗まれ、内部AIツールへ不正アクセス。**個人デバイス→SaaS統合→内部システム**という横移動を実証。
- **4/26→4/28** ADT。**ビッシングで従業員のOkta SSOを乗っ取り**、Salesforceへ到達。最終的に**約550万人**に影響（2024年に続く再侵害）。
- **4/28** Medtronic。同時期に同手口（ビッシング＋SSO）で**900万件**主張、コーポレートIT侵害を確認。医療機器・患者安全への影響は否定。

**第4波：アナリティクスベンダー再来とクラウドDWH（4月末〜5月）**

- **4/29→5/06** Vimeo。Anodotの認証情報窃取でVimeoの **Snowflake/BigQuery** に侵入、**約11.9万人**のメール等が流出。
- **5/09** NVIDIA GeForce NOW。地域パートナーGFN.am経由でアルメニアユーザーのデータが流出（当初ShinyHuntersが数百万件を主張）。
- **5/12** Zara。同じくAnodot→トークン→BigQuery/Snowflakeの経路で**約19.7万人**。報道は明確に「Vimeo・Rockstar・Instructureと同じ pay or leak キャンペーンの一環」と位置づけた。

**第5波：教育セクター大規模化（5月）**

- **5/05〜5/09** Instructure（Canvas LMS）。Salesforceインスタンス侵害で**3.65TB／約2億7,500万人／約9,000教育機関**（Harvard・MIT等を含むと主張）。**5/12が支払期限**。さらに数百校のログインページを改ざんし、**学校ごとに個別恐喝**へエスカレート（期末試験期間を直撃）。
- **5/13→5/14** Instructureがデータ削除で**合意に到達**と報道。各機関に「直接交渉は不要」と通知。ただし合意条件は非公開で、削除の実効性は保証されない。

**第6波：小売・旅行・通信・医療の大規模PII（5月中旬〜6月上旬）**

- **5/19→5/27** 7-Eleven。Salesforce環境侵害（フィッシング／設定不備が推定）で**9.4GB・約60万レコード**、影響者**約18.5万人**。$250,000要求→決裂→リーク。
- **5/19** Grafana。侵害を確認（「Coinbase Cartel」関連と報道）。
- **5/29** Carnival。**ソーシャルエンジニアリングで従業員アカウント侵害**、約600万人。後に870万件公開。政府発行ID番号を含む。
- **5/30** Charter Communications。4,200万件リーク、HaveIBeenPwn分析で**490万件**のユニークメール＋8.5万件の従業員レコード。身代金は支払われなかったとみられる。
- **6/05** DentaQuest（Sun Life傘下）。**234GB・260万アカウント**、健康保険情報・政府発行IDを含む。交渉決裂で公開。

**第7波：Oracle PeopleSoftゼロデイ（6月）— 手口の質的転換**

- **6/11** Oracle PeopleSoftサーバが標的と報道、100超の組織からの窃取を主張。
- **6/12** The Hacker News／BleepingComputerが詳細を報道。**CVE-2026-35273**（CVSS 9.8、PeopleTools 8.61/8.62、認証不要・ユーザー操作不要のRCE）を悪用。攻撃は**5/27開始**。ノッティンガム大学で**45万5千件**の固有メール流出（パスポート番号・障害情報・成績・支払記録等を含む）。
- **6/13** CISAがKEVに追加（緩和期限**6/15**、実質3日）。CyberScoop/Dark Readingが続報：**5/27〜6/9に300超のインスタンス**侵害、影響組織の**68%が米国高等教育機関**。
- **6/16** 欧州評議会が調査開始。**42.9万件超の文書**（2011〜2026年の給与明細40.9万件、職員1万人超、人事ファイル、履歴書）に銀行口座・社会保険番号・医療記録を含むと主張、6/16公開を脅迫。
- **6/16** Infinite Campus（K-12学生情報システム）。Salesforce経由で**13.7万教職員アカウント**（攻撃は3月発生）。
- **6/18→6/19** Kodak。**220万件超**（顧客PII＋内部企業データ）、6/18公開予告。

**読み取れる流れ**：4〜5月は「SaaSの設定／人の認証情報／ベンダーのトークン」という**既存の弱点を量産的に突く**フェーズ。6月は**ゼロデイ（CVE-2026-35273）という新兵器**を加え、特に教育機関を集中砲火するフェーズへ移行しました。攻撃面を「設定ミス」から「未パッチの製品脆弱性」へ広げた点が、この期間最大の変化です。

## 手口・技術詳細 (TTP)

報道された範囲で、ShinyHuntersの初期侵入は主に4経路に整理できます。

1. **Salesforce設定不備／OAuth・Connected App悪用**：McGraw-Hill、7-Eleven、Infinite Campus、Instructure等。公開ウェブページの設定ミスや、過剰権限の連携アプリ・OAuthトークンを起点にCRMデータを大量エクスポート。
2. **ビッシング（音声フィッシング）＋SSO乗っ取り**：ADT、Medtronic、Carnival等。従業員を電話で騙してOkta等のSSOを奪取し、MFAを回避してSalesforce等の中枢へ到達。
3. **サードパーティ分析ベンダーのトークン窃取→クラウドDWH**：Vimeo、Zara、（関連して）Rockstar。Anodotの認証情報を盗み、顧客側のSnowflake/BigQueryへ不正アクセス。
4. **インフォスティーラー経由のトークン窃取**：Vercel。個人デバイスのLumma Stealer（ゲームチート偽装）でOAuthトークンを取得し、企業SaaS統合へ。

**PeopleSoftキャンペーンの技術詳細（6月）**：

- 脆弱性 **CVE-2026-35273**（CWE-306：重要機能の認証欠如）。**PSEMHub/EMHubエンドポイント**が攻撃面。
- 侵入後、**Python SimpleHTTPサーバ**でツールを配布し、**Azureバイナリに偽装したMeshCentralエージェント**をC2として展開。C2ドメインとして **azurenetfiles.net** が報告。Azureサービスへの正規通信に紛れる偽装が特徴。
- その後 **SSHによる水平移動**、偵察、**クレデンシャルスプレー**を実施しデータを窃取。

**恐喝の運用**：期限を切った身代金要求（例：7-Elevenに$250,000、Canvasは5/12期限、Kodakは6/18公開予告）。決裂時はダークウェブフォーラム／リークサイトで公開。Instructureのように「データ削除の合意」に至るケースもあるが、条件非公開で実効性は保証されない。

## 影響と被害範囲

- **規模**：単体で100万件超の流出が複数（McGraw-Hill 1,350万、Medtronic 900万主張、ADT 550万、Carnival 600万、Charter 490万ユニークメール、DentaQuest 260万、Instructure 2億7,500万主張、Kodak 220万）。
- **データの質**：氏名・住所・電話・メールに加え、**生年月日・社会保障番号の一部・政府発行ID・パスポート番号・健康保険情報・医療記録・銀行口座**まで含む事案がある（DentaQuest、Carnival、欧州評議会、ノッティンガム大学）。これらは長期的ななりすまし・金融詐欺・標的型フィッシングの燃料になる。
- **セクター集中**：教育機関（Instructure/Canvas、Infinite Campus、PeopleSoft悪用の大学群）が突出。医療（Medtronic、DentaQuest）、重要インフラ寄り（Charter、ADT）、国際機関（欧州評議会）にも波及。
- **共通教訓**：直接の自社防御だけでは不十分で、**ベンダー（Anodot、地域パートナーGFN.am、第三者プラットフォーム）経由**の侵害が繰り返し起きている点が重い。

## 防御・推奨アクション

実務者が優先度順に着手すべき項目です。

**最優先（48時間以内）**

1. **PeopleSoft緊急対応**：CVE-2026-35273に対しCISAの緩和策を即時適用。**EMHub/PSEMHubエンドポイントのインターネット露出を遮断・隔離**。パッチ提供後は速やかに適用。MeshCentralの不審通信、Azureを装うアウトバウンド接続、SSH水平移動の痕跡を調査。
2. **Salesforce棚卸し**：公開ウェブフォーム／外部向けページのアクセス制御を監査。Connected Apps・OAuthトークン・APIキーを総点検し、**最小権限化・不要連携の失効・トークンローテーション**。異常な大量データエクスポートの兆候を調査。

**高優先（1〜2週間）**

3. **人的経路の遮断**：ビッシング対策訓練を強化し、**フィッシング耐性のあるMFA（FIDO2等）**を導入。Okta等SSOの不審ログイン検知ルールを見直す。
4. **サードパーティ／ベンダー管理**：分析ベンダー等に渡している認証トークンを棚卸し・ローテーション。Snowflake/BigQueryへの第三者アクセスを最小化し、アクセスログを精査。
5. **インフォスティーラー対策**：EDR導入、個人デバイスからの企業SaaS接続ポリシー見直し、ゲームMOD/チート等の持ち込み制限。

**継続的**

6. **CSPM（クラウド設定管理）**で公開設定を定期監査。IOC（azurenetfiles.net、SANDCLOCK関連等）をSIEM/検知ルールへ登録。データ最小化と保持期間短縮で「盗られても価値が低い」状態を作る。
7. **恐喝対応プレイブック**整備：期限付き要求への対応方針、規制当局への報告要件（GDPR/各国法）を事前に確認。

## 今後の見通し

- **量から質への移行が続く可能性**：4〜5月の「設定ミス・トークン窃取の量産」に、6月の「ゼロデイ悪用」が加わりました。今後もパッチ未提供のエンタープライズ製品（人事・財務・学務系の基幹SaaS/オンプレ）が狙われやすいと見られます。
- **教育セクターへの圧力継続**：Canvas、Infinite Campus、PeopleSoft悪用の大学群と、教育機関の集中が顕著。期末試験などの繁忙期を狙うタイミングも観測されており、当面ターゲットであり続ける可能性が高い。
- **「合意」の信頼性は不透明**：Instructureの削除合意のように交渉妥結例はあるものの、条件非公開で再公開リスクは残ります。支払い／合意がデータ消去を保証しない前提での対応が必要です。
- なお、各グループ（ShinyHunters／Scattered Spider／Lapsus$／Coinbase Cartel）の関係や帰属の一部は**報道時点で未確定**であり、続報での更新を要します。

## 参照記事

- TeamPCP Supply Chain Campaign: Update 007 - Cisco Source Code Stolen via Trivy-Linked Breach (SANS ISC, 2026-04-09) — https://isc.sans.edu/diary/rss/32880
- Stolen Rockstar Games analytics data leaked by extortion gang (BleepingComputer, 2026-04-14) — https://www.bleepingcomputer.com/news/security/stolen-rockstar-games-analytics-data-leaked-by-extortion-gang/
- McGraw-Hill confirms data breach following extortion threat (BleepingComputer, 2026-04-15) — https://www.bleepingcomputer.com/news/security/mcgraw-hill-confirms-data-breach-following-extortion-threat/
- Data breach at edtech giant McGraw Hill affects 13.5 million accounts (BleepingComputer, 2026-04-17) — https://www.bleepingcomputer.com/news/security/data-breach-at-edtech-giant-mcgraw-hill-affects-135-million-accounts/
- In Other News: Satellite Cybersecurity Act, $90K Chrome Flaw, Teen Hacker Arrested (SecurityWeek, 2026-04-18) — https://www.securityweek.com/in-other-news-satellite-cybersecurity-act-90k-chrome-flaw-teen-hacker-arrested/
- Vercel社員のAIツールアクセスがデータ侵害に — Lumma StealerとShinyHunters関与 (Dark Reading, 2026-04-21) — https://www.darkreading.com/application-security/vercel-employees-ai-tool-access-data-breach
- ADT confirms data breach after ShinyHunters leak threat (BleepingComputer, 2026-04-26) — https://www.bleepingcomputer.com/news/security/adt-confirms-data-breach-after-shinyhunters-leak-threat/
- Medtronic confirms breach after hackers claim 9 million records theft (BleepingComputer, 2026-04-28) — https://www.bleepingcomputer.com/news/security/medtronic-confirms-breach-after-hackers-claim-9-million-records-theft/
- Home security giant ADT data breach affects 5.5 million people (BleepingComputer, 2026-04-28) — https://www.bleepingcomputer.com/news/security/home-security-giant-adt-data-breach-affects-55-million-people/
- Vimeo Confirms User and Customer Data Breach (SecurityWeek, 2026-04-29) — https://www.securityweek.com/vimeo-confirms-user-and-customer-data-breach/
- 医療機器サイバー攻撃、ヘルスケア組織の4分の1が被害報告 (Infosecurity Magazine, 2026-04-30) — https://www.infosecurity-magazine.com/news/quarter-healthcare-medical-device/
- EdTech企業Instructureがデータ侵害を公表 (SecurityWeek, 2026-05-05) — https://www.securityweek.com/edtech-firm-instructure-discloses-data-breach/
- 教育テクノロジー大手 Instructure でデータ侵害 (TechCrunch, 2026-05-06) — https://techcrunch.com/2026/05/05/hackers-steal-students-data-during-breach-at-education-tech-giant-instructure/
- Vimeo data breach exposes personal information of 119,000 people (BleepingComputer, 2026-05-06) — https://www.bleepingcomputer.com/news/security/vimeo-data-breach-exposes-personal-information-of-119-000-people/
- Instructure Breach Exposes Schools' Vendor Dependence (Dark Reading, 2026-05-07) — https://www.darkreading.com/cyberattacks-data-breaches/instructure-breach-exposes-schools-vendor-dependence
- ShinyHunters claims nearly 9,000 schools affected by Canvas data breach (CyberScoop, 2026-05-09) — https://edscoop.com/shinyhunters-claims-nearly-9000-schools-affected-by-canvas-data-breach/
- NVIDIA confirms GeForce NOW data breach affecting Armenian users (BleepingComputer, 2026-05-09) — https://www.bleepingcomputer.com/news/security/nvidia-confirms-geforce-now-data-breach-affecting-armenian-users/
- Zaraデータ侵害、約20万人の顧客に影響 (Infosecurity Magazine, 2026-05-12) — https://www.infosecurity-magazine.com/news/zara-data-breach-impacts-200000/
- ShinyHuntersがCanvasへの恐喝キャンペーンを拡大 (Infosecurity Magazine, 2026-05-12) — https://www.infosecurity-magazine.com/news/shinyhunters-escalates-canvas/
- Deal Reached With Hackers to Delete Data Stolen From the Canvas Educational Platform (SecurityWeek, 2026-05-13) — https://www.securityweek.com/deal-reached-with-hackers-to-delete-data-stolen-from-the-canvas-educational-platform/
- Canvas Owner Reaches Agreement With Cybercriminals After Ransomware Attack (Infosecurity Magazine, 2026-05-14) — https://www.infosecurity-magazine.com/news/canvas-cybercriminals-agreement/
- 7-Eleven Data Breach Confirmed After ShinyHunters Ransom Demand (SecurityWeek, 2026-05-19) — https://www.securityweek.com/7-eleven-data-breach-confirmed-after-shinyhunters-ransom-demand/
- Grafana Confirms Breach After Hackers Claim They Stole Data (SecurityWeek, 2026-05-19) — https://www.securityweek.com/grafana-confirms-breach-after-hackers-claim-they-stole-data/
- 7-Eleven confirms data breach claimed by the ShinyHunters gang (BleepingComputer, 2026-05-20) — https://www.bleepingcomputer.com/news/security/7-eleven-confirms-data-breach-claimed-by-the-shinyhunters-gang/
- 185,000 Likely Impacted by 7-Eleven Data Breach (SecurityWeek, 2026-05-27) — https://www.securityweek.com/185000-likely-impacted-by-7-eleven-data-breach/
- Carnival Data Breach Exposed 6 Million People (SecurityWeek, 2026-05-29) — https://www.securityweek.com/carnival-data-breach-exposed-6-million-people/
- Charter Communications Data Breach Could Impact Nearly 5 Million (SecurityWeek, 2026-05-30) — https://www.securityweek.com/charter-communications-data-breach-could-impact-near-5-million/
- DentaQuest data breach exposed info of 2.6 million accounts (BleepingComputer, 2026-06-05) — https://www.bleepingcomputer.com/news/security/dentaquest-data-breach-exposed-info-of-26-million-accounts/
- Oracle PeopleSoft servers hacked in ShinyHunters data theft attacks (BleepingComputer, 2026-06-11) — https://www.bleepingcomputer.com/news/security/oracle-peoplesoft-servers-hacked-in-shinyhunters-data-theft-attacks/
- ShinyHunters Exploits Oracle PeopleSoft Zero-Day (CVE-2026-35273) to Breach Universities (The Hacker News, 2026-06-12) — https://thehackernews.com/2026/06/shinyhunters-exploits-oracle-peoplesoft.html
- Nottingham University Data Breach Affects Over 450,000 Students (BleepingComputer, 2026-06-12) — https://www.bleepingcomputer.com/news/security/nottingham-university-data-breach-affects-over-450-000-students/
- CVE-2026-35273: Oracle PeopleSoft Enterprise PeopleTools Missing Authentication (CISA KEV, 2026-06-13) — https://www.cisa.gov/known-exploited-vulnerabilities-catalog?search_api_fulltext=CVE-2026-35273
- ShinyHunters is actively extorting universities after exploiting an unpatched Oracle flaw (CyberScoop, 2026-06-13) — https://cyberscoop.com/oracle-peoplesoft-zero-day-vulnerability-shinyhunters-extortion/
- ShinyHunters Uses Oracle Zero-Day to Rampage Higher Ed (Dark Reading, 2026-06-13) — https://www.darkreading.com/vulnerabilities-threats/shinyhunters-oracle-zero-day-higher-ed
- Council of Europe investigates ShinyHunters data breach claims (BleepingComputer, 2026-06-16) — https://www.bleepingcomputer.com/news/security/council-of-europe-investigates-shinyhunters-data-breach-claims/
- Infinite Campus data breach affects 137,000 school staff accounts (BleepingComputer, 2026-06-16) — https://www.bleepingcomputer.com/news/security/infinite-campus-data-breach-affects-137-000-school-staff-accounts/
- Kodak confirms data breach claimed by ShinyHunters extortion gang (BleepingComputer, 2026-06-18) — https://www.bleepingcomputer.com/news/security/kodak-confirms-data-breach-claimed-by-shinyhunters-extortion-gang/
- Kodak Admits Data Breach After ShinyHunters Hack Claims (SecurityWeek, 2026-06-19) — https://www.securityweek.com/kodak-admits-data-breach-after-shinyhunters-hack-claims/
