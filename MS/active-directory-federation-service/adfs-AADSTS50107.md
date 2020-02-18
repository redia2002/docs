---
title: AADSTS50107 The requested federation realm object 'xxx' does not exist. (Issuer ID/Issuer URI と SupportMultipleDomain)
date: 2020-02-20
tags:
  - AD FS
  - AADSTS50107
  - Issuer ID
  - Issuer URI
  - supportMultipleDomain
---

# AADSTS50107 The requested federation realm object 'xxx' does not exist.
# --- Issuer ID / Issuer URI と SupportMultipleDomain ---

こんにちは、Azure & Identitiy サポートチームの竹村です。
今回は、多くのお客様からお問合せをいただく AADSTS50107 のエラーについて、既知の事例や対応方法をご紹介します。

## 1. ADSTS50107  と  Issuer ID、 Issuer URI

最初に、AADSTS50107 のエラーの内容と、その背景にある Issuer ID、および Issuer URI と呼ばれるものについて説明します。<br>
<br>
Issuer ID は、Azure AD と連携する IdP (例えば AD FS など) が発行するクレームです。<br>
AD FS (WS-Federation) の場合は、具体的には以下のクレームになります。<br>

```
http://schemas.microsoft.com/ws/2008/06/identity/claims/issuerid
```
一方、Azure AD のカスタムドメインには、フェデレーションの設定を行う際に Issuer URI というものが設定されます。<br>
AD FS の場合、通常は  Convert-MsolDomainToFederated コマンドレットや、Update-MsolFederatedDomain コマンドレットを使用してフェデレーションの設定を行いますが、この Powershell の処理で Issuer URI が設定されます。<br>
<br>
Azure AD でのフェデレーション認証の要件として、IdP が発行する Issuer ID が、認証されたユーザーのドメインに設定されている Issuer URI と合致している必要があります。<br>
AADSTS50107 は、IdP が発行した Issuer ID と、Azure AD のカスタムドメインにセットされている Issuer URI とが、合致しなかったことを示すエラーです。<br>

![](./adfs-AADSTS50107/adfs_AADSTS50107_01.png) 

上記のように、エラー画面には "realm object" と "does not exist." の間に文字列が表示されますが、これが IdP が発行した Issuer ID になります。<br>
そして、Azure AD のカスタムドメインには、ここに表示されている文字列の Issuer URI が存在していないことを示しています。<br>


## 2. -supportMultipleDomain オプションについて

次に、AADSTS50107 のエラーと密接に関わり合う -supportMultipleDomain オプションについて説明します。<br>
AD FS で Convert-MsolDomainToFederated コマンドレットや、Update-MsolFederatedDomain コマンドレットを使用してフェデレーションの設定を行う時に、-supportMultipleDomain というオプションを指定することがあります。<br>
これは、1つの AD FS ファームと、複数のカスタム ドメインの間でフェデレーションの設定を行う場合に必要となるオプションです。<br>
この背景には、カスタム ドメインに設定する Issuer URI は、重複が許可されず必ず一意にする必要がある、という Azure AD の要件があります。<br>
<br>
AD FS と Azure AD のカスタム ドメインが 1 対 1 の場合には、-supportMultipleDomain オプションは必要ありません。<br>
-supportMultipleDomain オプションを指定しない場合、 Convert-MsolDomainToFederated コマンドレットや、Update-MsolFederatedDomain コマンドレットは、AD FS の識別子 (http://<フェデレーションサービス名>/services/trust) を Issuer URI としてカスタム ドメインに設定します。<br>
AD FS 側では Issuer URI を発行するクレームルールは作成されません。<br>
ルールがない場合、AD FS は既定で Issuer URI として AD FS の識別子 (http://<フェデレーションサービス名>/services/trust)  をクレームにセットするため、Issuer URI と合致し、要件を満たすことができます。<br>
<br>
-supportMultipleDomain オプションを指定すると、Azure AD のカスタムドメイン側には、http://<カスタムドメイン>/services/trust/  という値が Issuer URI にセットされます。<br>
例えば、a.com と b.com という 2 つのドメインに対して、-supportMultipleDomain オプションを指定して 1つの AD FS ファームとフェデレーションを構成すると、それぞれのドメインには以下の Issuer URI がセットされます。<br>

```
a.com
http://a.com/services/trust/

b.com
http://b.com/services/trust/
```
一方で、AD FS 側には -supportMultipleDomain オプションを指定すると、以下のように認証するユーザーの UPN に応じて Issuer ID をを発行するルール (発行返還規則) が作成されます。

```
c:[Type == "http://schemas.xmlsoap.org/claims/UPN"]
 => issue(Type = "http://schemas.microsoft.com/ws/2008/06/identity/claims/issuerid", Value = regexreplace(c.Value, ".+@(?<domain>.+)", "http://${domain}/adfs/services/trust/"));
```
例えば、user01@a.com というユーザーで認証した場合、上記のルールによって UPN の @ 以下のドメイン部分が抽出されて、`http://a.com/adfs/services/trust/` という値が Issuer ID にセットされます。<br>
また、user02@b.com というユーザーで認証した場合、同様に `http://b.com/adfs/services/trust/` という値が Issuer ID にセットされます。<br>
<br>
このようにして、各ドメインの Issuer ID と IssuerURI の整合性を取ることを意図したものが、-supportMultipleDomain オプションの正体です。<br>
<br>
なお、1 対 1 の場合であっても、-supportMultipleDomain オプションを指定しても問題はありません。<br>
例えば、上記の例で a.com しか存在しない場合でも Issuer ID と Issuer URI の整合性は保たれます。<br>
Azure AD Connect のウィザードで AD FS を構成、管理している場合には、1 対 1 の場合にも -supportMultipleDomain  オプションを指定してフェデレーションが構成されます。<br>

## 3. トップレベルドメインとサブドメイン、エラー発生のシナリオについて

Azure AD では、サブドメインを作成することができます。<br>
例えば、a.com というドメインのサブドメインとして、sub1.a.com を作成することができます。<br>
このサブドメインの取り扱いは少々複雑で、注意が必要です。<br>
具体的には、親子関係が存在する場合と、存在しない場合とに分けられます。<br>

### (1) サブドメインを先に作成した場合
最初に sub1.a.com を作成し、後から a.com を作成した場合、親子関係は結ばれません。<br>
両方がトップレベルドメインとして扱われ、sub1.a.com と  a.com  は別々のドメインとみなされ、それぞれ固有のフェデレーションの設定を保持することができます。<br>
このケースでは、既定のルールでも問題は発生しません。<br>
上述の a.com と b.com の場合と同様の動作になります。<br>
ただし、さらにここから sub2.a.com を作成しますと、sub1.a.com と a.com は別々のドメインですが、a.com と sub2.a.com との間には親子関係が結ばれることになります。<br>
この場合、後述の「(2) サブドメインを後から作成した場合」を考慮して、ルールを書き換える必要があります。<br>

### (2) サブドメインを後から作成した場合
最初に a.com を作成し、後から sub1.com を作成した場合、a.com と sub1.a.com との間には「親子関係」が結ばれます。<br>
親子関係が結ばれる場合、子 (サブ) ドメインの認証方式は、すべて親ドメインの設定を継承します。<br>
例えば、親ドメインがマネージド ドメインであれば子もマネージドになり、親ドメインがフェデレーション ドメインであれば子もフェデレーションになります。<br>
フェデレーションの設定も親ドメインのものを継承します。<br>
ここで注意が必要なのが、Issuer URI の設定についても親ドメインしか持たないということです。<br>
つまり、子ドメインのユーザーで認証した場合でも、親ドメインに設定されている Issuer URI に合致する Issuer ID を AD FS で発行する必要があります。<br>
<br>
AD FS と 親ドメインが 1 対 1 で、-supportMultipleDomain を指定していない場合には、問題は生じません。<br>
親ドメインであっても、子ドメインであっても、既定の AD FS の識別子 (`http://<フェデレーションサービス名>/services/trust`) が固定で Issuer ID にセットされ、親ドメインにも同一の値が Issuer URI としてセットされているためです。<br>
<br>
-supportMultipleDomain を指定している場合には、既定のルールでは問題が発生します。<br>
例えば、user01@sub1.a.com というユーザーで認証した場合を考えます。<br>
この時、既定のルールですと、UPN の @ 以下のドメイン部分が抽出されて、`http://sub1.a.com/adfs/services/trust/` という値が Issuer ID にセットされます。<br>
しかし、親ドメインの Issuer URI は `http://a.com/adfs/services/trust/` ですので合致しません。<br>
<br>
AADSTS50107 のエラーメッセージに、`http://sub1.a.com/adfs/services/trust/` のような、サブドメインの UPN からそのまま抽出して作成された値が表示されている場合、ほとんどがこのケースに該当します。<br>
<br>
以下の既定のルールを削除して<br>

```
c:[Type == "http://schemas.xmlsoap.org/claims/UPN"]
 => issue(Type = "http://schemas.microsoft.com/ws/2008/06/identity/claims/issuerid", Value = regexreplace(c.Value, ".+@(?<domain>.+)", "http://${domain}/adfs/services/trust/"));
```

以下のようなルールに書き換えることで対処することができます。

```
c:[Type == "http://schemas.xmlsoap.org/claims/UPN"]
=> issue(Type = "http://schemas.microsoft.com/ws/2008/06/identity/claims/issuerid", Value = regexreplace(c.Value, "(?i)(^([^@]+)@)(.*)(?<domain>(a\.com|b\.co\.jp|c\.net))$", "http://${domain}/adfs/services/trust/"));
```

上記は、a.com、b.co.jp、c.net というトップレベルドメインと、その子ドメインが存在する環境に対応したルールです。<br>
例えば、以下のような構成に対応します。<br>

```
a.com (親 ドメイン)
   sub1.a.com (子ドメイン)
   sub2.a.com (子ドメイン) 

b.co.jp (親 ドメイン)
   sub1.b.co.jp (子ドメイン)
   sub2.b.co.jp (子ドメイン) 

c.net (親 ドメイン)
   sub1.c.net (子ドメイン)
   sub2.c.net (子ドメイン) 
```
ルールに列挙したドメイン  (\ は . をエスケープしています) と、その親子関係のあるサブドメインすべてについて、それぞれの親ドメインに相当する部分を抽出して Issuer ID をセットします。<br>
<br>
なお、Azure AD Connect で AD FS を構成、管理している場合には、自動で以下のようなルールが作成されます。<br>

```
c1:[Type == "http://schemas.xmlsoap.org/claims/UPN"] && c2:[Type == "http://schemas.microsoft.com/ws/2012/01/accounttype", Value == "User"] 
=> issue(Type = "http://schemas.microsoft.com/ws/2008/06/identity/claims/issuerid", Value = regexreplace(c1.Value, "(?i)(^([^@]+)@)(.*)(?<domain>(a\.com|b\.co\.jp|c\.net))$", "http://${domain}/adfs/services/trust/"));
```
しかし、黄色くハイライトした部分が抜け落ちており、子ドメインに対応することができません。<br>
Azure AD Connect で構成、管理している場合で、子ドメインが存在する場合には、(.*) を追記してください。<br>
AADSTS50107 のエラーメッセージに子ドメインのユーザーの UPN がそのまま表示されている場合、ほとんどがこちらのケースに該当します。<br>

## 4. より複雑な環境におけるルールの作成例
例えば、sub1.a.com、a.com、sub2.com の順番にドメインを作成し、また他に b.co.jp、sub1.b.co.jp  の順番でドメインを作成したとします。<br>
以下のような構成になります。<br>

```
sub1.a.com (トップレベル ドメイン)

a.com (トップレベル ドメイン)
&nbsp&nbsp+--&nbsp&nbspsub2.a.com (a.com の 子ドメイン)

b.co.jp (トップレベル ドメイン)
&nbsp&nbsp+--&nbsp&nbspsub1.b.co.jp (b.co.jp の子ドメイン)
```

この時、それぞれのドメインでセットすべき Issuer ID は以下のようになります。

```
sub1.a.com&nbsp&nbsp: 
&nbsp&nbsp`http://sub1.a.com/adfs/services/trust/`

a.som および sub2.a.com&nbsp&nbsp:  
&nbsp&nbsp`http://a.com/adfs/services/trust/`

b.co.jp および sub1.b.co.jp&nbsp&nbsp:
&nbsp&nbsp`http://b.co.jp/adfs/services/trust/`
```

このように、親子関係を持たないサブドメインと、親子関係を持つサブドメインの両方が存在する場合には、個別にドメインを判定して適切な Issuer ID を送信するルールが必要になります。<br>
上記のような構成では、以下のようなルールを作成します。<br>

```
@ UPN サフィックスをそのまま "http://domain/upnsuffix" というクレームとして発行
c:[Type == "http://schemas.xmlsoap.org/claims/UPN"]
 => issue(Type = "http://domain/upnsuffix", Value = regexreplace(c.Value, "(^([^@]+)@)(?<domain>(.*))$", "${domain}"));

@ sub1.a.com の場合
c:[Type == "http://domain/upnsuffix", Value =~ "^(?i)sub1\.a\.com$"]
 => issue(Type = "http://schemas.microsoft.com/ws/2008/06/identity/claims/issuerid", Value = "http://sub1.a.com/adfs/services/trust/");

@ a.com およびその子ドメインの場合 (sub1.a.com はトップレベルドメインなので除く)
c1:[Type == "http://domain/upnsuffix", Value =~ "(?i)a\.com$"]  && c2:[Type == "http://domain/upnsuffix", Value =~ "^(?!(?i)sub1.a.com$)"]
 => issue(Type = "http://schemas.microsoft.com/ws/2008/06/identity/claims/issuerid", Value = "http://a.com/adfs/services/trust/");

@ b.co.jp およびその子ドメインの場合
c:[Type == "http://domain/upnsuffix", Value =~ "(?i)b\.co.jp$"]
 => issue(Type = "http://schemas.microsoft.com/ws/2008/06/identity/claims/issuerid", Value = "http://b.co.jp/adfs/services/trust/");
```

いかがでしたでしょうか。<br>
上記内容が少しでも参考となりますと幸いです。<br>
<br>
<br>
製品動作に関する正式な見解や回答については、お客様環境などを十分に把握したうえでサポート部門より提供させていただきますので、ぜひ弊社サポート サービスをご利用ください。<br>
<br>
※本情報の内容（添付文書、リンク先などを含む）は、作成日時点でのものであり、予告なく変更される場合があります。
