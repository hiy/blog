---
author: hiy
datetime: 2020-05-19
title: iPhoneのショートカットアプリとNFCタグで、Hueの照明をオンオフする
slug:
featured: false
draft: true
tags:
  - Hue
  - iPhone
  - ショートカット
  - NFC
ogImage: ""
description: iPhoneのショートカットアプリとNFCタグの連携を試してみました。
---

## はじめに

iPhoneのショートカットアプリとNFCタグの連携を試してみました。
<br />
iPhoneをNFCタグにかざすと、家に設置しているHue照明がオン/オフするようにします。
必要なものは、iPhone、NFCタグの２つで、アプリは「IFTTT」と「ショートカット」です。
<br />※事前に家のHue照明の設定済みです。

![nfctag](https://user-images.githubusercontent.com/2444124/208324769-283818ab-3ebf-4b3b-90a0-805e87cb7fc0.jpg)

![000](https://user-images.githubusercontent.com/2444124/208324773-f69186ac-a3b4-43ac-b237-5a192f54ce07.png)

## IFTTTの設定

IFTTTのAppletを作成します。
Get moreから+ボタンで新規作成します。

![001](https://user-images.githubusercontent.com/2444124/208324781-d85c32ed-9fcb-4526-8da8-0859808de987.png)

![002](https://user-images.githubusercontent.com/2444124/208324790-126c2c2d-e7cd-4da8-886a-5e6e6e132c0b.png)

次に「This」にWebhooksを割り当てます。

![003](https://user-images.githubusercontent.com/2444124/208324795-e7cf2895-5758-47d1-93af-12b520e8474c.png)


![004](https://user-images.githubusercontent.com/2444124/208324798-3cc88e0c-a837-4e03-92b7-dcd6fc89e08b.png)


イベントに名前をつけます。この名前がWebhooksイベントのトリガーになるので、わかりやすい名前をつけておきます。
ここでは「toggle\_home\_lights」としました。

![005](https://user-images.githubusercontent.com/2444124/208324817-8e0a3c32-578a-4f1d-a7dd-c4ddc7811d6d.png)

次に「That」にHueを割り当てます。

![006](https://user-images.githubusercontent.com/2444124/208324826-3aa76c71-1918-4e65-a593-e4e3d4288071.png)
![007](https://user-images.githubusercontent.com/2444124/208324832-c601e1a1-c5ec-4dda-a7c4-a56b311c7bde.png)

Hueに照明をオン/オフするアクションを割り当てます。

![008](https://user-images.githubusercontent.com/2444124/208324837-29107aa6-e950-422c-9178-dab23b6b96b4.png)

IFTTTからHueにログインして、どのライトをオン/オフするかを選択します。

![009](https://user-images.githubusercontent.com/2444124/208324843-c69523d0-2fff-42d1-b830-7a145ddc1da7.png)


これでWebhooksとHueの設定は完了です。
<br/>
一旦、WebhooksとHueが正しく連携されていることを確認してみます。
「Get more」から「Documentation」を選択するとイベントをテストできます。

![010](https://user-images.githubusercontent.com/2444124/208324850-97087d8b-d899-4cd2-8ee8-c9d0e7b32722.png)

![011](https://user-images.githubusercontent.com/2444124/208324852-649e6982-d711-47ee-9437-1646cb52e187.png)


イベント名を「toggle\_home\_lights」を画面のテキストボックスに入力して、
「Test it」ボタンをタップします。
タップするとHueの照明がオン/オフすることが確認できると思います。

![012](https://user-images.githubusercontent.com/2444124/208324856-5fe1382b-325f-438d-8f02-c121b5f08192.png)

ここで「ショートカット」アプリの設定で使用するためにURLをコピーしておきます。

![013](https://user-images.githubusercontent.com/2444124/208324860-a8cb98b7-e3c7-4c32-b4fa-f5c3325f969c.png)

## ショートカットでオートメーションの作成

IFTTTで設定したWebhookをNFCタグと連携します。
ショートカットアプリのオートメーションのタブを選択して、右上の+ボタンをタップします。

![014](https://user-images.githubusercontent.com/2444124/208324864-1fba867b-1a62-4bb5-9ebf-170e95beea78.png)

個人用オートメーションからNFCを選択します。

![016](https://user-images.githubusercontent.com/2444124/208324869-02465e62-0e89-4cbe-8db7-2dcf7d232d9d.png)

![017](https://user-images.githubusercontent.com/2444124/208324872-dc11e3d2-610b-4ed8-b934-517078defa5a.png)

iPhoneの先端をNFCタグにかざしてスキャンして、タグに名前をつけます。
ここでは「電気」にしました。

![019](https://user-images.githubusercontent.com/2444124/208324877-dfa61dc5-6e58-42e5-a706-f13fafa9c362.png)

![021](https://user-images.githubusercontent.com/2444124/208324881-062e7aea-f63f-4816-8506-1b7899ad6cda.png)

「次へ」をタップして、
NFCタブにかざした時のアクションを追加します。
IFTTTのアプリでコピーしておいたURLにアクセスするために「Web」を選択します。

![022](https://user-images.githubusercontent.com/2444124/208324886-5b73992a-1340-4087-9862-840656df6b57.png)

![023](https://user-images.githubusercontent.com/2444124/208324889-bf042291-43cb-4428-a0b1-5e3c2d2e5cf4.png)

「URLの内容を取得」を選択して、IFTTTのアプリでコピーしておいたURLを貼り付けます。
<br />
「次へ」をタップして、オートメーションの作成を確定します。

![024](https://user-images.githubusercontent.com/2444124/208324897-2a3a8015-8a01-42fe-b795-39ec230e8d88.png)

![025](https://user-images.githubusercontent.com/2444124/208324901-01e708f0-d310-4713-a438-a3b2534afb86.png)

最後に、毎回実行時に確認を挟むと面倒なので、
作成したオートメーションの「実行の前に尋ねる」は、オフにしておきます。

![028](https://user-images.githubusercontent.com/2444124/208324906-7689853f-90e6-44f0-a788-0ce8ff8e14f7.png)

完了すると赤枠のようになります。

![029](https://user-images.githubusercontent.com/2444124/208324911-49ff7024-78a5-410d-8b59-210d03470b32.png)

以上で、設定は完了です。
この状態で、iPhoneの先端をNFCタグにかざすことで、Hue照明がオン/オフすることができます。

iPhoneのショートカットアプリを使うことで、簡単にNFCタグと連携できることが確認できました。今後は他の使い道も考えてみようと思います。
