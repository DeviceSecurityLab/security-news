---
title: "ShinyHunters — 「脆弱性」ではなく「ログイン」を狩る、SaaS恐喝の産業化"
date: 2026-07-28T07:00:00+09:00
tags: ["security", "intelligence", "深掘り", "ShinyHunters"]
draft: false
---

## 30秒サマリ

- ShinyHunters（UNC6240）は2026年4〜7月にかけ、Rockstar、McGraw-Hill、Instructure(Canvas)、7-Eleven、Carnival、Charter、Medtronic、DentaQuest、Nissan、欧州評議会、EYなど数十組織を連続して侵害した、データ窃取・恐喝（pay or leak）特化グループ。
- 攻撃の主軸は3系統：①ビッシング＋SSO(Okta/Entra)乗っ取りによるSalesforce窃取、②サードパーティ分析ベンダー(Anodot)のトークン窃取によるSnowflake/BigQuery侵入、③Oracle PeopleSoftゼロデイ(CVE-2026-35273)悪用。
- 共通するのは「マルウェアや脆弱性ではなくID（認証情報・OAuthトークン・MFA）を狙う」構造的シフト。境界防御やパッチ管理だけでは防げない。
- 被害は個人情報（氏名・生年月日・SSN・パスポート番号・健康/保険情報）が中心で、教育機関・ヘルスケアが特に集中的に狙われた。
- リークデータは削除合意後も長期流通し、7月にはセクストーション詐欺の燃料として二次悪用が確認された。侵害の影響は初動対応で終わらない。

## 背景 — 何者か / 何が起きているか

ShinyHuntersは、データを盗んで「支払わなければ公開する（pay or leak）」形で金銭を要求する恐喝グループです。Google/Mandiantは本キャンペーン群を**UNC6240**として追跡しています。記事群からは、同グループがLapsus$やScattered Spiderといった若年層ハッカーの緩やかなネットワーク（一部報道では「Coinbase Cartel」とも）と重なりを持つことが示唆されています（7/3のScattered Spiderメンバー起訴記事、5/19のGrafana記事）。ただしグループ間の正確な関係は報道時点で完全には特定されていません。

重要なのは、SecurityWeekが6/23にまとめた通り、この攻撃者が**「エクスプロイトやマルウェアからIDベースの攻撃への構造的シフト」**を体現している点です。ソフトウェアの穴ではなく、ID管理の穴を突く。「ログインするだけ」の攻撃は、従来の境界防御では検知できません。

観測期間中、標的は業種横断でしたが、**教育機関とヘルスケア**への集中が顕著でした。いずれも大量の機微な個人情報を保有し、レガシーシステムやサードパーティ依存が高いセクターです。

## 時系列 — インシデントの連鎖

攻撃手口ごとに3つの「波」が重なりながら進行しました。

**第1波：サードパーティ分析ベンダー(Anodot)経由のクラウドDWH侵害**

- **04-14 Rockstar Games** — 分析ベンダーAnodotへの侵害を起点にデータ漏洩、ShinyHuntersが公開。
- **04-29 / 05-06 Vimeo** — Anodotの認証情報が盗まれ、VimeoのSnowflake/BigQueryに不正アクセス。約11.9万人のメール等が流出。
- **05-12 Zara(Inditex)** — 同じくAnodot起点でBigQuery/Snowflakeを侵害、約19.7万人に影響。

Anodotという単一ベンダーのトークンが、複数の大手企業のクラウドデータ基盤への「合鍵」になった典型的サプライチェーン連鎖です。

**第2波：ビッシング＋SSO乗っ取りによるSalesforce大量窃取**

- **04-15〜04-17 McGraw-Hill** — Salesforce設定不備を悪用。当初「限定的」と主張したが、最終的に1,350万アカウント・100GB超と判明。企業と攻撃者の主張の食い違いが繰り返されるパターンの初期例。
- **04-21 Vercel** — 社員個人デバイスがLumma Stealer（Robloxチート偽装）に感染しOAuthトークン窃取。内部AIツールに侵入、200万ドルで売却を試みた。
- **04-26〜04-28 ADT** — 従業員のOkta SSOをビッシングで乗っ取りSalesforceへ。550万人に影響（2024年に続く再侵害）。
- **04-28 / 07-03 / 07-04 Medtronic** — 同時期・同手口。最終的に380万人、SSN・健康情報を含む漏洩と確定。
- **05-05〜05-14 Instructure(Canvas LMS)** — Salesforce侵害で3.65TB・約9,000教育機関・2億7,500万人に影響と主張。ハーバードやMITも含むとされた。5/12には数百校のログインページを改ざんし**学校ごとに個別恐喝**へ拡大。5/13〜14に「データ削除合意」に達したと報じられたが、実効性は保証されない（試験期間を狙った点も悪質）。
- **05-19〜05-27 7-Eleven** — Salesforce侵害で約18.5万人。身代金25万ドルを拒否後にリーク。
- **05-29 Carnival** — ソーシャルエンジニアリングで従業員アカウント侵害、600万人・政府発行ID番号まで流出。
- **05-30 Charter Communications** — 約490万人＋従業員8.5万件。身代金は不払いとみられる。
- **06-05〜07-28 DentaQuest** — 当初260万件と報じられたが、7/28には最大2,340万人規模へ拡大。SSN・保険情報・診療情報を含む。
- **06-16 Infinite Campus** — K-12向け学生情報システム、13.7万教職員アカウント。

**第3波：Oracle PeopleSoftゼロデイ(CVE-2026-35273)悪用**

- **05-27頃〜06-09** — 攻撃開始。CVSS 9.8、認証不要のRCE。300以上のインスタンス、100以上の組織を侵害（うち**68%が米国の高等教育機関**）。
- **06-11〜06-13** — Mandiant/GTIGが追跡・通知。MeshCentralをAzure偽装でC2に使用、SSH水平移動。**06-12 ノッティンガム大学**で45.5万人の学生データ流出（パスポート番号・障害情報含む）。
- **06-13 CISA KEV登録** — 是正期限6/15（実質3日間）。Oracleは攻撃開始後に初めて緊急アドバイザリを発行した。
- **06-16 欧州評議会** — 42.9万件のHR・給与文書窃取を主張（銀行口座・医療記録含む）。
- **06-30〜07-01 Nissan** — 従業員のSSN・銀行口座・税務情報が米・加・墨・伯で流出。
- **06-30 NAIC(全米保険監督官協会)** — 攻撃者は3.1TB主張、機関側は「公開データのみ」と反論。主張の食い違いが続く。

**そして二次悪用フェーズ**

- **07-25 Health-ISAC** — ヘルスケア向けにビッシング＋MFAリセット型の新恐喝モデルを警告。
- **07-26** — ShinyHuntersの過去リークデータ（ADT、McGraw Hill、Panera Bread等）を再利用した**2,000ドル要求のセクストーション詐欺**が発生。実際の端末侵害はなく、リークデータによる心理的圧迫。
- **07-28 Ernst & Young(EY)** — サプライチェーン経由の認証情報入手を主張。

なお04-09のCisco/Trivyサプライチェーン侵害（UNC6780/TeamPCP）では、ShinyHuntersは盗取データを使ってFBI・DHS等を標的に恐喝を展開しており、他グループの成果物を恐喝に転用する「エコシステム」的な動きも見られます。

## 手口・技術詳細 (TTP)

記事群から確認できるTTPは、大きく3つの初期アクセス経路に集約されます。

**1. ID侵害系（最も多用）**

- **ビッシング（音声ソーシャルエンジニアリング）**：ヘルプデスクや従業員を電話で騙し、Okta / Microsoft Entra / Google SSOのMFAリセットやデバイス再登録を実行させる（Health-ISAC、ADT、Medtronic）。
- **MFA疲弊攻撃・ヘルプデスクなりすまし**（SecurityWeek 06-23）。
- 乗っ取ったSSOからSalesforce、M365、SharePointへ横展開しデータをエクスポート。
- Salesforce Experience Cloudのゲストユーザー設定ミス、Snowflakeの**MFA未設定**が悪用された。

**2. サードパーティ／OAuthトークン窃取系**

- インフォスティーラー（Lumma Stealer等、ゲームチート偽装）で個人デバイスからOAuthトークンを窃取（Vercel）。
- 分析ベンダーAnodotの認証トークンを盗み、顧客のSnowflake/BigQueryへ横断アクセス（Vimeo、Zara、Rockstar）。

**3. ゼロデイ悪用系**

- **CVE-2026-35273**（Oracle PeopleSoft PeopleTools、認証欠如CWE-306、CVSS 9.8）。PSEMHUB/EMHubエンドポイントを突く未認証RCE。
- **MeshCentral**エージェントをAzureバイナリ(azurenetfiles.net等)に偽装しC2化、Python SimpleHTTPサーバ、SSHによる水平移動、クレデンシャルスプレー。

**恐喝の運用面**：pay or leak型。身代金拒否時は段階的にリーク。被害組織との「データ削除合意」（Instructure）や、公開期限のカウントダウン、リークサイトからの削除（Medtronic＝支払い示唆）など、心理的圧力を運用として体系化しています。被害規模について攻撃者と組織の主張が食い違う（McGraw-Hill、NAIC）のも常態です。

## 影響と被害範囲

- **被害人数**：単発でMedtronic 380万人、Carnival 600万人、Charter 約490万人、DentaQuest 最大2,340万人、Instructure/Canvas 2億7,500万人（攻撃者主張）など、桁違いの規模。
- **データ種別の深刻化**：初期はメール・氏名中心だったが、後半はSSN、銀行口座、税務情報、パスポート番号、健康・保険・診療情報、政府発行IDまで拡大。なりすまし・医療ID詐欺・金融詐欺の長期リスク。
- **セクター集中**：教育（Instructure、Infinite Campus、PeopleSoft悪用の大学群）とヘルスケア（Medtronic、DentaQuest、Abbott、Health-ISAC警告）。いずれも未成年・患者の機微情報を含む。
- **業務影響**：Canvasは期末試験期間に混乱。NAICは信用格付け機関へのデータ提供を一時停止。医療機器そのものの動作影響は各社とも否定。
- **二次・三次被害**：リークデータはマーケットプレイスで長期流通し、数年単位で悪用が続く。7/26のセクストーション詐欺はその典型で、初動対応で終わらないことを示す。

## 防御・推奨アクション

実務者が優先度順に打てる策を、記事のaction群から統合します。

**最優先（今日着手）**

1. **フィッシング耐性MFAへの移行**：SMS/プッシュ通知型ではなく**FIDO2/WebAuthn**を全ユーザー（特に管理者・SSO）に強制。MFA疲弊・ビッシングの主要経路を塞ぐ（Health-ISAC、SecurityWeek 06-23）。
2. **ヘルプデスクの本人確認強化**：MFAリセット/デバイス再登録時に**アウトオブバンド確認**を必須化。ビッシングの成立点を潰す。
3. **Oracle PeopleSoft対応**：CVE-2026-35273のパッチ／CISA緩和策を即適用。EMHub/PSEMHUBエンドポイントをインターネットから隔離。MeshCentral通信・Azure偽装アウトバウンド・SSH水平移動の痕跡を調査。

**高優先（今週）**

4. **Salesforce/SaaS設定監査**：Experience Cloudのゲストユーザー権限、公開ウェブフォーム、外部向けページのアクセス制御を点検。CSPM導入を検討。
5. **OAuthトークン・Connected Appsの棚卸し**：スコープ最小化、定期ローテーション、不要な連携の失効。Snowflake/BigQuery等DWHへの第三者アクセスを再確認しMFAを強制。
6. **サードパーティ／ベンダーリスク評価**：分析ベンダー等の連携トークンをローテーション。データ最小化と最小権限を適用。単一ベンダー集中リスクを評価。

**継続的**

7. **インフォスティーラー対策**：EDR導入、BYOD/個人デバイスからの企業SaaS接続ポリシー見直し、ゲームMOD/チート等のダウンロード禁止周知。
8. **認証異常の監視**：SSOの不審ログイン、異常なデータエクスポート、大量ダウンロードの検知ルール。ShinyHunters関連IoCをSIEMに登録。
9. **被害後対応の準備**：影響者通知、クレジット/ダークウェブモニタリング提供、そして**セクストーション等の二次詐欺への注意喚起**（支払わず報告）。

## 今後の見通し

- **IDベース攻撃はさらに主流化**する見込みです。パッチ管理が成熟する一方、SSO・OAuth・ヘルプデスクという「人とID」の接点が最も弱い環。FIDO2移行の遅れた組織が狙われ続けます。
- **サプライチェーン（ベンダー・SaaS連携）が引き続き最大の攻撃面**。1つのベンダートークンが数十社への合鍵になる構造は、EY(07-28)まで一貫しています。
- **教育・ヘルスケアへの圧力は継続**。レガシー資産と機微データの組み合わせが標的価値を高めます。PeopleSoftのような基幹システムの新たなゼロデイが出れば同様の大量キャンペーンが再発しうる。
- **リークデータの長期二次悪用**が新たな常態に。侵害から数ヶ月〜数年後の詐欺・恐喝を前提とした、被害者への継続的な注意喚起が必要です。
- なお、Scattered Spider関連メンバーの起訴・送還（07-03）など**法執行の進展**は見られますが、ShinyHuntersを含む緩やかなネットワークの活動が停止したという確証は報道時点でありません。

## 参照記事

- TeamPCP Supply Chain Campaign: Update 007 - Cisco Source Code Stolen via Trivy-Linked Breach — https://isc.sans.edu/diary/rss/32880
- Stolen Rockstar Games analytics data leaked by extortion gang — https://www.bleepingcomputer.com/news/security/stolen-rockstar-games-analytics-data-leaked-by-extortion-gang/
- McGraw-Hill confirms data breach following extortion threat — https://www.bleepingcomputer.com/news/security/mcgraw-hill-confirms-data-breach-following-extortion-threat/
- Data breach at edtech giant McGraw Hill affects 13.5 million accounts — https://www.bleepingcomputer.com/news/security/data-breach-at-edtech-giant-mcgraw-hill-affects-135-million-accounts/
- In Other News: Satellite Cybersecurity Act, $90K Chrome Flaw, Teen Hacker Arrested — https://www.securityweek.com/in-other-news-satellite-cybersecurity-act-90k-chrome-flaw-teen-hacker-arrested/
- Vercel社員のAIツールアクセスがデータ侵害に — https://www.darkreading.com/application-security/vercel-employees-ai-tool-access-data-breach
- ADT confirms data breach after ShinyHunters leak threat — https://www.bleepingcomputer.com/news/security/adt-confirms-data-breach-after-shinyhunters-leak-threat/
- Medtronic confirms breach after hackers claim 9 million records theft — https://www.bleepingcomputer.com/news/security/medtronic-confirms-breach-after-hackers-claim-9-million-records-theft/
- Home security giant ADT data breach affects 5.5 million people — https://www.bleepingcomputer.com/news/security/home-security-giant-adt-data-breach-affects-55-million-people/
- Vimeo Confirms User and Customer Data Breach — https://www.securityweek.com/vimeo-confirms-user-and-customer-data-breach/
- 医療機器サイバー攻撃、ヘルスケア組織の4分の1が被害報告 — https://www.infosecurity-magazine.com/news/quarter-healthcare-medical-device/
- EdTech企業Instructureがデータ侵害を公表 — https://www.securityweek.com/edtech-firm-instructure-discloses-data-breach/
- 教育テクノロジー大手 Instructure でデータ侵害 — https://techcrunch.com/2026/05/05/hackers-steal-students-data-during-breach-at-education-tech-giant-instructure/
- Vimeo data breach exposes personal information of 119,000 people — https://www.bleepingcomputer.com/news/security/vimeo-data-breach-exposes-personal-information-of-119-000-people/
- Instructure Breach Exposes Schools' Vendor Dependence — https://www.darkreading.com/cyberattacks-data-breaches/instructure-breach-exposes-schools-vendor-dependence
- ShinyHunters claims nearly 9,000 schools affected by Canvas data breach — https://edscoop.com/shinyhunters-claims-nearly-9000-schools-affected-by-canvas-data-breach/
- NVIDIA confirms GeForce NOW data breach affecting Armenian users — https://www.bleepingcomputer.com/news/security/nvidia-confirms-geforce-now-data-breach-affecting-armenian-users/
- Zaraデータ侵害、約20万人の顧客に影響 — https://www.infosecurity-magazine.com/news/zara-data-breach-impacts-200000/
- ShinyHuntersがCanvasへの恐喝キャンペーンを拡大 — https://www.infosecurity-magazine.com/news/shinyhunters-escalates-canvas/
- Deal Reached With Hackers to Delete Data Stolen From the Canvas Educational Platform — https://www.securityweek.com/deal-reached-with-hackers-to-delete-data-stolen-from-the-canvas-educational-platform/
- Canvas Owner Reaches Agreement With Cybercriminals After Ransomware Attack — https://www.infosecurity-magazine.com/news/canvas-cybercriminals-agreement/
- 7-Eleven Data Breach Confirmed After ShinyHunters Ransom Demand — https://www.securityweek.com/7-eleven-data-breach-confirmed-after-shinyhunters-ransom-demand/
- Grafana Confirms Breach After Hackers Claim They Stole Data — https://www.securityweek.com/grafana-confirms-breach-after-hackers-claim-they-stole-data/
- 7-Eleven confirms data breach claimed by the ShinyHunters gang — https://www.bleepingcomputer.com/news/security/7-eleven-confirms-data-breach-claimed-by-the-shinyhunters-gang/
- 185,000 Likely Impacted by 7-Eleven Data Breach — https://www.securityweek.com/185000-likely-impacted-by-7-eleven-data-breach/
- Carnival Data Breach Exposed 6 Million People — https://www.securityweek.com/carnival-data-breach-exposed-6-million-people/
- Charter Communications Data Breach Could Impact Nearly 5 Million — https://www.securityweek.com/charter-communications-data-breach-could-impact-near-5-million/
- DentaQuest data breach exposed info of 2.6 million accounts — https://www.bleepingcomputer.com/news/security/dentaquest-data-breach-exposed-info-of-26-million-accounts/
- Oracle PeopleSoft servers hacked in ShinyHunters data theft attacks — https://www.bleepingcomputer.com/news/security/oracle-peoplesoft-servers-hacked-in-shinyhunters-data-theft-attacks/
- ShinyHunters Exploits Oracle PeopleSoft Zero-Day (CVE-2026-35273) to Breach Universities — https://thehackernews.com/2026/06/shinyhunters-exploits-oracle-peoplesoft.html
- Nottingham University Data Breach Affects Over 450,000 Students — https://www.bleepingcomputer.com/news/security/nottingham-university-data-breach-affects-over-450-000-students/
- CVE-2026-35273: Oracle PeopleSoft Enterprise PeopleTools (CISA KEV) — https://www.cisa.gov/known-exploited-vulnerabilities-catalog?search_api_fulltext=CVE-2026-35273
- ShinyHunters is actively extorting universities after exploiting an unpatched Oracle flaw — https://cyberscoop.com/oracle-peoplesoft-zero-day-vulnerability-shinyhunters-extortion/
- ShinyHunters Uses Oracle Zero-Day to Rampage Higher Ed — https://www.darkreading.com/vulnerabilities-threats/shinyhunters-oracle-zero-day-higher-ed
- Council of Europe investigates ShinyHunters data breach claims — https://www.bleepingcomputer.com/news/security/council-of-europe-investigates-shinyhunters-data-breach-claims/
- Infinite Campus data breach affects 137,000 school staff accounts — https://www.bleepingcomputer.com/news/security/infinite-campus-data-breach-affects-137-000-school-staff-accounts/
- Kodak confirms data breach claimed by ShinyHunters extortion gang — https://www.bleepingcomputer.com/news/security/kodak-confirms-data-breach-claimed-by-shinyhunters-extortion-gang/
- Kodak Admits Data Breach After ShinyHunters Hack Claims — https://www.securityweek.com/kodak-admits-data-breach-after-shinyhunters-hack-claims/
- What the Latest ShinyHunters Breaches Reveal About Modern Cyberattacks — https://www.securityweek.com/what-the-latest-shinyhunters-breaches-reveal-about-modern-cyberattacks/
- CMC Releases Analysis and Guidance for Education Sector After Canvas Data Breach — https://www.infosecurity-magazine.com/news/cmc-analysis-education-canvas-data/
- Nissan discloses employee data breach linked to Oracle zero-day attacks — https://www.bleepingcomputer.com/news/security/nissan-discloses-employee-data-breach-linked-to-oracle-zero-day-attacks/
- NAIC says public data stolen in ShinyHunters' PeopleSoft breach — https://www.bleepingcomputer.com/news/security/naic-says-public-data-stolen-in-shinyhunters-peoplesoft-breach/
- Nissan Discloses Employee Data Breach Linked to Oracle Zero-Day — https://www.infosecurity-magazine.com/news/nissan-oracle-peoplesoft-zero-day/
- Medtronic notifies customers impacted by ShinyHunters data breach — https://www.bleepingcomputer.com/news/security/medtronic-notifies-customers-impacted-by-shinyhunters-data-breach/
- Alleged Scattered Spider Member Extradited to US — https://www.infosecurity-magazine.com/news/scattered-spider-member-extradited/
- Medtronic Data Breach Impacts 3.8 Million People — https://www.securityweek.com/medtronic-data-breach-impacts-3-8-million-people/
- Shiny Hunters Impact to Health Sector and Recommended Mitigation Strategies — https://health-isac.org/shiny-hunters-impact-to-health-sector-and-recommended-mitigation-strategies/
- In Other News: Dolphin X AI-Powered Malware, Car Anti-Theft Device Hack, 400 Linux Kernel Flaws — https://www.securityweek.com/in-other-news-dolphin-x-ai-powered-malware-car-anti-theft-device-hack-400-linux-kernel-flaws/
- ShinyHunters data leaks fuel $2,000 sextortion email scam — https://www.bleepingcomputer.com/news/security/shinyhunters-data-leaks-fuel-2-000-sextortion-email-scam/
- DentaQuest Data Breach Potentially Impacts Over 23 Million People — https://www.securityweek.com/dentaquest-data-breach-potentially-impacts-over-23-million-people/
- Ernst & Young data breach claimed by ShinyHunters extortion gang — https://www.bleepingcomputer.com/news/security/ernst-and-young-data-breach-claimed-by-shinyhunters-extortion-gang/
