---
title: "Microsoft 365 — 「MFAを入れたから安全」が終わった年：PhaaS産業化と正規クラウドのC2化"
date: 2026-09-15T07:00:00+09:00
tags: ["security", "intelligence", "深掘り", "Microsoft 365"]
draft: false
---

## 30秒サマリ

- 2026年4月〜9月の約5か月で、Microsoft 365 を標的とする**MFAバイパス型フィッシングのサービス化（PhaaS）**が連続して報じられた。W3LL、Tycoon2FA、ARToken/EvilTokens、Forg365、Kratos、BigBear 2.0 と、解体されても後継が即座に埋める構図が見える。
- 攻撃の主眼は「パスワードを盗む」から「**セッション/トークンを盗む**」へ完全に移行。AitM、デバイスコードフィッシング、OAuth同意詐取（ConsentFix/ClickFix）が主要な突破口になっている。
- 同時進行で、**Microsoft Graph API 自体がC2インフラとして悪用**されている（Webworm の GraphWorm、Group-IB が報告した HollowGraph のカレンダーC2）。攻撃者は自前サーバを持たず、Microsoft の信頼されたインフラに寄生する。
- 侵入経路も多様化。公衆Wi-Fiゲートウェイ、BYOD端末、そして **IT部門を騙るビッシングで経営層を直接狙う**手口（2026年9月）まで確認された。
- 実務上の結論はシンプル。**フィッシング耐性MFA（FIDO2/パスキー）への移行、デバイスコードフローの制限、OAuth同意の管理者承認化、メールボックスルールとGraph APIアクセスの継続監査**——この4点が今期の最優先事項。

---

## 背景 — 何者か / 何が起きているか

このテーマは単一の脅威アクターの話ではありません。**「Microsoft 365 という標的面（attack surface）に、複数の独立した経済圏と国家系アクターが同時に集まっている」**という構造の話です。

観測された動きは、大きく3つのレイヤーに分かれます。

**レイヤー1：PhaaS（フィッシング・アズ・ア・サービス）の産業化**
W3LL、Tycoon2FA、EvilTokens/ARToken、Forg365、Kratos、BigBear 2.0。いずれも Microsoft 365 の認証情報・セッション窃取に特化したツールキットで、サブスクリプション型で販売されています。Forg365 は Telegram 経由で月額400ドル／年額3,800ドルという価格が報じられました。つまり、**高度なMFAバイパス能力が「月4万円台の経費」として流通している**わけです。攻撃者側に必要なのは技術力ではなく、クレジットカードとターゲットリストだけになりました。

**レイヤー2：正規クラウドサービスのC2転用**
中国系アクター Webworm は Discord と Microsoft Graph API をC2に使う EchoCreep / GraphWorm を展開。Group-IB が特定した HollowGraph は、乗っ取った Microsoft 365 のカレンダー予定を暗号化された双方向C2チャネルとして使います。**攻撃者が自前のサーバを一切持たない**という点が決定的で、IPレピュテーションやドメインブロックといった従来型の防御が原理的に効きません。

**レイヤー3：人間とデバイスの隙間を突く初期侵入**
公衆Wi-Fiゲートウェイの侵害による出張者の認証情報収集（2026年7月）、BYOD経由のビッシング（2026年9月）、IT部門を騙る電話で幹部を狙う手口（2026年9月）。技術的防御を迂回して、**人とデバイス管理の穴から入る**方向へ回帰しています。

重要なのは、これら3レイヤーが独立ではなく連結していることです。PhaaS で取ったトークンが、Graph API 経由の内部偵察に使われ、その足場が ShinyHunters のような恐喝グループへ引き渡される——2026年9月11日の報道が示したのは、まさにこの**分業サプライチェーン**の完成形でした。

---

## 時系列 — インシデントの連鎖

### 第1期：基盤の露出（2026年4月）

**2026-04-14｜FBIがPhaaS「W3LL」を解体、被害総額2,000万ドル超**（Infosecurity Magazine）
FBIとインドネシア当局が W3LL を解体。MFAバイパス機能を持つツールキットを提供し、Microsoft 365 を標的とした BEC キャンペーンに使われていました。**世界17,000人以上の被害者、25,000以上の侵害アカウントが取引**され、中心的開発者「G.L.」が特定、w3ll.store ドメインが押収されています。

**2026-04-14｜メールボックスルール悪用が侵害後の隠密持続手法として台頭**（Infosecurity Magazine）
同じ日に、研究者が Microsoft 365 のメールボックスルール悪用を警告。アクティビティの隠蔽、データ窃取、侵害後のアクセス維持に使われています。

> **この2本が同日に出たことの意味**：片方は「どう入るか」（W3LL）、もう片方は「入った後どう居座るか」（メールボックスルール）。BECのキルチェーンが前半と後半で同時に可視化された日でした。そして重要なのは、**W3LL が潰れても後半の手口は無傷で残る**という点です。以降の記事が示すのは、まさにその「後半だけ残って、前半が次々に入れ替わる」パターンです。

### 第2期：MFAバイパス手法の多様化（2026年5月〜7月）

**2026-05-18｜Tycoon2FA がデバイスコードフィッシングでM365アカウントを乗っ取り**（BleepingComputer）
Tycoon2FA がデバイスコードフィッシングに対応。Trustifi のクリックトラッキングリンクを悪用してセッショントークンを窃取します。**正規のメールセキュリティ製品のトラッキングリンクを踏み台にする**点が巧妙で、URL評価による検知をすり抜けます。

**2026-05-21｜Webworm が Discord と MS Graph API でバックドアを展開**（The Hacker News）
ここで脅威の性質が変わります。中国系 Webworm（少なくとも2022年から政府機関を標的）が、C2 に Discord と Microsoft Graph API を使う EchoCreep / GraphWorm を展開。**Microsoft 365 が「盗まれる対象」から「攻撃インフラそのもの」になった**転換点です。

**2026-07-03｜ConsentFix と ClickFix：3秒でM365アカウントが乗っ取られる**（BleepingComputer）
偽プロンプトと OAuth フローを悪用し、数秒でトークンを窃取。**ユーザーは正規の Microsoft 同意画面で「許可」を押すだけ**。これはフィッシングというより、正規の認可フローを悪用した設計上の穴の突き方です。MFAは一切関係なく迂回されます。

**2026-07-04｜ARToken PhaaS が EvilTokens のツールキットを露出**（BleepingComputer）
新たな PhaaS「ARToken」が EvilTokens のアフィリエイトとして活動。研究者がツールキット内部を分析する機会を得たと報じられました。

**2026-07-09｜「ゴーストフィッシング」が従来型メールセキュリティを破る**（The Hacker News）
EvilTokens キャンペーンで、米欧企業を標的にした新手法を確認。**悪意あるページは被害者のブラウザ内で復号されるまで非表示**に保たれ、従来のURL検査では検知不能。ゲートウェイでのスキャン時点では「無害なページ」にしか見えません。

**2026-07-14｜Forg365 PhaaS がデバイスコード＋AitM でM365を標的**（The Hacker News）
Forg365 が登場。デバイスコードフィッシング、AitM、**AI支援による誘導文作成**、侵害後のメールボックス操作を統合。Telegram 経由で月額400ドル／年額3,800ドル。

> **4月から7月への流れを読む**：4月に別々に報じられた「侵入」と「持続化」が、7月の Forg365 では**一つの製品に統合**されています。さらに AI による文面生成まで同梱。PhaaS が単なるフィッシングページ配布から、**キルチェーン全体をカバーするプラットフォーム**へ進化したことが読み取れます。

**2026-07-21｜HollowGraph が M365 カレンダーを秘密C2に**（Infosecurity Magazine）
Group-IB が特定した Windows マルウェア。Graph API を悪用し、乗っ取った Microsoft 365 カレンダーを**暗号化された双方向C2チャネル**として使用。イスラエル関連組織を標的に、**2026年6月3日〜7月9日で12件の感染**を確認。Cavern バックドアフレームワークとの関連が高く、イラン関連アクター Lyceum との技術的類似性が指摘されています。DNSトンネリングで Entra ID 認証情報を取得・更新し、RSA+AES-256-GCM のハイブリッド暗号化を採用。**攻撃者は自前サーバを一切使わず、Microsoft の信頼されたインフラに完全依存**します。

**2026-07-23｜Kratos フィッシングキットのインフラを法執行機関が解体**（The Hacker News）
ドイツと米国の法執行機関が「世界で最も広く使用されている犯罪用フィッシングキット」Kratos のコアインフラを解体。インドネシア当局が開発者・運営者を逮捕しました。M365 セッション窃取と MFA バイパス機能を備えていました。

> **4月の W3LL、7月の Kratos——いずれもインドネシア当局が関与**しています。PhaaS の運営拠点が特定地域に集中していることを示唆する重要な符合です（ただし両者の関係性については報道時点で明示されていません）。

**2026-07-28｜侵害された公衆Wi-Fiゲートウェイで企業認証情報を収集**（SecurityWeek）
攻撃者が侵害した公衆Wi-Fiゲートウェイ機器で、**出張中の従業員の Microsoft 365 認証情報を収集**していることが判明。

### 第3期：標的の選別化と分業（2026年9月）

**2026-09-08｜IT部門を騙る偽電話で幹部を狙うM365データ窃取・恐喝**（The Hacker News）
IT部門を装うビッシング、AitM によるトークン窃取、**住宅用プロキシ経由のログイン**を組み合わせたキャンペーン。標的は主に**取締役・VPクラスの幹部**。住宅用プロキシの使用は、「不審なIPからのサインイン」という検知ロジックを無力化します。

**2026-09-08 / 09-09｜BigBear 2.0 が258組織でMFAを突破、5,000件超の認証情報を窃取**（BleepingComputer / Infosecurity Magazine）
CloudSEK の調査により判明。**258組織でMFAを突破し、5,000件超の Microsoft 365 認証情報を窃取**。Kratos 解体からわずか1か月半での大規模被害の表面化です。

**2026-09-11｜ビッシングがBYODを突いてM365・企業データへ到達**（Dark Reading）
脅威アクターが **Graph API を悪用して価値の高い標的を特定**し、その侵入アクセスを **ShinyHunters などの恐喝グループへ引き渡す**手口を確認。

> **9月の3本が示す到達点**：①標的が無差別から**幹部への選別**へ、②検知回避が**住宅用プロキシ＋BYOD**という「正常に見える経路」へ、③そして**初期侵入役と恐喝役の分業**が確立。5月時点では「MFAを破る技術の話」だったものが、9月には「**誰から奪い、誰に売るか**というビジネスモデルの話」に変わっています。

---

## 手口・技術詳細 (TTP)

### 1. トークン／セッション窃取系（MFAバイパスの本命）

**AitM（Adversary-in-the-Middle）**
被害者と Microsoft の間にリバースプロキシを挟み、認証フロー全体を中継。ユーザーが正規にMFAを完了した結果として発行される**セッションクッキー／トークンをそのまま奪取**します。パスワードもMFAコードも「正しく」入力されているため、Entra ID 側からは正常なサインインに見えます。Tycoon2FA、Forg365、BigBear 2.0、9月のビッシングキャンペーンすべてで採用。

**デバイスコードフィッシング**
Microsoft の正規デバイスコードフロー（テレビやCLIなど入力困難な端末向けの認証）を悪用。攻撃者が生成したコードを「IT部門からの依頼」等の文脈でユーザーに入力させると、**攻撃者側の端末にトークンが発行**されます。フィッシングサイトが一切不要で、ユーザーは本物の microsoft.com で認証しているため、URL の真偽判定がまったく役に立ちません。Tycoon2FA（5月）、Forg365（7月）で確認。

**OAuth同意フィッシング（ConsentFix / ClickFix）**
偽プロンプトから正規の OAuth 同意画面へ誘導し、悪意あるアプリに権限を付与させる手口。**3秒で完了**すると報じられています。同意された時点で攻撃者のアプリが正当なトークンを保持するため、パスワードリセットやMFA再登録では追い出せません。**アプリの同意を取り消すまで、アクセスは生き続けます**。

### 2. 検知回避系

**ゴーストフィッシング**：悪意あるページのコンテンツを暗号化しておき、**被害者のブラウザ内で復号されるまで非表示**に保つ。メールゲートウェイがURLを検査する時点では無害なページしか見えません（EvilTokens、7月）。

**正規サービスの踏み台化**：Trustifi のクリックトラッキングリンクを経由させることで、URLレピュテーションを回避（Tycoon2FA、5月）。

**住宅用プロキシ**：企業の一般ユーザーと区別がつかないIPからログインし、地理的異常検知をすり抜ける（9月）。

**AI支援による誘導文作成**：Forg365 が同梱。定型文パターンによる検知ルールが効かなくなります。

### 3. C2インフラとしての Microsoft 365

**GraphWorm / EchoCreep**（Webworm、5月）：Microsoft Graph API と Discord をC2に使用。

**HollowGraph**（7月）：最も洗練された事例です。

- 乗っ取った M365 **カレンダー予定**を双方向C2チャネルに使用
- **RSA + AES-256-GCM** のハイブリッド暗号化
- **DNSトンネリング**で Entra ID 認証情報を取得・更新
- **自前C2サーバを一切持たない**（Microsoft インフラに完全依存）
- Cavern バックドアフレームワークと関連、イラン関連 Lyceum と技術的類似性
- 標的はイスラエル関連組織、2026/6/3〜7/9 で12件の感染

企業ネットワークで graph.microsoft.com への通信を遮断するのは現実的に不可能です。**信頼済みドメインへの正当な通信として、防御網を素通りする**のがこの手法の本質です。

### 4. 侵害後の持続化

**メールボックスルール悪用**（4月）：受信メールの自動転送、特定キーワードを含むメールの自動削除（被害者への警告メールを隠す）、外部への転送設定。SOC の監視対象から外れやすく、長期間気づかれません。Forg365 は「侵害後のメールボックス操作」を機能として統合済みです。

### 5. 初期侵入経路

公衆Wi-Fiゲートウェイの侵害（7月）、BYOD端末（9月）、IT部門を騙るビッシング（9月）。いずれも**企業が完全に管理できない領域**を狙っています。

---

## 影響と被害範囲

**数値で確認できるもの（報道ベース）**

- W3LL：世界17,000人以上の被害者、25,000以上の侵害アカウントが取引、被害総額2,000万ドル超
- BigBear 2.0：258組織でMFA突破、5,000件超の M365 認証情報を窃取
- HollowGraph：2026/6/3〜7/9 に12件の感染（イスラエル関連組織）
- Kratos：「世界で最も広く使用されている犯罪用フィッシングキット」と評されるが、具体的な被害組織数は報道時点で未提示

**地理・業種**

- EvilTokens／ゴーストフィッシング：米国・欧州企業
- HollowGraph：イスラエル関連組織
- Webworm：政府機関および関連取引事業者
- PhaaS全般（W3LL、Forg365、BigBear 2.0）：**業種・地域を問わず Microsoft 365 利用組織全般**

**役職による偏り**
2026年9月のキャンペーンは**取締役・VPクラスを明示的に標的**としています。幹部アカウントは M&A 情報、財務データ、人事情報へのアクセス権を持ち、かつセキュリティ例外が適用されがちという二重のリスクを抱えます。

**被害の連鎖**
認証情報窃取 → BEC（不正送金）／データ窃取 → **恐喝**。2026年9月11日の報道が示したように、初期侵入者は自分で換金せず、ShinyHunters のような恐喝グループへアクセスを売却します。つまり**「アカウント1つの侵害」が「全社データの公開脅迫」に直結する**経路が確立しています。

**注意すべき不確実性**

- 各 PhaaS 間の運営者レベルでの関係性は、報道時点では明示されていません（W3LL と Kratos がともにインドネシアで摘発された点は符合しますが、同一組織かは未確認）
- HollowGraph と Lyceum の関係は「技術的類似性の指摘」であり、帰属が確定したものではありません
- BigBear 2.0 の被害258組織の具体名は報道時点で公表されていません

---

## 防御・推奨アクション

### 優先度【最高】今週中に着手すべきもの

**1. デバイスコードフローの制限（条件付きアクセス）**
Entra ID の条件付きアクセスで認証フローポリシーを作成し、**デバイスコードフローをブロック**します。業務上必要な例外（会議室デバイス等）だけを限定的に許可。Tycoon2FA と Forg365 の主要突破口を1設定で塞げるため、費用対効果が最も高い施策です。

**2. OAuth アプリの管理者同意ワークフロー化**
ユーザーによるアプリ同意を既定で無効化し、**管理者承認フローを必須**に。ConsentFix/ClickFix は「ユーザーが自分で同意できる」前提に依存しているため、この設定で無効化できます。併せて、既に同意済みのエンタープライズアプリを棚卸しし、不審なアプリの同意を取り消してください（パスワードリセットでは追い出せない点に注意）。

**3. メールボックスルールの緊急監査**
全ユーザーの受信トレイルールを棚卸しし、以下を洗い出します。

- 外部ドメインへの自動転送
- 「invoice」「payment」「security alert」等のキーワードでの自動削除・アーカイブ
- RSS フォルダ等の見落としがちな場所への移動ルール

同時に、**新規フォワーディングルール作成をアラート化**（Microsoft Purview の監査ログ／Defender for Office 365 のアラートポリシー）。

### 優先度【高】今四半期中に

**4. フィッシング耐性MFA（FIDO2／パスキー）への移行**
AitM に対する**唯一の構造的な対策**です。FIDO2 はオリジンバインディングにより、リバースプロキシ経由の認証が原理的に成立しません。全社一斉は困難なので、**幹部・IT管理者・財務担当から優先展開**してください。9月のキャンペーンが幹部を狙っていることを踏まえると、この順序には明確な根拠があります。

**5. サインインログの遡及調査**

- w3ll.store 等、報じられた IOC への過去通信ログの突合
- 住宅用プロキシ経由と思われる異常サインイン（通常と異なるASN、短時間での地理的移動）
- 同一アカウントの複数デバイスからの同時セッション
- EchoCreep / GraphWorm / HollowGraph の IOC を脅威インテリフィードと突合

**6. Graph API アクセスの監視**

- 異常な Graph API 呼び出しパターン（大量のメールボックス列挙、ディレクトリ全体のユーザー情報取得）
- テナント外からの Graph API アクセス
- **カレンダー予定の異常な作成・更新**（HollowGraph 対策として特に重要）
- Graph 権限を持つサービスプリンシパルの棚卸し

**7. トークン保護とセッション寿命の見直し**
条件付きアクセスでのサインイン頻度制御、継続的アクセス評価（CAE）の有効化、および準拠デバイスからのアクセス要求。トークンを盗まれても**デバイス条件で弾く**多層構成にします。

### 優先度【中】体制・プロセス

**8. ヘルプデスクの本人確認プロセス強化**
2026年9月の攻撃はここを突いています。MFAリセット・パスワードリセット依頼に対して、**電話だけで完結させない**（マネージャー承認、既登録デバイスへのコールバック、ビデオ通話での本人確認）。逆に「IT部門から電話でコードを聞かれることは絶対にない」という周知も必須です。

**9. BYOD の条件付きアクセス強化**
私用デバイスからの M365 アクセスに対し、アプリ保護ポリシー（Intune MAM）の適用、ダウンロード制限付きのブラウザセッション、準拠デバイス要求のいずれかを適用します。

**10. 出張・公衆Wi-Fi 対策**
常時接続VPNまたは SSE/ZTNA の適用を徹底。「公衆Wi-Fi は侵害済みの前提で扱う」という運用に切り替えます。

**11. メールセキュリティの多層化**
ゴーストフィッシングは**ゲートウェイでのURL静的検査を原理的に回避**します。開封時の動的分析（time-of-click 保護）、ブラウザ内での振る舞い検知、そして**ユーザーによる報告体制**の強化を組み合わせてください。この脅威に対しては、現実的にユーザー報告が最後の砦になります。

### ユーザー教育で伝えるべき3点

1. **予期しない同意画面では絶対に「許可」を押さない**（3秒で乗っ取られる）
2. **誰かに言われてデバイスコードを入力することは、絶対にない**
3. **IT部門を名乗る電話でコードや認証操作を求められたら、必ず切って自分から折り返す**

---

## 今後の見通し

**PhaaS の「潰しても湧く」構図は続く**
4月に W3LL、7月に Kratos が解体されましたが、その間に ARToken、Forg365 が登場し、9月には BigBear 2.0 の大規模被害が表面化しました。**摘発から次の大型キャンペーン表面化までのサイクルが1〜2か月に短縮**しています。テイクダウンは有効ですが、それを恒久的な安心材料として扱うことはできません。

**MFAバイパスは「前提」になる**
BigBear 2.0 が258組織でMFAを突破した事実が示すのは、「MFA導入済み」がもはや防御の証明にならないということです。今後の議論は**MFAの有無ではなく、MFAの種類（フィッシング耐性があるか）**に移っていきます。パスワードレス／FIDO2 への移行速度が、そのまま組織のリスク差になります。

**正規クラウドのC2化はさらに広がる可能性**
Webworm（5月）と HollowGraph（7月）が示した Graph API／カレンダーC2 は、**IPブロックもドメインブロックも効かない**という点で防御側に構造的な不利を強います。手法が公開された以上、他アクターへの拡散が想定されます。検知の軸は「どこと通信したか」から「**何を、どのくらい、いつ**呼び出したか」という振る舞いベースへ移行せざるを得ません。

**分業モデルの定着**
2026年9月11日の報道が示した「初期アクセス獲得役 → 恐喝グループ」の受け渡し構造は、ランサムウェアで確立した IAB（Initial Access Broker）モデルの SaaS 版です。これが定着すると、**M365 アカウント1つの侵害が、数週間後に全社データの公開脅迫として返ってくる**シナリオが常態化します。初動対応の速度が、そのまま被害規模を決めます。

**狙われる層の移動**
無差別大量フィッシングから、幹部・高権限アカウントへの選別的攻撃へシフトしています。ビッシングと AitM の組み合わせは手間がかかる分、**リターンの大きい標的にしか使われません**。逆に言えば、幹部層の防御を固めることの投資対効果が上がっているということです。

**注記**：本稿は2026年4月14日〜9月11日の報道に基づきます。各 PhaaS 間の運営者の関係、HollowGraph の帰属、BigBear 2.0 の被害組織の具体名などは、報道時点では未確認または未公表です。続報による更新を前提にお読みください。

---

## 参照記事

| 日付 | タイトル | ソース |
|---|---|---|
| 2026-04-14 | [FBIがフィッシング・アズ・ア・サービス「W3LL」を解体、被害総額2,000万ドル超](https://www.infosecurity-magazine.com/news/fbi-dismantles-phishing-operation/) | Infosecurity Magazine |
| 2026-04-14 | [メールボックスルール悪用が侵害後の隠密持続手法として台頭](https://www.infosecurity-magazine.com/news/mailbox-rule-abuse-stealthy-post/) | Infosecurity Magazine |
| 2026-05-18 | [Tycoon2FA hijacks Microsoft 365 accounts via device-code phishing](https://www.bleepingcomputer.com/news/security/tycoon2fa-hijacks-microsoft-365-accounts-via-device-code-phishing/) | BleepingComputer |
| 2026-05-21 | [Webworm Deploys EchoCreep and GraphWorm Backdoors Using Discord and MS Graph API](https://thehackernews.com/2026/05/webworm-deploys-echocreep-and-graphworm.html) | The Hacker News |
| 2026-07-03 | [ConsentFix and ClickFix: How Microsoft 365 Accounts are Hijacked in 3 Seconds](https://www.bleepingcomputer.com/news/security/consentfix-and-clickfix-how-microsoft-365-accounts-are-hijacked-in-3-seconds/) | BleepingComputer |
| 2026-07-04 | [ARToken PhaaS exposes EvilTokens' Microsoft 365 phishing toolkit](https://www.bleepingcomputer.com/news/security/artoken-phaas-exposes-eviltokens-microsoft-365-phishing-toolkit/) | BleepingComputer |
| 2026-07-09 | [New Ghost Phishing Wave Is Breaking Traditional Email Security](https://thehackernews.com/2026/07/new-ghost-phishing-wave-is-breaking.html) | The Hacker News |
| 2026-07-14 | [Forg365 PhaaS Targets Microsoft 365 with Device Code and AitM Session Theft](https://thehackernews.com/2026/07/forg365-phaas-targets-microsoft-365.html) | The Hacker News |
| 2026-07-21 | [New HollowGraph Malware Hijacks Microsoft 365 Calendars for Covert C2 Communications](https://www.infosecurity-magazine.com/news/hollowgraph-microsoft-calendars/) | Infosecurity Magazine |
| 2026-07-23 | [Police Dismantle Kratos Phishing Kit Built to Steal Microsoft 365 Sessions and Bypass MFA](https://thehackernews.com/2026/07/police-dismantle-kratos-phishing-kit.html) | The Hacker News |
| 2026-07-28 | [Hacked Public Wi-Fi Gateways Used to Harvest Corporate Credentials](https://www.securityweek.com/hacked-public-wi-fi-gateways-used-to-harvest-corporate-credentials/) | SecurityWeek |
| 2026-09-08 | [Fake IT Calls Target Executives in Microsoft 365 Data Theft and Extortion Attacks](https://thehackernews.com/2026/09/microsoft-365-attackers-use-help-desk.html) | The Hacker News |
| 2026-09-08 | [BigBear Microsoft 365 phishing service bypassed MFA at 258 organizations](https://www.bleepingcomputer.com/news/security/bigbear-microsoft-365-phishing-service-bypassed-mfa-at-258-organizations/) | BleepingComputer |
| 2026-09-09 | [BigBear 2 PhaaS Campaign Steals 5000+ Microsoft Credentials](https://www.infosecurity-magazine.com/news/bigbear-2-phaas-5000-microsoft/) | Infosecurity Magazine |
| 2026-09-11 | [Voice Callers Exploit BYOD to Reach Microsoft 365, Corporate Data](https://www.darkreading.com/threat-intelligence/voice-callers-exploit-byod-microsoft-365-corporate-data) | Dark Reading |
