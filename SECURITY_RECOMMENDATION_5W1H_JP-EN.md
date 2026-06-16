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

## 🧭 目次 (Table of Contents)

1. [📖 用語集 (Glossary)](#-用語集-glossary--まず先に読むと分かりやすい)
2. [📌 推奨 (Recommendation)](#-推奨-recommendation)
3. [❓ なぜ (Why)](#-なぜ-why)
4. [📦 何を (What)](#-何を-what)
5. [🛠️ どうやって (How)](#-どうやって-how)
6. [👥 誰が (Who)](#-誰が-who)
7. [📍 どこで (Where)](#-どこで-where)
8. [⏰ いつ (When)](#-いつ-when)
9. [💰 費用 (Cost)](#-費用-cost)
10. [✅ 動作確認 (Verification)](#-動作確認-verification)
11. [🚀 速度と「100MBの上限」 (Performance & the 100 MB limit)](#-速度と100mbの上限-performance--the-100-mb-limit)
12. [🚚 移行とアクセス対象 (Migration & Access Targets)](#-移行とアクセス対象-migration--access-targets)
13. [👍 良い点・注意点 (Pros & Cons)](#-良い点注意点-pros--cons)
14. [🔗 出典 (Sources)](#-出典--sources)

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

## 🚀 速度と「100MBの上限」 (Performance & the 100 MB limit)

**🔧 技術的**

- トンネルは「利用者 → 近くの Cloudflare 拠点 → サーバー」と1回**遠回り**します。1リクエストあたり **およそ 50〜200ms** 追加されます。
- ウェブUIの閲覧やテストサイト、ふつうの小〜中サイズの `git push` / `pull` では**ほぼ気になりません**。
- 本当の注意点は速度ではなく**ハードな上限**です。**Free / Pro プランでは、1回のHTTPアップロードが 100MB を超えると拒否されます**（遅いのではなく**エラー**になる）。
- GitLab で影響するもの:**HTTPS 経由の大きな `git push`**、**Git LFS**、**コンテナレジストリ（Dockerイメージ）**、**100MB超のCI成果物**。

**回避策**

1. **Git は SSH で使う**（`cloudflared access ssh`）。SSHはHTTPプロキシを通らないので **100MB上限を回避**できる。← GitLab での一番重要な対策。
2. **レジストリ / LFS / 大きな成果物**は、トンネル経由ではなく **オブジェクトストレージ（例:S3）** に直接置く。上限も速度低下も避けられる。
3. スループットが必要なら **`cloudflared` を複数台（レプリカ）**で動かし、`cloudflared` を動かすサーバーのCPUを不足させない。

**🙂 やさしく**

- 通路を通る分、ほんの少し**遠回り**になります（1回につき約0.05〜0.2秒）。ウェブ画面やふつうの作業では**気づかないレベル**です。
- 本当の落とし穴は「遅さ」ではなく「**大きすぎる荷物は通せない**」こと。無料プランでは **1回に100MBを超えるアップロードは止められます**（遅いのではなく**失敗**します）。
- これに引っかかるのは、**大きなコードの送信・大きなファイル・Dockerイメージ・大きなビルド成果物**など。

**回避策**

1. コードの送受信は **SSH** を使う。SSHは別の通り方なので、**100MBの壁を回避**できます（GitLabでの一番大事なコツ）。
2. 大きなファイルやイメージは、通路を通さず **専用の保管庫（S3など）** に直接置く。
3. もっと速さが欲しければ、通路（`cloudflared`）を**複数本**にして、動かすサーバーの力に余裕を持たせる。

> ✅ まとめ:あなたの規模（50人未満、テストサイト＋GitLab）では**遅くて困ることはほぼありません**。
> 唯一の本当の注意は **100MBのアップロード上限**。SSHを使えば、その心配もなくなります。

*English*

**🔧 Technical**

- The tunnel adds one **detour**: user → nearest Cloudflare location → server. That's about **50–200 ms extra** per request.
- For web-UI browsing, the test site, and normal small-to-medium `git push` / `pull`, **you won't really notice it**.
- The real catch isn't speed — it's a **hard limit**. On the **Free / Pro plans, any single HTTP upload over 100 MB is rejected** (not slow — an **error**).
- What this affects in GitLab: **large `git push` over HTTPS**, **Git LFS**, the **container registry (Docker images)**, and **CI artifacts over 100 MB**.

**Workarounds**

1. **Use Git over SSH** (`cloudflared access ssh`). SSH isn't HTTP-proxied, so it **bypasses the 100 MB limit**. ← The most important fix for GitLab.
2. Put the **registry / LFS / large artifacts** in **object storage (e.g. S3)** directly, not through the tunnel. Avoids both the size limit and throughput caps.
3. For more throughput, run **multiple `cloudflared` replicas** and don't starve the server running `cloudflared` of CPU.

**🙂 Easy**

- Going through the corridor is a tiny **detour** (about 0.05–0.2 seconds each time). For web screens and normal work it's **not noticeable**.
- The real pitfall isn't "slow" — it's that **a too-big package can't pass**. On the free plan, **any upload over 100 MB at once is stopped** (it **fails**, it isn't just slow).
- This bites things like **sending big code, big files, Docker images, and large build outputs**.

**Workarounds**

1. Send/receive code over **SSH**. SSH goes a different way, so it **gets around the 100 MB wall** (the most important tip for GitLab).
2. Put big files and images in a **dedicated storage (like S3)** directly, not through the corridor.
3. If you want more speed, run **several corridors** (`cloudflared`) and give the server enough power.

> ✅ Bottom line: At your size (under 50 people, test site + GitLab), it is **very unlikely to feel slow**.
> The only real thing to watch is the **100 MB upload limit** — and using SSH removes that worry too.

---

## 🚚 移行とアクセス対象 (Migration & Access Targets)

実際の質問に対する詳しい回答です。**(1) 既存サーバーの移行**と **(2) 安全にアクセスしたい対象**の2つに分けて説明します。
Detailed answers to the real questions, in two parts: **(1) migrating existing servers** and **(2) what we want to give secure access to.**

---

### 🚚 パート1:AWSアカウント作成と既存サーバーの移行 (Part 1 — New AWS account & migrating existing servers)

**❓ 論点 / The point**

新しい（テスト用の）AWSアカウントを作る場合、今ある **EC2 インスタンス**・**RDS データベース**・**静的サイト（CloudFront + S3）** を移行できるか? 移行に停止が必要なら、いつ・どれくらい止まるのか?

If you create a new (test) AWS account, can existing **EC2 instances**, **RDS databases**, and a **static site (CloudFront + S3)** be migrated? And if a stop is needed, when and for how long?

**✅ 回答 / Answer**

はい、すべて新しいAWSアカウントに移行できます。アカウントをまたぐ移行なので、切り替えの瞬間に**短い停止時間**が発生します。事前にお知らせし、利用の少ない時間帯に作業すれば、影響は抑えられます。

Yes — all of them can be migrated to a new AWS account. Because it crosses accounts, there is a **short
downtime at the cutover moment**. Announcing it in advance and doing the work during low-usage hours keeps the impact small.

**🔧 技術的:各リソースの移行方法**

| リソース | 移行方法 | 停止時間 |
|---|---|---|
| **EC2**（アプリのサーバー） | ① インスタンスの **AMI（マシンイメージ）** を作成 → ② 新アカウントへ **共有** → ③ 新アカウントのVPC内で **そのAMIから起動**。OS・アプリ・設定・証明書・cron などサーバー丸ごと移る。 | 停止／スナップショット ＋ 切替の作業枠（通常 数十分） |
| **RDS**（データベース） | ① **手動スナップショット**を取得 → ② 新アカウントへ **共有** → ③ 新VPCで **新DBとして復元**。 | 最終同期・切替の作業枠 |
| **ホームページ = CloudFront + S3** | S3 は「移動」ではなく、**中身を新バケットへコピー**（`aws s3 sync`）→ それを指す **CloudFront を作り直し** → **DNS を切替**。 | ほぼゼロ（DNS切替のみ） |

**⚠️ 移行前に押さえる重要点**

1. **IPアドレスが変わります。** 移行後、サーバーの住所が新しくなるので、DNS／Elastic IP／Cloudflare のホスト名を**指し直す**必要があります。（Cloudflare を前に置いていれば、トンネルの宛先を直すだけで楽になります。）
2. **暗号化された RDS スナップショット**は1手間増えます。**カスタマー管理の KMS キー**を使い、そのキーも共有する必要があります（AWS既定のキーはアカウント間で共有不可）。
3. **RDS の停止時間を最小化したい場合**は、**AWS DMS（Database Migration Service）**で新旧DBを同期し続け、停止を **約5分未満**に抑えられます。スナップショット復元はより簡単ですが、作業枠は長めになります。
4. **本番切替の前に、復元したコピーを新アカウントで必ずテスト**してください。

> 💡 補足:別アカウントは**任意**です。分離が目的なら、**同じアカウント内の別VPC**でもよく、その場合アカウントをまたぐ移行は不要です。請求やログインまで完全に分けたい時だけ、別アカウントにします。

*English*

**🔧 Technical: how each resource migrates**

| Resource | Method | Downtime |
|---|---|---|
| **EC2** (application server) | ① Create an **AMI (machine image)** of the instance → ② **share** it with the new account → ③ **launch from that AMI** inside the new account's VPC. The whole server moves: OS, app, configs, certs, cron. | Stop/snapshot + cutover window (usually tens of minutes) |
| **RDS** (database) | ① Take a **manual snapshot** → ② **share** it with the new account → ③ **restore as a new DB** in the new VPC. | Final-sync / cutover window |
| **Homepage = CloudFront + S3** | S3 isn't "moved" — **copy the contents to a new bucket** (`aws s3 sync`) → **recreate CloudFront** pointing at it → **switch DNS**. | Near-zero (just a DNS switch) |

**⚠️ Important points to note before migrating**

1. **IP addresses change.** After migration the servers get new addresses, so DNS / Elastic IP / Cloudflare hostnames must be **repointed**. (If Cloudflare is already in front, you just update the tunnel's target — easier.)
2. **Encrypted RDS snapshots** need one extra step: use a **customer-managed KMS key** and share that key too (the default AWS-managed key cannot be shared across accounts).
3. **To minimize RDS downtime**, use **AWS DMS (Database Migration Service)** to keep old and new DB in sync and cut the outage to **under ~5 minutes**. A plain snapshot-restore is simpler but needs a longer window.
4. **Always test the restored copy** in the new account **before** the final cutover.

> 💡 Note: A separate account is **optional**. If the goal is just isolation, a **separate VPC in the same account** works too and avoids cross-account migration. Use a separate account only if you want fully separate billing/logins.

---

### 📋 移行の手順（ステップ・バイ・ステップ） (Migration — step by step)

以下はすべて **AWS マネジメントコンソール（管理画面）** で操作できます。専門コマンドがなくても進められます。
作業の前に、両方のアカウントの **アカウントID（12桁の数字）** を控えておきます。
*（旧 = 今のアカウント、新 = テスト用に作る新アカウント）*

You can do all of this in the **AWS Management Console**. No special commands are required.
Before starting, note both accounts' **Account IDs (the 12-digit number)**.
*(old = current account, new = the new account you create for test)*

**ステップ 0:準備 (Step 0 — Prepare)**

1. 新アカウントを作る:旧アカウントで **AWS Organizations** を開き、**「AWSアカウントを追加」** で新アカウントを作成（無料）。
2. 新アカウントにログインし、**移行先のVPC**（建物）を作る。本番VPCとは**つながない**。
3. **同じリージョン**で作業すると一番簡単（例:旧も新も東京リージョン）。

**ステップ 1:EC2（サーバー）を移行 (Step 1 — Migrate EC2)**

1. **旧アカウント** → EC2 → 対象インスタンスを選ぶ → **「アクション」→「イメージとテンプレート」→「イメージを作成」**。
   - 名前を付けて作成。これが **AMI（サーバー丸ごとのコピー）** になる。
2. 作った **AMI** を選ぶ → **「アクション」→「AMIを共有」** → **新アカウントのアカウントID** を入力して共有。
   - AMIが暗号化されている場合は、使っている **KMSキーも新アカウントへ共有**する。
3. **新アカウント** → EC2 → **「AMI」** → フィルタを **「プライベートイメージ」** にすると共有されたAMIが見える。
4. そのAMIを選び **「AMIからインスタンスを起動」** → **新しいVPC／サブネット**を選び、**セキュリティグループ**を設定して起動。
5. 起動後、アプリが正しく動くか**新環境でテスト**する（まだ本番切替はしない）。

**ステップ 2:RDS（データベース）を移行 (Step 2 — Migrate RDS)**

1. **旧アカウント** → RDS → 対象DB → **「アクション」→「スナップショットの取得」**（手動スナップショット）。
2. そのスナップショットを選ぶ → **「アクション」→「スナップショットの共有」** → **新アカウントのID** を追加。
   - 🔑 **暗号化されている場合:** AWS既定キーのままだと共有できません。先に **カスタマー管理のKMSキー**でスナップショットをコピーし、そのキーも新アカウントへ共有してから共有します。
3. **新アカウント** → RDS → **「スナップショット」→「他のアカウントと共有」** タブに共有されたスナップショットが出る。
4. それを選び **「スナップショットを復元」** → **新VPC**・インスタンスサイズを指定して新しいDBとして起動。
5. 新サーバー（ステップ1）の接続先を、この**新DBのエンドポイント（住所）**に向け直す。
6. 🟢 **停止を短くしたい場合:** スナップショット復元の代わりに **AWS DMS** で新旧DBを同期し続け、切替時の停止を **約5分未満**にできる。

**ステップ 3:ホームページ（CloudFront + S3）を移行 (Step 3 — Migrate the static site)**

1. **新アカウント** → S3 → **新しいバケットを作成**。
2. 旧バケットの中身を新バケットへ**コピー**する（コンソールのコピー、または `aws s3 sync s3://旧バケット s3://新バケット`）。
3. **新アカウント** → CloudFront → **新しいディストリビューションを作成** → オリジンに**新しいS3バケット**を指定。
   - 独自ドメイン・SSL証明書を使っている場合は、新ディストリビューションにも設定（**ACM**で証明書を発行）。
4. 新CloudFrontのURLで**表示を確認**してから、次のステップでDNSを切り替える。

**ステップ 4:本番切替（カットオーバー） (Step 4 — Cutover)**

1. 事前に利用者へ**メンテナンス時間を周知**（利用の少ない時間帯を選ぶ）。
2. 切替直前に、RDSを**最終同期**（DMS利用時）またはアプリを止めて**最後のスナップショット**を取得。
3. **DNS を新環境に向け直す**（Cloudflare のホスト名 / Route 53 のレコード / CloudFront のドメイン）。
4. **動作確認**:サイト表示・ログイン・予約や保存などの主要機能をひと通りテスト。
5. 問題があれば **DNSを旧環境に戻す**だけで**ロールバック**できる（旧環境は切替成功までしばらく残しておく）。
6. 数日〜1週間、新環境を監視して問題がなければ、旧リソースを停止・削除。

*English*

All steps below are done in the **AWS Management Console**. No special commands needed.
Before starting, note both accounts' **Account IDs**. *(old = current, new = the new test account)*

**Step 0 — Prepare**

1. Create the new account: in the old account open **AWS Organizations** → **"Add an AWS account"** (free).
2. Log in to the new account and create the **destination VPC** (building). Do **not** connect it to the production VPC.
3. Working in the **same region** for both is easiest (e.g. both in Tokyo).

**Step 1 — Migrate EC2 (the server)**

1. **Old account** → EC2 → select the instance → **Actions → Image and templates → Create image**.
   - Give it a name. This becomes the **AMI (a full copy of the server)**.
2. Select the **AMI** → **Actions → Share AMI** → enter the **new account's Account ID**.
   - If the AMI is encrypted, also **share the KMS key** with the new account.
3. **New account** → EC2 → **AMIs** → set the filter to **Private images** to see the shared AMI.
4. Select it → **Launch instance from AMI** → choose the **new VPC/subnet**, set a **security group**, and launch.
5. After it boots, **test the app in the new environment** (don't switch production yet).

**Step 2 — Migrate RDS (the database)**

1. **Old account** → RDS → select the DB → **Actions → Take snapshot** (a manual snapshot).
2. Select that snapshot → **Actions → Share snapshot** → add the **new account's ID**.
   - 🔑 **If encrypted:** you can't share it while it uses the default AWS key. First **copy** the snapshot with a **customer-managed KMS key**, share that key with the new account, then share the snapshot.
3. **New account** → RDS → **Snapshots → Shared with me** tab shows the shared snapshot.
4. Select it → **Restore snapshot** → choose the **new VPC** and instance size to launch it as a new DB.
5. Repoint the new server (Step 1) to this **new DB's endpoint (address)**.
6. 🟢 **To shorten downtime:** instead of snapshot-restore, use **AWS DMS** to keep old and new DB in sync and cut the switch-over outage to **under ~5 minutes**.

**Step 3 — Migrate the static site (CloudFront + S3)**

1. **New account** → S3 → **create a new bucket**.
2. **Copy** the old bucket's contents into the new one (console copy, or `aws s3 sync s3://old-bucket s3://new-bucket`).
3. **New account** → CloudFront → **create a new distribution** → set the origin to the **new S3 bucket**.
   - If you use a custom domain / SSL, configure it on the new distribution too (issue the cert in **ACM**).
4. **Check it loads** on the new CloudFront URL before switching DNS in the next step.

**Step 4 — Cutover**

1. **Announce a maintenance window** to users in advance (pick a low-usage time).
2. Just before switching, do a **final sync** of RDS (if using DMS), or stop the app and take a **final snapshot**.
3. **Repoint DNS to the new environment** (Cloudflare hostname / Route 53 record / CloudFront domain).
4. **Verify:** test the site loading, login, and key actions (bookings, saving, etc.).
5. If anything's wrong, **roll back** by simply pointing **DNS back to the old environment** (keep the old setup around until the switch is confirmed good).
6. Watch the new environment for a few days to a week; once it's clean, stop and delete the old resources.

---

### 🔐 パート2:安全にアクセスしたい対象 (Part 2 — Secure access targets)

**❓ 論点 / The point**

安全にアクセスしたい代表的な対象は次の3つ。これらを Cloudflare で保護できるか?

- **EC2 インスタンス** — Linux:**SSH（例:Tera Term）** / Windows:**リモートデスクトップ（RDP）**
- **ウェブサイト**

The three common targets we want to give secure access to — can Cloudflare protect them?

- **EC2 instances** — Linux: **SSH (e.g. Tera Term)** / Windows: **Remote Desktop (RDP)**
- **Website**

**✅ 回答 / Answer**

はい、3つともすべて Cloudflare で保護できます。どの場合も**サーバーの受信ポートは閉じたまま**にできます。ただし、**利用者の“つなぎ方”が対象ごとに違います**。

Yes — all three can be protected by Cloudflare, and in every case the **servers' inbound ports stay
closed**. The difference is **how each user connects**.

**🔧 技術的:対象ごとのつなぎ方**

| 対象 | 可否 | つなぎ方 | 利用者のPCに必要なもの | 閉じるポート |
|---|---|---|---|---|
| **ウェブサイト** | ✅ 簡単 | サイトを開く → Cloudflare のログイン画面 → 入る | **ブラウザだけ** | 80 / 443 |
| **Linux SSH（Tera Term）** | ✅ 可能 | いつも通り Tera Term を開き、サーバーの**プライベートアドレス**に接続 | 無料の **WARP クライアント**（インストール＋ログイン） | 22 |
| **Windows リモートデスクトップ** | ✅ 可能 | いつも通りリモートデスクトップを開き、**プライベートアドレス**に接続 | 無料の **WARP クライアント**（インストール＋ログイン） | 3389 |

**重要な違い**

- **ウェブサイト**は利用者のPCに何も要りません。**ブラウザとログインだけ**で入れます。
- **SSH（Tera Term）と RDP（リモートデスクトップ）**はブラウザではなく**専用ツール**を使います。これらを無料で安全にする一番きれいな方法が **「WARP-to-Tunnel」** 方式です:
  1. サーバー側で **`cloudflared` トンネル**を動かす（外向きだけの接続なので、SSH/RDP の公開ポートを閉じられる）。
  2. 各利用者が無料の **Cloudflare WARP クライアント**をPCに入れてログイン（同じメールの本人確認）。
  3. ログインすると、そのPCは**社内ネットワークにいるかのように**振る舞うので、**Tera Term とリモートデスクトップは今まで通りサーバーのプライベートアドレスに接続**できる（ただし先に本人確認が入る）。

つまり、**今使っているツール（Tera Term・リモートデスクトップ）はそのまま**で、ポート 22（SSH）と 3389（RDP）はインターネットに**むき出しでなくなります**。

**補足**

- 上記はすべて**無料のゼロトラスト（最大50人）**に含まれます（WARP クライアントとプライベートネットワーク接続を含む）。
- Cloudflare は**クライアント不要のブラウザRDP**（WARPもリモートデスクトップアプリも不要）も提供しています。便利ですが、Tera Term や標準のリモートデスクトップを使い続けたい場合は WARP 方式が最適です。
- どのメールが接続してよいかの**アクセスポリシー**や、任意の **MFA** は、ウェブサイトと同じように SSH/RDP にも適用されます。

*English*

**🔧 Technical: how to connect, per target**

| Target | Works? | How users connect | What the user's PC needs | Ports to close |
|---|---|---|---|---|
| **Website** | ✅ Easy | Open the site → Cloudflare login page → in | **A browser only** | 80 / 443 |
| **Linux SSH (Tera Term)** | ✅ Yes | Open Tera Term as usual, connect to the server's **private address** | The free **WARP client** (install + log in) | 22 |
| **Windows Remote Desktop** | ✅ Yes | Open Remote Desktop as usual, connect to the **private address** | The free **WARP client** (install + log in) | 3389 |

**The key difference**

- The **website** needs nothing on the user's PC — **a browser and login** is enough.
- **SSH (Tera Term) and RDP (Remote Desktop)** use **desktop tools**, not a browser. The cleanest free way to secure them is the **"WARP-to-Tunnel"** model:
  1. Run a **`cloudflared` tunnel** on the server (outbound-only, so the public SSH/RDP ports can be closed).
  2. Each user installs the free **Cloudflare WARP client** and logs in (same email-based identity).
  3. Once logged in, the PC behaves **as if it's on the private network**, so **Tera Term and Remote Desktop connect to the server's private address exactly as today** — just with identity checked first.

So your **existing tools (Tera Term, Remote Desktop) stay unchanged**, while ports 22 (SSH) and 3389 (RDP) are **no longer exposed** to the internet.

**Notes**

- All of the above is included in the **free Zero Trust plan (up to 50 users)**, including the WARP client and private-network routing.
- Cloudflare also offers **clientless, browser-based RDP** (no WARP client, no Remote Desktop app). Convenient, but the WARP method is the best fit if people want to keep using Tera Term / standard Remote Desktop.
- **Access policies** (which emails may connect) and optional **MFA** apply to SSH and RDP the same way they apply to the website.

---

## 👍 良い点・注意点 (Pros & Cons)

この方式（独立VPC ＋ Cloudflare ゼロトラスト）の良い点と注意点をまとめます。
A quick summary of the upsides and the things to watch with this approach (separate VPC + Cloudflare Zero Trust).

**👍 良い点 (Pros)**

| 項目 | 内容 |
|---|---|
| 💸 **費用** | 50人未満なら永続的に **$0**。追加費用なし。 |
| 🔒 **本人ベース** | IPではなく**本人のログイン**で制御。住所が変わっても本人なら入れる。 |
| 🙈 **サーバーを隠せる** | 受信ポートを全部閉じられる。生のIPは外から**到達不可**になる。 |
| 🧩 **IdP不要で始められる** | メールコード（One-time PIN）だけで開始可能。後でGoogleログインやMFAを追加できる。 |
| 🏢 **本番に手を触れない** | テストを別VPC（任意で別アカウント）に作るだけ。本番は止めず変えない。 |
| ⚙️ **構築が簡単** | 画面のコピペ中心。専用ハードや難しいネットワーク設定は不要。 |

**⚠️ 注意点 (Cons / Watch-outs)**

| 項目 | 内容 |
|---|---|
| 📦 **100MBの上限** | Free / Pro ではHTTPアップロードが100MB超で**失敗**。大きな`git push`・LFS・Dockerイメージは**SSHやS3で回避**が必要。 |
| 🐢 **わずかな遅延** | 1リクエストあたり約50〜200msの遠回り。ふつうの作業では気にならないが、遠い拠点では体感することも。 |
| 🔑 **SSHは追加設定** | Gitを SSH で使うチームは `cloudflared access ssh` の設定が必要。 |
| 🚪 **締め出しリスク** | 設定ミスやCloudflare障害で入れなくなる恐れ。**AWS Session Manager の緊急経路**を必ず残す。 |
| 🌐 **依存先が増える** | アクセスがCloudflareに依存する（Cloudflareが落ちると入口も止まる）。 |
| 👥 **50人の上限** | 無料は50人まで。**それを超えると有料**プランが必要。 |

*English*

**👍 Pros**

| Item | Detail |
|---|---|
| 💸 **Cost** | **$0** forever for under 50 people. No added cost. |
| 🔒 **Person-based** | Controlled by **the person's login**, not IP. The right person gets in even when their address changes. |
| 🙈 **Hides the server** | You can close all inbound ports. The raw IP becomes **unreachable** from outside. |
| 🧩 **Start with no IdP** | Begin with just an email code (One-time PIN). Add Google login or MFA later. |
| 🏢 **Production untouched** | You only build test in a separate VPC (optionally a separate account). Prod is never stopped or changed. |
| ⚙️ **Easy to set up** | Mostly copy-paste from the screen. No special hardware or hard networking. |

**⚠️ Cons / Watch-outs**

| Item | Detail |
|---|---|
| 📦 **100 MB limit** | On Free / Pro, an HTTP upload over 100 MB **fails**. Large `git push`, LFS, and Docker images need a **workaround (SSH or S3)**. |
| 🐢 **Small latency** | A ~50–200 ms detour per request. Unnoticeable for normal work, but far-away locations may feel it. |
| 🔑 **SSH needs extra setup** | Teams using Git over SSH must set up `cloudflared access ssh`. |
| 🚪 **Lock-out risk** | A misconfig or Cloudflare outage could lock you out. Always keep the **AWS Session Manager emergency path**. |
| 🌐 **One more dependency** | Access now depends on Cloudflare (if Cloudflare is down, the entrance is down too). |
| 👥 **50-user cap** | Free covers 50 people. **Beyond that you need a paid** plan. |

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

**速度・上限 / Performance & limits**

- アップロードの上限は Free / Pro で100MB / Upload size limit is 100 MB on Free / Pro:
  https://community.cloudflare.com/t/maximum-upload-size-is-limit/418490
- トンネルの遅延・スループットの傾向（コミュニティ報告） / Tunnel latency & throughput behavior (community reports):
  https://community.cloudflare.com/t/slower-tunnel-performance-outside-us-3-4x-slower/912777

**移行・アクセス対象 / Migration & access targets**

- RDS DB インスタンスを別VPC・別アカウントへ移行 / Migrate an RDS DB instance to another VPC or account:
  https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/migrate-an-amazon-rds-db-instance-to-another-vpc-or-account.html
- RDS スナップショットのコピー・共有（暗号化時のKMS含む） / Copy & share an RDS snapshot (incl. KMS for encrypted):
  https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_CopySnapshot.html
- WARP-to-Tunnel で RDP に接続 / Connect to RDP using WARP-to-Tunnel:
  https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/use-cases/rdp/rdp-warp-to-tunnel/
- ブラウザでRDPに接続（クライアント不要） / Connect to RDP in a browser (clientless):
  https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/use-cases/rdp/rdp-browser/
