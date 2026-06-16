# 🔐 セキュリティ推奨の要点 — 5W1H で深く理解する
# 🔐 The Security Recommendation in 5W1H — Understand It Deeply

> このファイルはセキュリティ推奨の要点をまとめたものです。
> **推奨・なぜ・何を・どうやって・誰が・どこで・いつ** に整理しています。
> 各項目は **技術的な説明** と **やさしい説明** の両方で書いています。
> **日本語が先、その下に英語**を載せています。
>
> This file is a summary of the security recommendation.
> It is sorted into **Recommendation, Why, What, How, Who, Where, When**.
> Each point is written in two ways: a **technical** one and an **easy** one.
> **Japanese is first. English is right below.**

---

## 📖 用語集 (Glossary) — まず先に読むと分かりやすい

技術用語の意味をやさしく説明します。下のどの言葉も、後の章で出てきます。

| 用語 | やさしい意味（例え） |
|---|---|
| **AWS** | アマゾンのクラウド。コンピュータ（サーバー）を買わずに、アマゾンから借りて使う仕組み。 |
| **サーバー / インスタンス** | アマゾンのデータセンターで動いているコンピュータ。GitLab やテストサイトが、それぞれ1台で動く。 |
| **本番（プロダクション）** | 利用者が実際に使う**本物**のシステム。絶対に止めてはいけない。 |
| **テスト環境** | 安全に試すための**コピー／砂場**。壊れても本物の利用者には影響しない。 |
| **VPC** | アマゾンの中にある、あなた専用の**フェンスで囲まれた建物**。同じ建物の中のサーバー同士は話せるが、外からは入れない。 |
| **AWSアカウント** | アマゾンでのあなたの作業場全体。**複数持てる**（本番用・テスト用など）。VPCより強い分離になる。 |
| **セキュリティグループ** | サーバーの**ファイアウォール／ドアの番人**。誰がどのドア（ポート）をノックしてよいかを決める。 |
| **ポート（80 / 443 / 22）** | サーバーの**ドアの番号**。80・443 はウェブ用、22 は SSH（遠隔操作）用。 |
| **IPアドレス** | ネット上の機器の**住所**。場所が変わると変わるので、住所で制御するのは不安定。 |
| **Cloudflare（クラウドフレア）** | サイトの**前に立つ無料サービス**。中に入れる前に本人確認をし、サーバーを外から隠してくれる。今回使う道具。 |
| **ゼロトラスト（Zero Trust）** | 「**住所ではなく、本人を信じる**」という新しい考え方。GitLab やテストサイトに着く前に、毎回ログインで本人を確かめる。 |
| **トンネル（`cloudflared`）** | サーバーに入れる小さなプログラム。サーバーから Cloudflare へ**内側から外へ**つなぐ。だから受信ドアを全部閉じても、Cloudflare の検問所だけは通れる。 |
| **One-time PIN（ワンタイムPIN）** | メールに届く**使い捨ての確認コード**。それを入力して本人確認する。追加アカウント不要。 |
| **MFA（多要素認証）** | パスワードに加えて**もう1つの確認**（コードなど）を求める仕組み。より安全。 |
| **AWS Session Manager** | ドアを開けずにサーバーへ入れる**緊急用の入口**。締め出され防止に残しておく。 |
| **SSH** | サーバーを**遠隔操作する仕組み**。Git をSSHで使うチームは追加設定が要る。 |

*English*

## 📖 Glossary — read this first, it makes the rest easy

Plain-language meanings of the technical terms. Each word below shows up in the later sections.

| Term | Easy meaning (analogy) |
|---|---|
| **AWS** | Amazon's cloud. You rent computers (servers) from Amazon instead of buying them. |
| **Server / instance** | A computer running in Amazon's data center. GitLab and the test site each run on one. |
| **Production ("prod")** | The **real** system your users actually use. It must never go down. |
| **Test environment** | A **copy/sandbox** to try things safely. If it breaks, real users aren't affected. |
| **VPC** | Your own **fenced-off building** inside Amazon. Servers in the same building can talk; outsiders can't get in. |
| **AWS account** | Your whole workspace at Amazon. You can have **more than one** (prod, test). Stronger separation than a VPC. |
| **Security group** | A server's **firewall / door guard**. It decides who may knock on which door (port). |
| **Port (80 / 443 / 22)** | A **door number** on a server. 80 and 443 are for web; 22 is for SSH (remote control). |
| **IP address** | A device's **street address** on the internet. It changes with location, so controlling by address is shaky. |
| **Cloudflare** | A **free service that stands in front** of your sites. It checks ID before letting anyone in and hides your server. The tool we use. |
| **Zero Trust** | A new idea: **trust the person, not the address.** Before reaching GitLab or test, you log in every time. |
| **Tunnel (`cloudflared`)** | A small program on the server. It connects **from the inside out** to Cloudflare. So you can shut all inbound doors and still get in through Cloudflare's checkpoint. |
| **One-time PIN** | A **single-use code emailed to you** that you type in to prove who you are. No extra account needed. |
| **MFA (multi-factor auth)** | Asking for **one more check** (like a code) on top of a password. More secure. |
| **AWS Session Manager** | An **emergency entrance** to a server without opening a door. Keep it so you don't get locked out. |
| **SSH** | A way to **control a server remotely**. Teams using Git over SSH need one extra setup step. |

---

## 📌 推奨 (Recommendation)

**🔧 技術的**

やることは2つだけです。

1. テスト環境を本番とは別の **VPC**（＝アマゾンの中のあなた専用の囲われた建物）に置く。本番とは接続しない。
2. GitLab とテストサイトの前に **Cloudflare（＝サイトの前に立つ無料サービス）** の **ゼロトラスト（＝住所ではなく本人を信じる仕組み）** を置く。
   - IPアドレス（＝ネット上の住所）ではなく、**本人のログイン**でアクセスを制御する。
   - サーバーへの受信ポート（＝外から入るドア）は閉じる。

> 💡 さらに強くしたい場合のみ、テストを **別のAWSアカウント** に置けます（**任意**）。

**🙂 やさしく**

考え方は2つだけです。

1. **建物を分ける** — テストを別の建物に入れて、本番に迷惑をかけないようにする。
2. **ドアの前に受付を置く** — 誰が入る時も、受付で本人確認をする。

> 💡 もっと強くしたいなら、テストを「まったく別の場所」に置くこともできます（やってもやらなくてもOK）。

*English*

**🔧 Technical**

There are only two things to do.

1. Put the test environment in its own **VPC** (= your own fenced-off building inside Amazon). Do not connect it to production.
2. Put **Cloudflare (= a free service that stands in front of your sites)** with **Zero Trust (= trust the person, not the address)** in front of GitLab and the test site.
   - Control access by **each person logging in**, not by IP address (= an address on the internet).
   - Close the servers' inbound ports (= the doors that let people in from outside).

> 💡 Only if you want it even stronger, you can put test in a **separate AWS account** (**optional**).

**🙂 Easy**

There are only two ideas.

1. **Separate the buildings** — put test in its own building so it can't disturb production.
2. **Put a reception at the door** — everyone shows ID at reception when they come in.

> 💡 If you want it stronger, you can put test in a "completely separate place" too (you can skip this — it's optional).

---

## ❓ なぜ (Why)

**🔧 技術的**

- テストと本番が同じVPCを共有すると、テスト側の障害や侵害が本番へ広がることがあります。
- 本番は止められないので、一切触らず、テストを別の場所に作ります。
- 「特定IPだけ許可」は弱い方法です。自宅・職場・スマホ・カフェで**アドレスが変わるたびに壊れます**。
- 今はサーバーがインターネットに**むき出し**になっています。

**🙂 やさしく**

- テストと本番が同じ建物にあると、テストでの失敗やハッキングが本番にこぼれます。
- 本番は止められないので触らず、テストだけ別の建物に建てます。
- 「この住所からだけ入れる」という鍵は弱いです。住所（IP）が変わると開かなくなります。
- 今は、サーバーが通りに面して**開けっぱなし**の状態です。

*English*

**🔧 Technical**

- If test and production share one VPC, a failure or hack in test can spread to production.
- Production must never go down, so we don't touch it. We build test somewhere else.
- "Allow only these IPs" is weak. It **breaks every time an address changes** (home, office, phone, café).
- Right now the servers are sitting **exposed** on the internet.

**🙂 Easy**

- If test and production share one building, a mistake or hack in test leaks into production.
- Production can't be stopped, so we leave it alone and build test in another building.
- Locking the door with "only this address may enter" is weak. It stops working when the address (IP) changes.
- Right now the server is left **wide open** on the street.

---

## 📦 何を (What)

**🔧 技術的**

用意するものは4つです。

1. テスト用の独立した **VPC**。
2. 無料の **Cloudflare アカウント**。
3. 各サーバー（GitLab・テストサイト）に入れる小さなプログラム **`cloudflared`（トンネル）**。
4. **ログインポリシー** — 許可するメールの一覧、または `@yourcompany.com` で終わるメール。

**🙂 やさしく**

必要なものは4つです。

1. テスト用の「別の建物」。
2. 無料の Cloudflare の登録。
3. 各サーバーに入れる小さな**「専用通路」プログラム**。
4. 「このメールの人だけ入れる」という**入場ルール**。

*English*

**🔧 Technical**

You need four things.

1. A separate **VPC** for test.
2. A free **Cloudflare account**.
3. A small program, **`cloudflared` (the tunnel)**, on each server (GitLab and the test site).
4. A **login policy** — a list of allowed emails, or emails ending in `@yourcompany.com`.

**🙂 Easy**

You need four things.

1. A "separate building" for test.
2. A free Cloudflare sign-up.
3. A small **"private corridor" program** on each server.
4. An **entry rule**: "only people with these emails may come in."

---

## 🛠️ どうやって (How)

**🔧 技術的**

1. AWS コンソールで **VPC を作成**（`test-vpc`）。本番VPCとは**つながない**。中にテストサイトとテスト用GitLabを起動。
2. ドメインを **Cloudflare に追加**し、ネームサーバーを向ける。
3. ダッシュボードで **Zero Trust（Freeプラン）** を有効化。
4. **Networks → Tunnels** で各サーバーにトンネルを作り、ホスト名を設定（`gitlab.yourcompany.com` / `test.yourcompany.com`）。
5. **Access → Applications** でアプリを追加し、**Allow ポリシー**でメールを指定。
6. **Settings → Authentication** で **One-time PIN**（メールに届くコード）を選ぶ。任意で **Sign in with Google**。
7. AWS の **セキュリティグループ**で旧IP許可ルールを削除し、受信ポート **80/443/22 を閉じる**。
8. （推奨）Access ポリシーで **MFA（多要素認証）** を要求。

> 💡 任意:別アカウントにしたい場合は、AWS Organizations で新アカウントを作り、同じ手順をその中で行います。

**🙂 やさしく**

1. テスト用の「別の建物」をAWSで作る。本番の建物とは**つながない**。
2. 自分のドメインを Cloudflare に登録する（画面の指示をコピペするだけ）。
3. 無料の「ゼロトラスト（最大50人）」をオンにする。
4. 各サーバーに**専用通路**を引き、`gitlab...` と `test...` の住所を割り当てる。
5. 「このメールの人だけ入れる」という**受付ルール**を追加する。
6. 本人確認は、まず**メールに届くコード**でOK（追加アカウント不要）。必要ならGoogleログインも。
7. 古い「この住所だけ許可」のルールを消し、サーバーの**表のドア（ポート）を全部閉じる**。
8. （おすすめ）**もう一段の確認**をオンにすると安心。

> 💡 任意:もっと強くしたいなら、テストを「まったく別の場所（別アカウント）」に作って同じ手順をやります。

*English*

**🔧 Technical**

1. In the AWS Console, **create a VPC** (`test-vpc`). Do **not** connect it to the production VPC. Launch the test site and a test GitLab inside it.
2. **Add your domain to Cloudflare** and point its nameservers there.
3. Turn on **Zero Trust (Free plan)** in the dashboard.
4. Under **Networks → Tunnels**, create a tunnel on each server and set the hostname (`gitlab.yourcompany.com` / `test.yourcompany.com`).
5. Under **Access → Applications**, add an app and set an **Allow policy** with the emails.
6. Under **Settings → Authentication**, pick **One-time PIN** (a code emailed to them). Optionally add **Sign in with Google**.
7. In AWS **Security Groups**, remove the old IP-allow rules and **close inbound ports 80/443/22**.
8. (Recommended) Require **MFA** in the Access policy.

> 💡 Optional: For a separate account, create one in AWS Organizations and do the same steps inside it.

**🙂 Easy**

1. Build the test "separate building" in AWS. **Don't connect it** to the production building.
2. Register your domain with Cloudflare (just copy-paste what the screen shows).
3. Turn on the free "Zero Trust" (up to 50 people).
4. Lay a **private corridor** into each server and give them the `gitlab...` and `test...` addresses.
5. Add the **reception rule**: "only people with these emails may enter."
6. For ID, a **code emailed to them** is enough to start (no extra accounts). Add Google login if you want.
7. Delete the old "only this address allowed" rule and **shut all the front doors (ports)**.
8. (Recommended) Turn on **one more check** for extra safety.

> 💡 Optional: If you want it stronger, build test in a "completely separate place (separate account)" and do the same steps.

---

## 👥 誰が (Who)

**🔧 技術的**

- 対象はあなたのチーム（**50人未満**）。
- Cloudflare のゼロトラスト Free は **最大50ユーザー**まで永続的に無料。要件を満たします。
- 比較:Tailscale の無料枠は **6人だけ**（2026年4月に変更）。人数が足りません。
- アクセスは**許可リストのメールを持つ人**だけに限定されます。

**🙂 やさしく**

- 入れるのは**チームのメンバーだけ**（今は50人未満）。
- Cloudflare は50人まで無料なので、ちょうど収まります。
- 似たツールの Tailscale は無料だと6人までしか入れず、足りません。
- 鍵は「人」に紐づくので、許可されたメールの人だけが入れます。

*English*

**🔧 Technical**

- The audience is your team (**under 50 people**).
- Cloudflare Zero Trust Free covers **up to 50 users**, forever. It meets your need.
- Compare: Tailscale's free tier is only **6 users** (changed April 2026). Too few.
- Access is limited to **people whose emails are on the allowlist**.

**🙂 Easy**

- Only **your team members** get in (currently under 50).
- Cloudflare is free for up to 50 people, so you fit nicely.
- The similar tool Tailscale only allows 6 people free — not enough.
- The key is tied to the *person*, so only people with an allowed email can enter.

---

## 📍 どこで (Where)

**🔧 技術的**

設定する場所は2つです。

1. **AWS コンソール** — VPCの作成、サーバー起動、**セキュリティグループ**でのポート開閉。
2. **Cloudflare の Zero Trust ダッシュボード** — Tunnels、Access のアプリとポリシー、Authentication 設定。

入口は生のIPではなく、Cloudflare 経由のホスト名（`gitlab.yourcompany.com` / `test.yourcompany.com`）になります。

**🙂 やさしく**

作業する画面は2つです。

1. **AWSの管理画面** — 建物（VPC）を作り、ドア（ポート）を閉める所。
2. **Cloudflareの管理画面** — 通路と受付ルールを作る所。

みんなの入口は、サーバーの生の住所ではなく、受付を通る `test.yourcompany.com` のような住所になります。

*English*

**🔧 Technical**

There are two places to set up.

1. The **AWS Console** — create the VPC, launch servers, open/close ports in **Security Groups**.
2. The **Cloudflare Zero Trust dashboard** — Tunnels, Access apps and policies, Authentication settings.

The entry point becomes Cloudflare hostnames (`gitlab.yourcompany.com` / `test.yourcompany.com`), not the raw IPs.

**🙂 Easy**

There are two screens to work in.

1. The **AWS dashboard** — where you build the building (VPC) and shut the doors (ports).
2. The **Cloudflare dashboard** — where you build the corridor and the reception rule.

Everyone comes in through an address like `test.yourcompany.com` that goes through reception — not the server's raw address.

---

## ⏰ いつ (When)

**🔧 技術的**

- 本人確認は**ログインのたびに毎回**行われます（IPで一度許可して通すのではない）。
- 切り替える理由:人の**IPアドレスは場所ごとに変わる**から。自宅でも、出張先でも、スマホでも、本人なら入れます。
- 任意で **MFA** を要求すれば、毎回より強い確認をかけられます。

**🙂 やさしく**

- 受付での本人確認は**入るたびに毎回**やります。
- 住所（IP）は場所が変わると変わるので、「人」を確認する方式なら、どこからでも本人なら入れます。
- 心配なら、もう一段の確認（MFA）を毎回付けられます。

*English*

**🔧 Technical**

- Identity is checked **every time someone logs in** (not "allow once by IP and let through").
- Why switch: a person's **IP changes by location**. At home, on a trip, or on a phone — the right person still gets in.
- Optionally requiring **MFA** adds a stronger check on every login.

**🙂 Easy**

- The ID check at reception happens **every time you come in**.
- An address (IP) changes when your location changes. Checking the *person* lets the right person in from anywhere.
- If you're worried, you can add one more check (MFA) every time.

---

## 💰 費用 (Cost)

**🔧 技術的**

- 別VPC:**$0**（払うのはテストサーバー本体の利用料だけ。普通のテスト機と同じ）。
- 別AWSアカウント（任意）:**$0**。
- Cloudflare Zero Trust（50人未満）:永続的に **$0**。
- One-time PIN / Googleログイン:**$0**。
- **追加コストはゼロ。**

**🙂 やさしく**

- 別の建物（VPC）はタダ。
- もっと分けたい時の別アカウントもタダ（任意）。
- Cloudflareの受付も50人まではずっとタダ。
- メールコードやGoogleログインもタダ。
- **新しく増える費用はありません。**

*English*

**🔧 Technical**

- Separate VPC: **$0** (you only pay for the test server itself, like any test machine).
- Separate AWS account (optional): **$0**.
- Cloudflare Zero Trust (under 50 people): **$0**, forever.
- One-time PIN / Google login: **$0**.
- **No added cost.**

**🙂 Easy**

- The separate building (VPC) is free.
- The separate account, if you want more separation, is also free (optional).
- Cloudflare's reception is free for up to 50 people, forever.
- Email codes and Google login are free too.
- **Nothing new to pay.**

---

## ✅ 動作確認 (Verification)

**🔧 技術的**

1. **分離:** テストサーバーから本番のプライベートリソースに**到達できない**こと、本番が稼働し変わっていないことを確認。
2. **非公開:** （Wi-Fiを切った）スマホからサーバーの**生IP**にアクセス → **到達不可**。`test.yourcompany.com` → **Cloudflareのログイン画面**が出る。
3. **ログイン:** 許可メールでログイン → 入れる。無関係なメール → **拒否**される。
4. **IP非依存:** 新しい場所（スマホのテザリング）からログイン成功 → アクセスが「人」に紐づくことを確認。

> ⚠️ 注意
> - SSHでのGit利用には `cloudflared access ssh` の追加設定が必要。
> - 締め出し防止に AWS **Session Manager** の緊急経路を必ず残す。

**🙂 やさしく**

1. **建物が分かれているか** — テスト側から本番の中身に**入れない**こと、本番がちゃんと動いていることを確認。
2. **開けっぱなしでないか** — スマホで生の住所を打つ → **何も出ない**。`test...` を打つ → **受付（ログイン画面）**が出る。
3. **受付が効くか** — 許可した人のメール → 入れる。知らないメール → **入れない**。
4. **どこからでも本人なら入れるか** — 別の場所（スマホ）からログインして成功するか試す。

> ⚠️ 注意
> - コードをSSHで送るチームは追加設定が要ります。
> - 自分が締め出されないよう、緊急用の入口（AWS Session Manager）を必ず残しておく。

*English*

**🔧 Technical**

1. **Separation:** From the test server, confirm you **cannot** reach production's private resources, and that production is still running and unchanged.
2. **Not exposed:** From a phone (off Wi-Fi), hit the server's **raw IP** → **unreachable**. Hit `test.yourcompany.com` → you get the **Cloudflare login page**.
3. **Login:** Log in with an allowed email → you get in. An unrelated email → **blocked**.
4. **IP-independent:** Log in from a new location (phone tethering) → confirms access follows the *person*.

> ⚠️ Notes
> - Git over SSH needs the extra `cloudflared access ssh` step.
> - Keep an emergency path via AWS **Session Manager** so you're never locked out.

**🙂 Easy**

1. **Are the buildings separated?** Confirm you **can't** reach production's contents from test, and that production still runs fine.
2. **Is it no longer wide open?** Type the raw address on your phone → **nothing appears**. Type `test...` → the **reception (login page)** appears.
3. **Does reception work?** An allowed person's email → gets in. An unknown email → **can't get in**.
4. **Can the right person get in from anywhere?** Try logging in from another location (your phone) and check it works.

> ⚠️ Notes
> - Teams that push code over SSH need an extra setup step.
> - Always keep an emergency entrance (AWS Session Manager) so you don't lock yourself out.

---

## 🔗 出典 / Sources

すべての主要な事実は、以下の公式・一次情報で確認済みです（2026年6月時点）。
All key facts below were verified against these official / primary sources (as of June 2026).

**料金・人数 / Pricing & user limits**

- Cloudflare Zero Trust の料金（無料プランは最大50ユーザー） / Cloudflare Zero Trust plans (free = up to 50 users):
  https://www.cloudflare.com/plans/zero-trust-services/
- Tailscale 無料枠は現在6ユーザー（2026年4月に変更） / Tailscale free tier is now 6 users (changed April 2026):
  https://tailscale.com/docs/account/manage-plans/free-plans-discounts

**仕組み / How it works**

- トンネル（`cloudflared`）は「内側から外へ」つなぐので受信ポートが不要 / Tunnel (`cloudflared`) is outbound-only, so no inbound ports are needed:
  https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/
- One-time PIN（メールコード）でのログイン — IdP不要 / One-time PIN (email code) login — no identity provider required:
  https://developers.cloudflare.com/cloudflare-one/integrations/identity-providers/one-time-pin/
- メールアドレスでアクセスを許可するポリシー / Access policies that allow by email address:
  https://developers.cloudflare.com/cloudflare-one/access-controls/policies/
- MFA（多要素認証）をポリシーで必須にする / Require MFA in an Access policy:
  https://developers.cloudflare.com/cloudflare-one/access-controls/policies/mfa-requirements/
