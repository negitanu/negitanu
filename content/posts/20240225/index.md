---
title: "Microsoft 365 (Exchange Online) における SPF, DKIM, DMARC 設定方法"
date: 2024-02-25T0:00:00+09:00
draft: false
summary: "企業で多く利用されている Microsoft 365 (Exchange Online) において、送信者として適切なメールセキュリティを設定する手順を紹介します。"
categories:
- Security
- Config
tags:
- Microsoft 365
- E-mail
---

# Exchange Online における SPF, DKIM, DMARC 設定方法

2024 年 2 月以降、Google を始めとしたビッグテックによるメールサービスに対するメールが reject される可能性があります。

* [Google - メール送信者のガイドライン](https://support.google.com/a/answer/81126)
* [Yahoo! - More Secure, Less Spam: Enforcing Email Standards for a Better Experience](https://blog.postmaster.yahooinc.com/post/730172167494483968/more-secure-less-spam)

Google, Yahoo! を始めとするビッグテックの示すガイドラインには、電子メールにおける `SPF`, `DKIM`, `DMARC` と言ったセキュリティ要素, 及び `Enable Easy Unsubscription（簡単な登録解除を容易にする）` と言った要件がメール送信者に求められています。  
簡単にこれらの要素の内容を示します。

* `SPF` (Sender Policy Framework)
    * `Envelope-From` のドメインについて認証する。
    * `SMTP` で使用される `MAIL FROM` で受信したメールアドレスのドメインを使用して認証する。`TXT` レコードを用いており、レコードに登録されたドメインや IP アドレスを解決した結果を参照して認証を行う。
* `DKIM`（Domainkeys Identified Mail）
    * 送信側で署名された `DKIM-Signature` の `d=` と `s=` の値を用いて、DNS サーバに登録された公開鍵を参照して認証する。
* `DMARC`（Domain-based Message Authentication Reporting and Conformance）
    * `SPF` と `DKIM` の認証に失敗した場合、どのようにメールを取り扱うか指定する。

`SPF` は「SMTP で使用された送信元メールアドレス（`Envelope-From`）のドメインと、解決した送信元 IP アドレスが一致しているか」確認します。`DKIM` は、「送信元の**署名に含まれるドメインに登録されている公開鍵**と、メールヘッダに含まれるデジタル署名」を用いて確認します。即ちここまででは。**人間が通常目にする `From` のメールアドレスから本当に送られているか** という点は検証されていません。  
`DMARC` は `Header-From` と `Envelope-From` を確認し、`SPF` と `DKIM` で認証したドメインとメーラーに表示される `From` ドメインが一致するか確認します。また、認証に失敗した場合は `DMARC` ポリシーに従ってメールを処理します。  
後述しますが、`DMARC` ポリシーの内容を抜粋すると以下のとおりです。

* `p=none`
    * DMARC 認証が失敗した場合でも、受信者に何も動作を望まない（「望まない」ため、受信後の動作は受信者のメールサーバーに依存する点に注意）。
* `p=quarantine`
    * DMARC 認証が失敗した場合、検疫処理を望む。迷惑メールフォルダに入れる等の動作が見込まれるが、それらは受信者のメールサーバーに依存する。
* `p=reject`
    * DMARC 認証が失敗した場合、SMTP トランザクションの途中での拒否を期待する（そもそも受け取らない）。

`DMARC` による認証は `SPF` と `DKIM` を用いますので、これらを適切に設定することで、ビッグテックが提供するメールサービスへの不着を防ぐことができます。それには適切なメールサーバーと DNS レコードの設定が必須となります。  
今回、企業で用いられることが多い Microsoft 365（Exchange Online）における `SPF`, `DKIM`, `DMARC` の設定方法を記します。SendGrid などのメール配信サービスにも応用が可能です。以降での「`DNS サービス`」では、筆者が用いている [Cloudflare](https://www.cloudflare.com/ja-jp/)を用います。

----

[手順]

1. [Microsoft Entra 管理センター](https://entra.microsoft.com/)の `ID` - `設定` - `ドメイン名` - `カスタム ドメインの追加` より、自身の所持する（DNS レコードを設定できる）ドメインを登録します。
2. Microsoft Entra 管理センター上の `ID` - `ユーザー` で、メールを使用するユーザーを作成します。
3. [Microsoft 365 管理センター](https://admin.microsoft.com/) の `ユーザー` - `アクティブなユーザー` より、必要なユーザーに対し Microsoft 365 Business Standard や Exchange Online のライセンスを割り当てます。
4. Microsoft 365 管理センター上で、DMARC Report を受信したいユーザーのメールエイリアスを作成します（任意）。
5. Microsoft 365 管理センター上の `設定` - `ドメイン` - `1.で追加したドメイン` - `DNS レコード` より登録すべき MX, SPF レコードの値を確認します。
6. Cloudflare 上で MX, SPF レコードの設定を行います。
7. [ドメインキー識別済みメール](https://security.microsoft.com/dkimv2) より、DKIM 署名を作成します。
8. Cloudflare 上で DKIM の設定を行います。
9. DMARC ポリシーを検討し、DMARC Report の送信先を 4. で作成したユーザーのメールエイリアスとして DNS サービス上で設定を行います。
10. Gmail 等にテストメールを送信し、メールヘッダから SPF, DKIM, DMARC が設定できていることを確認します。

----

## 1. Microsoft Entra 管理センター - カスタムドメインの追加

1. [Microsoft Entra 管理センター](https://entra.microsoft.com/) `ID` - `設定` - `ドメイン名` - `カスタム ドメインの追加` より、自身の所持する（DNS レコードを設定できる）ドメインを追加します。ここでは `TXT` レコードを利用します。
2. DNS サービス上で TXT レコードを追加します。
3. `1.` の画面に戻り、追加した TXT レコードを Microsoft が確認出来ていることを確認します。

{{< alert info >}}
`MX` レコードではなく `TXT` レコードを追加するのは、宛先アドレスが `ms00000000.msv1.invalid` 等の無効な値のためです。  
後に追加する MX レコードに対し、DNS の伝搬に由来する影響を与えない方が設定手順上望ましいと考えています。
{{< /alert >}}

`TXT` レコードは以下の値となります。

| 項目 | 値 | 備考 |
| ---- | ---- | ---- |
| レコードの種類 | TXT | なし |
| エイリアスまたはホスト名 | "@" | なし |
| 宛先または参照先のアドレス | "MS=ms00000000" | `ms` の後は 8 桁の数字 |
| TTL | 3600 | 1 時間 |

{{< alert info >}}
Entra 管理センター上でドメイン所有権の認証が完了後、このレコードは削除しても問題ありません。
{{< /alert >}}

## 2. Microsoft Entra 管理センター - メールを使用するユーザーの追加

1. [Microsoft Entra 管理センター](https://entra.microsoft.com/) の `ID` - `ユーザー` - `すべてのユーザー` より、必要なユーザーを追加します。

{{< alert warning >}}
Windows 上でログインアカウントとして Microsoft Entra ID を利用する場合、`表示名` が `%HOMEDRIVE%\Users` 配下のユーザーディレクトリ名（例：`C:\Users\表示名`）となります。`OOBE` や `control userpasswords2`、あるいは応答ファイルを用いて Windows のユーザー追加する際、実用上の無用なトラブルを防ぐため表示名を日本語とすべきではありません。  
例えば日本語版コマンドプロンプトは `CP932: Shift_JIS (ANSI)` を標準入力・標準出力としていますが、Windows 11 のメモ帳や Visual Studio Code、あるいは他の OS では UTF-8 が標準です。メモ帳で `*.bat` を作成する時代は過ぎ去りましたが、回避できる文字コードの問題は極力回避すべきでしょう。
{{< /alert >}}

## 3. Microsoft 365 管理センター - ユーザーへのライセンス割り当て

1. `課金情報` - `ライセンス` - `ライセンスの割り当て` より、任意のユーザーに対して `Exchange Online` が含まれたライセンスを割り当てます。

{{< alert info >}}
少数の任意のユーザーに対して特定のライセンスを割り当てたい場合、`ユーザー` - `アクティブなユーザー` から選択して割り当てるのが便利です。
{{< /alert >}}

## 4. Microsoft 365 管理センター - DMARC Report を受信するユーザーのメールエイリアス作成（参考）

DMARC Report を受信するユーザーに対し、メールエイリアスを設定します。通常利用するユーザーに割り当てるべきではありません。`admin@...` や `postmaster@...` と言ったユーザーを用意できない場合、ライセンスを消費せず利用する手段とお考えください。

{{< alert info >}}
Microsoft 365 Business Standard や Exchange Online において、メールエイリアスではライセンスを消費しません。  
> [Microsoft 365 Business サブスクリプション ユーザーの別のメール エイリアスを追加する](https://learn.microsoft.com/ja-jp/microsoft-365/admin/email/add-another-email-alias-for-a-user?view=o365-worldwide)  
> ユーザーあたり最大 400 個のエイリアスを作成できます。 追加の料金やライセンスは必要ありません。 
{{< /alert >}}

1. [Microsoft 365 管理センター](https://admin.microsoft.com/) の `ユーザー` - `アクティブなユーザー` で、DMARC Report を受信したいユーザーの `表示名` 欄右端、縦の `…` より `ユーザー名の設定とメールアドレスの管理` を選択します。
2. 右側タブの `エイリアス` に、DMARC Report を受信したいユーザー名（エイリアス名）を入力して `追加`、`変更の保存` を選択します。

{{< alert info >}}
筆者が利用している Cloudflare では `DMARC Management` と言う機能があります。管理コンソールから様々な情報を確認できます。  
> [Cloudflare DMARC Managementでブランドのなりすましを阻止する](https://blog.cloudflare.com/dmarc-management-ja-jp)
{{< /alert >}}

## 5～6. Microsoft 365 管理センター - MX レコード・SPF レコードの確認

1. [Microsoft 365 管理センター](https://admin.microsoft.com/) の `設定` - `ドメイン` - `1.で追加したドメイン` - `DNS レコード` を確認し、`Microsoft Exchange`, `基本的なモビリティとセキュリティ` 内のレコードを確認します。

`MX` レコードの値は、以下の値となります。

| 項目 | 値 | 備考 |
| ---- | ---- | ---- |
| レコードの種類 | MX | なし |
| エイリアスまたはホスト名 | "@" | なし |
| 参照先のアドレスまたは値 | "contoso.com.mail.protection.outlook.com" | `contoso.com` は登録したドメイン |
| TTL | 3600 | 1 時間 |

`SPF` レコードの値は、以下の値となります。

| 項目 | 値 | 備考 |
| ---- | ---- | ---- |
| レコードの種類 | TXT | `SPF` レコードを追加する。 |
| エイリアスまたはホスト名 | "@" | なし |
| 参照先のアドレスまたは値 | "v=spf1 include:spf.protection.outlook.com -all" | (1) |
| TTL | 3600 | 1 時間 |

{{< alert warning >}}
 (1) `SPF` レコードの定義中で `-all` がデフォルトで設定されています。ここに含まれる `-all` は `spf.protection.outlook.com` （Microsoft 365）以外のメールサーバーからメールを送信した場合は `fail` となり、認証に失敗することを指します。Microsoft 365 以外から直接送信するサービス等がある場合、`include:` の値を適切に設定するか、`~fail`（softfail）に設定する必要があります。しかし例えば CDIR 表記で IP アドレスを指定した場合、アドレスブロックを大きく指定すると `SPF` の認証の意味が無くなるため、ブロックされる場合があります。  
> [SPF（Sender Policy Framework） - 6. SPF認証結果](https://salt.iajapan.org/wpmu/anti_spam/admin/tech/explanation/spf/)  
> [SPFレコードとは（徹底解説）](https://kinsta.com/jp/knowledgebase/spf-record/)
{{< /alert >}}

上記以外の `CNAME` レコードも確認し、DNS サービスへ登録します。

## 7～8. Microsoft Defender - DKIM 署名の追加

Microsoft 365 では、`onmicrosoft.com` ドメインには初期設定で DKIM 署名が作成されていますが、実際に利用するカスタムドメインには作成されていません。  
カスタムドメインの場合、手動で DKIM 署名を利用するよう設定する必要があります。

1. [ドメインキー識別済みメール](https://security.microsoft.com/dkimv2) より、DKIM 署名を追加するカスタムドメインを選択します。
2. `DKIM キーの作成` をクリックし、表示された DNS レコード情報を控えます。

`DKIM` キーは、以下の値となります。

| 項目 | 値 | 備考 |
| ---- | ---- | ---- |
| レコードの種類 | CNAME | (2) |
| エイリアスまたはホスト名 | "selector1._domainkey" | `selector2` も同様。 |
| 参照先のアドレスまたは値 | "selector1-contoso.com._domainkey.contoso123.onmicrosoft.com"| `selector2` も同様。 |
| TTL | 3600 | 1 時間 |

{{< alert info >}}
(2) 受信者側が公開鍵を参照する際、Microsoft の DNS レコードを参照するために `CNAME` で登録します。  
`@contoso.com` から送信したメールは以下の例のように、`d=contoso.com; s=selector1` を参照して `selector1._domainkey.contoso.com` にアクセスします（`[s: selector]._domainkey.[d: domain]` を参照するのは DKIM の仕様です）。  
その後、`CNAME` の値として返された `selector1-contoso.com._domainkey.contoso123.onmicrosoft.com` を参照し、公開鍵を取得します。  
> [DKIM (Domainkeys Identified Mail)](https://salt.iajapan.org/wpmu/anti_spam/admin/tech/explanation/dkim/)  
{{< /alert >}}

{{< alert info >}}
RFC6376 では DKIM は本来 `TXT` レコードで登録すべき仕様となっていますが、現実的に難しいのではないでしょうか。  
> [DNSのDKIMレコードとは？](https://www.cloudflare.com/ja-jp/learning/dns/dns-records/dns-dkim-record/)  
> [DomainKeys Identified Mail (DKIM) Signatures](https://datatracker.ietf.org/doc/html/rfc6376/#section-7.5)
{{< /alert >}}

{{< alert info no-icon >}}
Microsoft 365 メールサーバーから送信した DKIM 署名の例

```text
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed; d=kirisame.tech; s=selector1;
h=From:Date:Subject:Message-ID:Content-Type:MIME-Version:X-MS-Exchange-SenderADCheck; bh=kxVahZVtg5qg1yQk7BpOqEcC2lEfGAF+Ajd2f5kHkkg=;
b=b57z8qh/brot2HpdRLh2j9Nz9OyZ7UVdQ1/e9bRwMmo8Y1gVEPjwqL629OTQzUgH6xGk1OCo2GVa8nPFBZpmBX3AJChi+BcxqwfYZfKUb
+FIyDsGEYcComaUQaGebkXzFYA4rNWuyaTg5UHoBC/SdNSCnVpYQfsCro7nIIJOLxl1VOVGDd3O344IR9s2bmSw9NoO1uneY7P0Rhjh2CalIk6SmMGu2Inlo
l+8WNegdrLjeQDNt17z2q0X2siOWS29RKbptjXBpa+z/DQnDqXh4FSgKUqa37q1N/joP+W8KPPdTMSyC8RM3LDLM4y5xDdxj8kZY0HbRdw9GfzW4qyS6A==
```

{{< /alert >}}

## 9. DMARC ポリシーの検討

以下の資料を参考として、DMARC ポリシーを検討します。

* [DMARC - DMARC を使用してメールを検証する](https://learn.microsoft.com/ja-jp/microsoft-365/security/office-365-security/email-authentication-dmarc-configure?view=o365-worldwide)）
* [DMARC をなめるな](https://creators.bengo4.com/entry/2024/01/22/080000)
* [DMARC レコードの例](https://www.naritai.jp/guidance_dmarc_example.html)

`DMARC` への対応として、最低限必要な設定は以下の通りです（ただしこれは `DMARC` の箱を用意したに過ぎず、中身は存在しないに等しいです）。

| 項目 | 値 | 備考 |
| ---- | ---- | ---- |
| レコードの種類 | TXT | なし |
| エイリアスまたはホスト名 | "_dmarc" | なし |
| 参照先のアドレスまたは値 | "v=DMARC1; p=none;" | (3)  |
| TTL | 3600 | 1 時間 |

(3) は `DMARC` ポリシーの値となり、検討が必要です。

* `v=DMARC1`
    * DMARC のバージョン（`version`）情報であり、現時点では `DMARC1` のみしか指定できません。
* `p=none`
    * ポリシー（`policy`）情報であり、メール受信者に望む認証失敗時の動作です。`none` は何もしないことを受信者に望みます。

次に、追加で指定する必要のあるパラメータを示します。参考としたのは Cloudfrare の `DMARC Management` です。

* `sp=`
    * サブドメイン（`subdomain policy`）におけるポリシー情報であり、メール受信者に望む認証失敗時の動作です。`none` の場合、何もしないことを望みます。
* `rua=`
    * 受信結果の集約レポート（XML）の送信先です。

従って、上記をまとめると登録すべき DMARC の値は以下となります。  
  
`"v=DMARC1;  p=none; sp=reject; rua=mailto:[4. で登録したエイリアス名]@contoso.com"`  
  
ここで、メール受信者に望む動作は `none` であるため**DMARC を有効にしただけの状態に過ぎない**点に注意が必要です。以下に、ポリシーの設定値を示します。

* `p=none`
    * DMARC 認証が失敗した場合でも、受信者に何も動作を望みません（「望まない」だけであるため、受信後の動作は受信者のメールサーバーに依存する点に注意が必要です）。
* `p=quarantine`
    * DMARC 認証が失敗した場合、検疫処理を望みます。迷惑メールフォルダに入れる等の動作が見込まれますが、`none` と同様に受信者のメールサーバーに依存します。
* `p=reject`
    * DMARC 認証が失敗した場合、SMTP トランザクションの途中での拒否されることをを期待します。即ち、そもそもメールを受け取りません）。

## 10. 外部メールサービスを用いた確認

ここまでで `MX レコード`, `SPF`, `DKIM`, `DMARC` を構成しました。最後に設定を行った Microsoft 365 Exchange Online から Gmail にテストメールを送信し、正しく設定が反映されていることを確認します。  

{{< alert info no-icon >}}
メールヘッダ

```text
（中略）
Authentication-Results: mx.google.com;
       dkim=pass header.i=@contoso.com header.s=selector1 header.b="b57z8qh/";
       arc=pass (i=1 spf=pass spfdomain=contoso.com dkim=pass dkdomain=contoso.com dmarc=pass fromdomain=contoso.com);
       spf=pass (google.com: domain of sender@contoso.com designates 2a01:111:f403:201a::601 as permitted sender) smtp.mailfrom=sender@contoso.com;
       dmarc=pass (p=NONE sp=REJECT dis=NONE) header.from=contoso.com
（中略）
```

{{< /alert >}}

上記で（メール送信者としての）メールセキュリティ設定は概ね問題ないと考えています。無論、実用上は EOP (Exchange Online Protection) の利用や、受信するメールの添付ファイルに許可する拡張子の検討、Microsoft の各プランで利用できるセキュリティ設定を検討する必要があります。こちらについてはまたいずれ。

## 参考

* [SPF - SPF を設定して、スプーフィングを防止する](https://learn.microsoft.com/ja-jp/microsoft-365/security/office-365-security/email-authentication-spf-configure?view=o365-worldwide)
* [DKIM - DKIM を使用して、カスタム ドメインから送信される送信電子メールを検証する](https://learn.microsoft.com/ja-jp/microsoft-365/security/office-365-security/email-authentication-dkim-configure?view=o365-worldwide)
* [DMARC - DMARC を使用してメールを検証する](https://learn.microsoft.com/ja-jp/microsoft-365/security/office-365-security/email-authentication-dmarc-configure?view=o365-worldwide)
* [2024年のDMARCレポートとその読み方とは？](https://powerdmarc.com/ja/how-to-read-dmarc-reports/)