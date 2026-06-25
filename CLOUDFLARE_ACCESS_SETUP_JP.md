# 🔐 社内リモートアクセス — Cloudflare Zero Trust（無料）セットアップ

**目的：** 社内の AWS アプリ（GitLab、ダッシュボード、サーバー、データベース）に
**どこからでも** アクセスし、**会社メール** でログインし、すべてを **公開インターネット
から隠す** — 在宅勤務で壊れる「オフィス IP 許可リスト」を置き換えます。
Cloudflare の **無料** プラン（最大 50 ユーザー、$0）で実現します。

本ガイドは単体で完結します。AWS の数ステップは管理者が行い、Cloudflare 側の操作は
非エンジニアでも追えるようクリック手順で示します。

---

## 🧭 全体像（最初に読む）

AWS ネットワーク内に小さなプログラム（**`cloudflared`**＝コネクター）を置きます。これは
Cloudflare へ **外向きのみ** 接続し、サーバー側に **受信ポートは一切開きません**。
Cloudflare が **玄関** になり、アプリの URL を開くと **ログイン画面** を出し、
**会社メール** を確認してから初めてアプリへ通します。

**アプリの種類ごとの方法：**

| アクセス対象 | 方法 | 端末にアプリ必要？ |
|---|---|---|
| Web アプリ（GitLab、Grafana、管理画面） | トンネル **公開ホスト名** ＋ **Access** ログイン | **不要** — ブラウザのみ |
| **SSH**（サーバーへの端末） | **ブラウザ内 SSH** ＋ Access | 不要（ブラウザ内） |
| **RDP**（Windows リモートデスクトップ） | **ブラウザ内 RDP** ＋ Access | 不要（ブラウザ内） |
| **データベース / その他の生接続** | **WARP アプリ** ＋ プライベート経路 ＋ ポリシー | **WARP** 必要 |

つまり **Web・SSH・RDP はインストール不要** — URL を開いてログインするだけ。
**データベース / 生 TCP ツール** だけ、ラップトップに **WARP** が必要です。

**無料の 3 つの部品：**
- **Tunnel（トンネル）** — AWS への安全なコネクター。
- **Access** — 各アプリの前に立つログイン関門（会社メール）。
- **WARP** — 端末アプリ。データベース / 生 TCP のときだけ。

---

## ✅ はじめる前に（前提条件）

1. **Cloudflare 管理下のドメイン。** Access には Cloudflare が管理するドメインが必要。
   自社ドメインの DNS を Cloudflare へ移す（無料）か、`apps.ourcompany.com` のような
   サブドメインを委任します。アプリは `gitlab.apps.ourcompany.com` のような住所になります。
2. **会社メールでログインする手段**（ステップ A で設定）。
3. **AWS を操作できる管理者**（コネクター導入と、後でアプリを締める作業）。

> ⚠️ Cloudflare 管理ドメインが無いと、アプリの前にログイン画面を置けません。
> まずこれを解決してください。

---

## 🟦 ステップA — 会社メールのログインを有効化（あなた、Cloudflare 側）

1. **Zero Trust** を開く（<https://one.dash.cloudflare.com>）。
2. **Settings → Authentication → Login methods → Add new**。
3. 会社メールの確認方法を選ぶ：
   - **推奨：** **Google Workspace** または **Microsoft Entra / 365** — 会社メールが
     Google か Microsoft なら最もきれい（本物のシングルサインオン、会社アカウントに直結）。
   - **最も簡単・準備不要：** **One-time PIN** — メールを入力してコードを受け取る方式。
4. 方法をクリック → 案内に従い **Save** → **Test** で動作確認。

✅ **表示されるはず：** ログイン方法が一覧に出て Test 成功。以下の各アプリはこの
ログインを必須にします。

---

## 🟦 ステップB — AWS にコネクターを入れる（管理者が実施、あなたは画面を確認）

コネクターは VPC 内の小さなプログラムで、AWS と Cloudflare をつなぎます。

1. Zero Trust → **Networks → Connectors → Create a tunnel → Cloudflared**。
2. **名前** を `aws-apps` → **Save tunnel**。
3. Cloudflare が **インストールコマンド** を表示。**管理者** がアプリと同じ VPC 内の小さな
   EC2（または ECS タスク）で実行。外向き接続のみです。
4. トンネルが **🟢 Healthy** になるまで待つ。
5. 後のデータベース / 生 TCP のため、管理者はコネクター設定で WARP ルーティングを有効化：
   ```
   warp-routing:
     enabled: true
   ```

✅ **表示されるはず：** `aws-apps` トンネルが **🟢 Healthy**。コネクター 1 つで
**すべて** のアプリを賄えます。

> 💡 既に健全なトンネル（例：`gitlab-tunnel`）があれば、新規作成せず再利用できます。

---

## 🟩 ステップC — WEB アプリを追加（繰り返す基本パターン）

Web アプリごとに 1 回。例：GitLab。

1. **ホスト名を公開。** Networks → **Connectors** → トンネルを開く → **Public hostname /
   Published application routes → Add**：
   - **サブドメイン/ドメイン：** `gitlab.apps.ourcompany.com`
   - **Service：** `https://` ＋ アプリの **内部** アドレス（コネクターが AWS 内で到達
     できる ALB かインスタンス）。
   - **Save**。
2. **前にログイン関門を置く。** Access → **Applications → Add an application →
   Self-hosted** → `gitlab.apps.ourcompany.com` を入力 → **Next**。
3. **ポリシーを追加：**
   - **Action：** `Allow`
   - **Include：** **Emails ending in** → `@ourcompany.com`
   - （任意：MFA 必須、またはグループ限定。）
   - **Save / Add application。**

✅ **テスト：** `https://gitlab.apps.ourcompany.com` を開く → Cloudflare ログイン画面 →
会社メールでサインイン → GitLab 表示。会社外メールは **拒否**。

🔁 **次の Web アプリ** は、固有のホスト名と内部アドレスでステップ C を繰り返す。

---

## 🟦 ステップD — SSH / RDP を追加（クライアント不要・ブラウザ内）

1. 同じトンネルにサーバー用の経路を追加：
   - **SSH：** **ブラウザ内 SSH / ターミナル** アプリ（または SSH 向け "Access for
     Infrastructure"）を、サーバー内部アドレス＋ポート 22 で追加。
   - **RDP：** **ブラウザ内 RDP** アプリを、Windows ホスト＋ポート 3389 で追加。
2. Access → ポリシーを付与：**Allow**、**Emails ending in** `@ourcompany.com`。

✅ **テスト：** アプリ URL を開く → 会社メールでログイン → **ターミナル**（SSH）または
**Windows デスクトップ**（RDP）が **ブラウザ内** に開く。PuTTY / RDP クライアント不要。

> ⚠️ これらブラウザ内アプリのポリシーは **Allow** か **Block** のみ
> （Bypass / Service Auth 非対応）。

---

## 🟧 ステップE — データベース / 生 TCP ツールを追加（WARP アプリが必要）

データベース等の非ブラウザツールはページ内に描画できないため、利用者の端末で **WARP**
アプリを動かします（初回インストール＋会社メールでログイン）。

1. **DB のプライベート IP を経路化。** Networks → **Routes → Add CIDR route** → DB ホストの
   プライベートアドレスを `/32` で（例：`172.31.40.10/32`）→ トンネル経由。
2. **WARP に運ばせる。** Team & Resources → **Devices → Device profiles → Default →
   Split Tunnels（Include）→ Manage** → 同じ `/32` を追加。
3. **ID で制限。** Gateway / Access の **ネットワークポリシー** で `@ourcompany.com` のみ
   そのアドレスへ到達可に。
4. **利用者側：** **WARP** アプリを導入し、チーム名＋会社メールでログイン → **Connect**。
   その後 DB ツールをプライベート IP に向ける。

✅ **テスト：** WARP **接続中** は DB ツールがプライベート IP に到達。WARP **オフ** だと
不可。（GitLab 開発サーバー例 `172.31.35.68/32` と同じ経路パターン。）

---

## 🔒 ステップF — 公開の入口を閉じる（管理者、各アプリ稼働後）

アプリが Cloudflare 経由でログイン制御付きで到達できたら：

1. その **ALB を internal 化**、または ALB の **セキュリティグループ** を
   **`cloudflared` コネクターのみ** 許可に絞り、**古いオフィス IP 許可** ルールを削除。
2. SSH ポートを閉じる前に **AWS Session Manager** を緊急入口として残す。

✅ **結果：** アプリは認証済み Cloudflare ホスト名 **経由でのみ** 応答。公開インターネット
から消え、「オフィス IP か自宅 IP か」は完全に無関係になります。

---

## 🔁 アプリ追加チェックリスト（毎回コピペ）

- [ ] ホスト名を決める。例：`appname.apps.ourcompany.com`。
- [ ] **Web：** Public hostname → 内部アドレスを追加（ステップ C-1）。
- [ ] **SSH/RDP：** ブラウザ内アプリを追加（ステップ D）。
- [ ] **DB/TCP：** CIDR route ＋ Split Tunnel include を追加（ステップ E）。
- [ ] Access ポリシー：**Allow**、**Emails ending in** `@ourcompany.com`（C-3 / D-2 / E-3）。
- [ ] **社外ネットワーク** から会社メールでテスト。
- [ ] 管理者が ALB/SG をコネクターに絞り、IP 許可を削除（ステップ F）。

---

## ⚠️ 無料プランの上限・注意

- **最大 50 ユーザー** 無料。ログ保持 **24 時間**。**3 拠点** 上限。サポートはコミュニティ。
  本用途には十分。
- **Web・SSH・RDP** は **クライアント不要**（ブラウザのみ）。**DB / 生 TCP** だけ
  **WARP** が必要。
- ブラウザ内 SSH/RDP のポリシーは **Allow/Block** のみ。
- アプリのホスト名を載せるドメインは **Cloudflare 管理**（または委任サブドメイン）必須。

---

## 🔎 全体の確認

1. **自宅 / モバイル回線** からアプリ URL を開く → Cloudflare ログインが出る。
2. 会社メール → アプリ表示。**会社外メール → 拒否。**
3. ステップ F 後：古い公開 ALB アドレスへ直接 → **ブロック**。Cloudflare ホスト名のみ可。
4. SSH/RDP はログイン後ブラウザ内で開く。DB ツールは WARP 接続時に動作し、オフだと不可。

---

## 🆘 うまくいかないとき

- **ログイン画面が出ない／素通り or 公開ブロックのまま：** そのホスト名に Access ポリシーが
  付いていない — ステップ C-2/3 を再確認。
- **ログインは通るがアプリが開かない：** トンネルの **Service** アドレス（C-1）に AWS 内の
  コネクターから到達できていない — 内部アドレス/ポートとサーバーのセキュリティグループを確認。
- **「このドメインは Cloudflare 上に無い」：** 前提条件を完了 — アプリのホスト名は
  Cloudflare 管理ドメイン配下である必要があります。
- **DB に接続できない：** WARP が **Connected** か、`/32` が CIDR route と Split Tunnel
  **Include** の **両方** にあるか、コネクターで `warp-routing` が有効かを確認。

---

## 📚 参考

- Zero Trust プラン・上限 — <https://www.cloudflare.com/plans/zero-trust-services/> ·
  <https://developers.cloudflare.com/cloudflare-one/account-limits/>
- セルフホストアプリの公開 —
  <https://developers.cloudflare.com/cloudflare-one/access-controls/applications/http-apps/self-hosted-public-app/>
- ブラウザ内 SSH / RDP —
  <https://developers.cloudflare.com/cloudflare-one/access-controls/applications/non-http/browser-rendering/>
