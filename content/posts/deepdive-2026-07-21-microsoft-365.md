---
title: "Microsoft 365 — 「攻撃者の常設インフラ」と化したクラウド、その3か月の攻防"
date: 2026-07-21T07:00:00+09:00
tags: ["security", "intelligence", "深掘り", "Microsoft 365"]
draft: false
---

## 30秒サマリ

- 2026年4〜7月、Microsoft 365 を狙う脅威は「認証情報を盗む入口」と「正規サービスをC2に転用する出口」の二方向で同時進行した。
- 入口側では PhaaS(フィッシング・アズ・ア・サービス)が主戦場化。W3LL 解体後も EvilTokens/ARToken、Forg365 と次々に後継が登場し、MFA を迂回するデバイスコード/AitM/OAuth同意攻撃が標準装備になった。
- 出口側では Webworm(GraphWorm)、そして HollowGraph が Microsoft Graph API やカレンダーを悪用し、攻撃者が「自前サーバーを持たない」C2 を実現した。
- 共通する本質は「Microsoft の信頼されたインフラそのものが攻撃面になった」こと。従来の URL 検査・IP/ドメイン検知は構造的に無力化しつつある。
- 実務対応の軸は3つ:フィッシング耐性のある認証(FIDO2/パスキー)、デバイスコード/OAuth同意フローの制限、そして Graph API・メールボックスルール・カレンダーへの振る舞い監視。

## 背景 — 何者か / 何が起きているか

Microsoft 365 は、メール・ファイル・Teams・ID(Entra ID)を束ねる、多くの組織の「業務の中枢」です。だからこそ攻撃者にとっての価値も高く、この観測期間(2026年4月14日〜7月21日)に報じられた事案を並べると、狙われ方が明確に二つの層に分かれていることが見えてきます。

第一の層は「入口=認証情報とセッションの窃取」です。特徴的なのは、これが個人の職人技ではなく **PhaaS(フィッシング・アズ・ア・サービス)というビジネスモデル** に乗って提供されている点です。ツールキットが月額課金で売られ、技術力の低い攻撃者でも MFA バイパスを含む高度な攻撃を実行できます。

第二の層は「出口=侵害後の隠密な通信と持続」です。攻撃者は自前のサーバーやドメインを用意する代わりに、Microsoft Graph API・カレンダー・メールボックスルールといった **正規の M365 機能そのもの** を C2(指令)チャネルや持続化の道具に転用しています。

この二層が同じ「Microsoft の信頼されたインフラ」を土台にしている、というのがこの3か月の物語の核心です。

## 時系列 — インシデントの連鎖

**2026-04-14 — 起点:W3LL 解体と、メールボックスルール悪用の警告(同日)**
FBI とインドネシア当局が PhaaS エコシステム「W3LL」を解体しました。MFA バイパス機能を備えたツールキットが M365 アカウントを狙う BEC(ビジネスメール詐欺)に使われ、被害総額は2,000万ドル超、世界17,000人以上の被害者・25,000超の侵害アカウントが取引されていたとされます。中心開発者「G.L.」が特定され、w3ll.store ドメインが押収されました。
同じ日に「メールボックスルール悪用が侵害後の隠密持続手法として台頭」との警告も出ています。この2本が同日に並んだことは象徴的です。**入口(PhaaS でアカウントを奪う)と出口(奪った後にルールで隠れて居座る)** という、その後3か月続くテーマがここで出そろいました。

**2026-05-18 — Tycoon2FA、デバイスコードフィッシングに対応**
フィッシングキット「Tycoon2FA」がデバイスコードフィッシングに対応し、Trustifi のクリックトラッキングリンクを悪用して M365 セッショントークンを窃取する手口が確認されました。W3LL が消えても市場は空白にならず、別のキットが MFA バイパス手法を進化させて穴を埋めた——PhaaS 市場の回復力(レジリエンス)を示す一手です。

**2026-05-21 — Webworm、Graph API を C2 に転用(出口の高度化)**
中国系アクター Webworm が、独自バックドア EchoCreep と GraphWorm を展開。C2 通信に Discord と **Microsoft Graph API** を悪用していることが報告されました。少なくとも2022年から政府機関を標的にしてきたアクターです。ここで「出口」の層が一段進みます。正規 SaaS を C2 に使えば、IP/ドメインベースの検知は無効化され、企業環境では Graph API を遮断しづらい——後の HollowGraph へとつながる系譜の起点です。

**2026-07-03 — ConsentFix / ClickFix、3秒で乗っ取り(入口の手口拡張)**
偽プロンプトと OAuth フローを悪用し、数秒で M365 トークンを盗む手法。ユーザーが正規そっくりの同意画面で「許可」を押すだけでトークンが攻撃者に渡り、MFA が実質的に迂回されます。フィッシングの狙いが「パスワード窃取」から「同意そのものの詐取」へ移った局面です。

**2026-07-04 — ARToken が EvilTokens ツールキットを露出**
新興 PhaaS「ARToken」が EvilTokens ツールキットのアフィリエイトとして活動していることが判明。研究者はこれを通じて M365 認証情報を狙うツールキット内部を分析する機会を得ました。W3LL が「解体される側」だったのに対し、EvilTokens 系は「拡大している側」として登場します。

**2026-07-09 — ゴーストフィッシング、従来のメール防御を突破**
米欧企業を標的にした EvilTokens キャンペーンで「ゴーストフィッシング」が確認されました。悪意あるページは被害者のブラウザ内で復号されるまで非表示に保たれ、従来の URL 検査では検知不能。5日前(7/4)に露出した EvilTokens ツールキットが、実際の攻撃キャンペーンとして観測された形で、**分析→実戦投入** の連続性が見えます。

**2026-07-14 — Forg365、フルスタック PhaaS として登場**
Forg365 は、デバイスコードフィッシング・AitM(adversary-in-the-middle)・AI 支援の誘導文作成・侵害後のメールボックス操作を一つにまとめた PhaaS。Telegram 経由で月額400ドル(年額3,800ドル)で販売されています。注目すべきは、これまで別々の記事で語られてきた要素——デバイスコード(5/18)、AitM/セッション窃取、そして侵害後のメールボックス操作(4/14 のルール悪用)——が **1製品にパッケージ化** された点です。攻撃キルチェーン全体が商品化されました。

**2026-07-21 — HollowGraph、カレンダーを C2 に(出口の到達点)**
Group-IB が Windows マルウェア「HollowGraph」を特定。**Microsoft Graph API を悪用し、乗っ取った M365 カレンダーを暗号化された双方向 C2 チャネルとして使用** します。イスラエル関連組織を標的とし、2026年6月3日〜7月9日に12件の感染を確認。Cavern バックドアフレームワークとの関連が高く、イラン関連アクター Lyceum との技術的類似性が指摘されています。DNS トンネリングで Entra ID 認証情報を取得・更新し、RSA+AES-256-GCM のハイブリッド暗号化を採用。**攻撃者は自前サーバーを一切使わず、Microsoft の信頼されたインフラに完全依存** しています。5/21 の GraphWorm が示した方向性の、洗練された到達点と言えます。

## 手口・技術詳細 (TTP)

**入口(認証情報・セッション窃取)側**

- **デバイスコードフィッシング**(Tycoon2FA、Forg365):正規のデバイスコード認証フローを悪用。ユーザーにコードを入力させ、セッショントークンを窃取して MFA をバイパス。
- **AitM(中間者)/セッション窃取**(Forg365):認証プロキシを挟んでセッションを乗っ取る。パスワードだけでなく認証済みセッションそのものを奪うため、多くの MFA を回避。
- **OAuth 同意フィッシング**(ConsentFix/ClickFix):偽の同意画面で「許可」を押させ、アプリにトークン権限を委譲させる。パスワードもコードも不要で、MFA が意味をなさない。
- **ゴーストフィッシング**(EvilTokens):ページ本体を暗号化し、被害者のブラウザ内で初めて復号。ゲートウェイの URL/コンテンツ検査を回避。
- **PhaaS 化と AI 支援**(W3LL、EvilTokens/ARToken、Forg365):ツールキットが月額課金で流通。Forg365 は AI で誘導文を作成し、参入障壁をさらに下げている。

**出口(C2・持続化)側**

- **正規 SaaS の C2 転用**(GraphWorm、HollowGraph):Microsoft Graph API を C2 に使用。企業では遮断しづらく、IP/ドメイン検知が無力化。
- **カレンダーの秘密チャネル化**(HollowGraph):暗号化したカレンダー予定でデータ窃取と命令取得を実行。DNS トンネリングで Entra ID 認証情報を取得・更新、RSA+AES-256-GCM を採用。
- **メールボックスルール悪用**:フォワーディング/削除ルールで活動を隠蔽し、侵害後もアクセスを維持。SOC の監視範囲外になりやすい。

いずれも共通するのは、**Microsoft の正規インフラ・正規フローを土台にすることで、境界型・シグネチャ型の検知を構造的に回避する** という設計思想です。

## 影響と被害範囲

- **対象の広さ**:M365 を運用する組織全般。PhaaS の標的は業種を問いません。EvilTokens/ゴーストフィッシングは米国・欧州企業、HollowGraph はイスラエル関連組織と、地政学的な標的も見られます。
- **確認された規模(報道ベース)**:W3LL は17,000人以上の被害者・25,000超の侵害アカウント・被害総額2,000万ドル超。HollowGraph は2026年6月3日〜7月9日に12件の感染。他の PhaaS 事案の具体的被害件数は各報道時点では未記載です。
- **被害の連鎖**:アカウント乗っ取り後は、メール・ファイル・Teams データの流出、BEC の踏み台、さらなる標的型攻撃の足掛かりへと発展します。メールボックスルールや Graph API による持続化で、**発覚が遅れるほど被害が深く静かに進行** します。
- **帰属の注記**:HollowGraph と Lyceum(イラン関連)の関連は「技術的類似性の指摘」段階であり、Webworm の中国系という帰属を含め、確定的な国家帰属は報道時点では慎重に扱うべきです。

## 防御・推奨アクション

実務者が優先度順に着手すべき施策です。

**優先度1:認証の強化(入口を塞ぐ)**

- **フィッシング耐性のある MFA(FIDO2/パスキー)への移行を評価・推進**。AitM・デバイスコード・OTP 型 MFA バイパスに対する最も本質的な対策。
- **Conditional Access でデバイスコードフローを制限/無効化**(Tycoon2FA・Forg365 対策)。業務で不要なら無効化を検討。

**優先度2:OAuth・アプリ同意の統制**

- **信頼できないアプリへのユーザー同意を Conditional Access で制限**し、管理者承認フローを導入(ConsentFix/ClickFix 対策)。
- **OAuth アプリケーションを定期監査**し、不審なアプリ・過剰な委譲権限を削除。
- ユーザー教育:**予期しない OAuth 同意画面・デバイスコード要求では絶対に「許可/入力」しない** ことを周知。

**優先度3:侵害後の持続化・C2 の検知(出口を塞ぐ)**

- **全ユーザーのメールボックスルールを定期監査**し、不審なフォワーディング/削除ルールをアラート化。
- **Graph API のテナント外・異常アクセスパターンを監視**(GraphWorm/HollowGraph 対策)。
- **M365 監査ログで不審なカレンダー予定の作成を監視**し、**DNS トンネリング検出ルール**を確認(HollowGraph 対策)。
- Entra ID のサインインログを監視し、不審なサインインを継続的にレビュー。

**優先度4:メール防御と運用の底上げ**

- URL 検査依存から脱却し、**振る舞い検知・開封後の動的分析を組み合わせた多層メールセキュリティ**へ(ゴーストフィッシング対策)。
- フィッシング訓練を継続し、**不審メールの報告体制を強化**。
- 過去ログで w3ll.store 等への通信痕跡や、EchoCreep/GraphWorm/HollowGraph の IOC を脅威インテリと突合。

## 今後の見通し

この3か月が示したのは、**W3LL のような単発の摘発では PhaaS 市場は止まらない** という現実です。解体の直後から Tycoon2FA、EvilTokens/ARToken、Forg365 と後継が連続して登場し、機能はデバイスコード・AitM・OAuth 同意・AI 支援へと積み上がりました。今後も MFA バイパス手法を標準装備した PhaaS が、より低価格・低スキルで流通していく方向は続くと見るのが妥当です。

同時に、GraphWorm から HollowGraph への流れが示すように、**「攻撃者が自前インフラを持たず、Microsoft の正規サービスに完全依存する」C2 モデル** は今後の高度標的型攻撃の主流になり得ます。Graph API・カレンダー・メールボックスといった正規機能が悪用される以上、防御側は「通信先が悪いか」ではなく「正規機能の使われ方が異常か」を見る振る舞いベースの監視へ、軸足を移す必要があります。

要するに、入口では認証をフィッシング耐性化し、出口では正規機能の異常利用を可視化する——この二正面での防御態勢が、今後の M365 セキュリティの標準になっていくでしょう。なお、本稿の各事案の被害規模・帰属の一部は各報道時点での暫定情報であり、続報での更新に留意してください。

## 参照記事

- FBIがフィッシング・アズ・ア・サービス「W3LL」を解体、被害総額2,000万ドル超(Infosecurity Magazine, 2026-04-14) — https://www.infosecurity-magazine.com/news/fbi-dismantles-phishing-operation/
- メールボックスルール悪用が侵害後の隠密持続手法として台頭(Infosecurity Magazine, 2026-04-14) — https://www.infosecurity-magazine.com/news/mailbox-rule-abuse-stealthy-post/
- Tycoon2FA hijacks Microsoft 365 accounts via device-code phishing(BleepingComputer, 2026-05-18) — https://www.bleepingcomputer.com/news/security/tycoon2fa-hijacks-microsoft-365-accounts-via-device-code-phishing/
- Webworm Deploys EchoCreep and GraphWorm Backdoors Using Discord and MS Graph API(The Hacker News, 2026-05-21) — https://thehackernews.com/2026/05/webworm-deploys-echocreep-and-graphworm.html
- ConsentFix and ClickFix: How Microsoft 365 Accounts are Hijacked in 3 Seconds(BleepingComputer, 2026-07-03) — https://www.bleepingcomputer.com/news/security/consentfix-and-clickfix-how-microsoft-365-accounts-are-hijacked-in-3-seconds/
- ARToken PhaaS exposes EvilTokens' Microsoft 365 phishing toolkit(BleepingComputer, 2026-07-04) — https://www.bleepingcomputer.com/news/security/artoken-phaas-exposes-eviltokens-microsoft-365-phishing-toolkit/
- New Ghost Phishing Wave Is Breaking Traditional Email Security(The Hacker News, 2026-07-09) — https://thehackernews.com/2026/07/new-ghost-phishing-wave-is-breaking.html
- Forg365 PhaaS Targets Microsoft 365 with Device Code and AitM Session Theft(The Hacker News, 2026-07-14) — https://thehackernews.com/2026/07/forg365-phaas-targets-microsoft-365.html
- New HollowGraph Malware Hijacks Microsoft 365 Calendars for Covert C2 Communications(Infosecurity Magazine, 2026-07-21) — https://www.infosecurity-magazine.com/news/hollowgraph-microsoft-calendars/
