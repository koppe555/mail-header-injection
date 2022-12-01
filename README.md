
# メールヘッダインジェクション


## 概要
<img width="436" alt="image" src="https://user-images.githubusercontent.com/44897118/204989661-1b059278-2c02-4419-af9a-a0dd102ef07a.png">

例えば、お問合せ用のフォームなどでメールを送信する機能があるとする。

サイトAのお問合せフォームで記述しsubmitすると、サイトAを運営する管理者にメールが送信される仕組みになっているとする。当然管理者以外にお問合せの内容を送ることはないので、ことのき宛先は管理者のみに固定されている。しかしメールヘッダインジェクションでは固定されている宛先以外にも送ることができ、迷惑メールとして利用されてしまう。


サイボウズなどの有名企業でも実際に脆弱性が見つかった（コーポレートサイトからは情報が消されている）
https://cs.cybozu.co.jp/information/20131202up01.html



## 仕組み



メールのヘッダと本文の区別が一行空いてることで判定している。意図的に改行を入れてヘッダを偽造して攻撃する。

メールヘッダの内容が外部からの入力に依存する場合に脆弱性となる。


**メールヘッダ一覧**

<img width="435" alt="image" src="https://user-images.githubusercontent.com/44897118/204989707-e916284f-6a2c-4df6-aacf-66c784a7cf3d.png">

https://atmarkit.itmedia.co.jp/fnetwork/rensai/netpro03/mail-header.html

メールヘッダのTo, Cc, Bccなどに迷惑メール送信対象のアドレスを入れる。


## 具体例

参照
https://www.invicti.com/learn/email-injection/

こんな感じの簡単なメールフォームがあったとする

```
<?php
if(isset($_POST['name'])) {
$name = $_POST['name'];
$replyto = $_POST['replyTo'];
$message = $_POST['message'];
$to = 'root@localhost';
$subject = 'My Subject';
// Set SMTP headers
$headers = "From: $name \n" .
"Reply-To: $replyto";
mail($to, $subject, $message, $headers);
}
?>
```

**POSTリクエスト**
```
# 通常のPOST
POST /contact.php HTTP/1.1
Host: www.example2.com
name=Anna Smith&replyTo=anna@example.com&message=Hello

# 攻撃者のPOST
POST /contact.php HTTP/1.1
Host: www.example2.com
name=Best Product\r\nbcc: everyone@example3.com&replyTo=blame_anna@example.com&message=Buy my product!
```

**メールヘッダ**
```

# 通常のメールヘッダ
name: Best Product
replayTo: blame_anna@example.com
message: Buy my product!

# 攻撃者のメールヘッダ
name=Best Product
bcc: everyone@example3.com
replyTo=blame_anna@example.com
message=Buy my product!
```

`name=Best Product\r\nbcc: everyone@example3.com` nameの中に改行コード(\r\n)を入れてbccを追加している。


SMTPは全て文字列のやり取りで行われている。

```
← 220 mail.xxx.yyy.zzz ESMTP
→ HELO mail.xxx.yyy.zzz      ← サーバーとのあいさつ
← 250-OK
← 250 WELCOME
→ MAIL FROM: aaa@xxx.yyy.zzz ← 発信元
← 250 OK
→ RCPT TO: bbb@xxx.yyy.zzz   ← 宛先
← 250 OK
→ DATA                       ← メールの開始
← 354 Start mail input; end with <CRLF>.<CRLF>
→ From: aaa@xxx.yyy.zzz      ← 発信元
→ To: bbb@xxx.yyy.zzz        ← 宛先
→ Subject: TEST              ← 件名
→                            ← ヘッダと本文の間には空行
→ Hello!!
→ Good by!!
→ .                          ← メールの終了
← 250 OK
→ QUIT                       ← SMTPの終了
← 221 BYE
```

https://www.tohoho-web.com/ex/draft/smtp.htm



## 対策

- メール系のライブラリ、最新バージョンを使う
  - 自作しない（する場合は以下の対策をする）
- 入力値に対して改行コードを削除する
- メールヘッダを固定値にして、外部からの入力は全てメール本文に出力する

```
from: hoge@exeample.com
to: fuga@example.com
bcc: everyone@example3.com
subject: this is subject

my product! ←この部分のみ入力値にする
```




