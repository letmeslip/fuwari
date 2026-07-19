---
title: RTX830へシリアルコンソール接続するまで
published: 2026-07-19
description: "安価なUSBコンソールケーブルを使い、WindowsとMacBookからRTX830へシリアル接続するまでの手順をまとめます。"
image: ""
tags: ["RTX830", "YAMAHA", "Windows", "PuTTY", "シリアルコンソール"]
category: ネットワーク
draft: true
lang: ja
---

お久しぶりです。

全5回を予定していた自宅サーバー監視シリーズを少し放置して、最近はYAMAHA RTX830という新しいおもちゃを手に入れ、専らいじって遊んでいます。

RTX830はLAN経由でSSH接続すればCLIへログインできるので、普段の操作はとても便利です。ただし、ネットワーク設定を間違えて「自閉モード」にしてしまうと、LAN側からアクセスできなくなり、最悪の場合は初期化が必要になる可能性があります。

そこで、いざというときにも物理コンソールから操作できるよう、Amazonで安価なUSBコンソールケーブルを購入しました。

- [今回購入したUSBコンソールケーブル（Amazon）](https://www.amazon.co.jp/dp/B0991WNST6)

ケーブルを挿せばすぐに使えると思っていたのですが、WindowsにCOMポートとして認識されず、一筋縄ではいきませんでした。結論から言うと、今回はFTDIのドライバーを手動でインストールする必要がありました。

同じところでつまずいたときに手順を忘れないよう、自分用のメモも兼ねて記事に残しておきます。

## この記事の流れ

大まかな流れは次のとおりです。

1. USBコンソールケーブルを接続し、COMポートを確認する
2. FTDI CDMドライバーをインストールする
3. PuTTYへRTX830用のシリアル接続設定を登録する
4. Windows Terminalから簡単に呼び出せるようにする
5. RTX830へログインする
6. MacBookからシリアル接続する

まずはWindowsでの手順から進めます。

> ケーブルに搭載されているチップによって、必要なドライバーは異なる場合があります。この記事では、上記のケーブルで使用した手順を紹介します。

## 1. COMポートが表示されているか確認する

USBコンソールケーブルをRTX830とWindows PCへ接続し、デバイスマネージャーを開きます。

今回の環境では「ポート（COMとLPT）」が表示されず、USBシリアルポートとして認識されていませんでした。

![ドライバー導入前のデバイスマネージャー](./01-device-manager-no-com-port.png)

この状態ではPuTTYから接続できないため、ケーブルに搭載されているFTDIチップ用のCDMドライバーをインストールします。

## 2. FTDI CDMドライバーをインストールする

ダウンロードしたFTDI CDM Driversのインストーラーを起動します。今回使用したバージョンは`2.12.36.20`です。

最初の画面で「Extract」をクリックし、ドライバーパッケージを展開します。

![FTDI CDM Driversの展開画面](./02-ftdi-driver-extract.png)

デバイスドライバーのインストールウィザードが起動したら、「次へ」をクリックします。

![デバイスドライバーのインストールウィザード](./03-ftdi-install-wizard-start.png)

使用許諾契約を確認し、「同意します」を選択して「次へ」をクリックします。

![FTDIドライバーの使用許諾契約](./04-ftdi-license-agreement.png)

インストールが完了するまで待ちます。2項目とも状態が「使用できます」になったことを確認し、「完了」をクリックします。

![FTDI CDMドライバーのインストール完了](./05-ftdi-install-complete.png)

## 3. COMポート番号を確認する

ドライバーのインストール後、デバイスマネージャーを開き直します。

「ポート（COMとLPT）」を展開すると、`USB Serial Port (COM4)`が表示されました。

![デバイスマネージャーでUSB Serial PortのCOM番号を確認](./06-device-manager-com4.png)

このCOM番号は、後ほどPuTTYの接続設定で使用します。割り当てられる番号は環境によって異なるため、自分のPCに表示された番号を確認してください。

PowerShellから確認する場合は、次のコマンドを実行します。

```powershell
Get-CimInstance Win32_SerialPort |
    Format-Table DeviceID, Name, Description -AutoSize
```

## 4. PuTTYをインストールする

シリアル接続に使用するターミナルソフトとして、今回はPuTTYを使用します。

[PuTTYの公式ダウンロードページ](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)を開き、`MSI ('Windows Installer')`にある`64-bit x86`をダウンロードします。

今回インストールしたバージョンはPuTTY `0.84 (64-bit)`です。インストーラーを起動し、「Next」をクリックします。

![PuTTYセットアップウィザードの開始画面](./07-putty-installer-start.png)

インストール先を指定します。今回は初期値の`C:\Program Files\PuTTY\`から変更せず、「Next」をクリックしました。

![PuTTYのインストール先](./08-putty-install-folder.png)

インストールする機能を確認します。今回は初期設定のまま「Install」をクリックしました。

![PuTTYのインストールオプション](./09-putty-install-options.png)

セットアップ完了画面が表示されたら、「Finish」をクリックします。READMEが不要な場合は、`View README file`のチェックを外しても構いません。

![PuTTYのインストール完了](./10-putty-install-complete.png)

## 5. PuTTYのシリアル接続設定を保存する

PuTTYを起動し、`Session`画面で次のように設定します。

- Serial line：`COM4`
- Speed：`9600`
- Connection type：`Serial`

`Serial line`には、デバイスマネージャーで確認したCOM番号を入力してください。

![PuTTYでCOM4と9600bpsを設定](./11-putty-serial-session-com4.png)

続いて、`Connection` → `Serial`を開き、RTX830の初期値に合わせて次のように設定します。

- Speed：`9600`
- Data bits：`8`
- Stop bits：`1`
- Parity：`None`
- Flow control：`XON/XOFF`

毎回入力する手間を省くため、`Saved Sessions`へ設定を保存します。今回は`COM4`という名前で保存しました。

次回以降は、保存したセッションを選択して「Load」をクリックすれば、同じ設定を呼び出せます。

## 6. Windows Terminalから接続できるようにする

PuTTYから直接接続するだけであれば、ここまでの設定で十分です。

今回は、Windows TerminalのドロップダウンからRTX830のコンソールを開けるように、Plinkを呼び出すプロファイルも追加しました。

Windows Terminalの設定を開き、新しいプロファイルを追加します。

- 名前：`Serial-COM4`
- コマンドライン：`C:\Program Files\PuTTY\plink.exe -load "COM4"`
- アイコン：任意（今回は`plink.exe`を指定）

![Windows Terminalに追加したSerial-COM4プロファイル](./12-windows-terminal-plink-profile.png)

`-load`には、PuTTYの`Saved Sessions`へ保存したセッション名を指定します。

設定を保存すると、Windows Terminalのドロップダウンから`Serial-COM4`を選ぶだけで、RTX830のコンソールを開けるようになります。

## 7. RTX830へログインする

Windows Terminalから`Serial-COM4`プロファイルを起動し、Enterキーを押します。

最初に`Password:`と表示されるので、何も入力せずにEnterキーを押します。すると、`Username:`が表示されます。

![パスワードを空のまま送信すると表示されるUsernameプロンプト](./13-rtx830-username-prompt.png)

`Username:`へ管理者ユーザー名を入力してEnterキーを押します。続いて表示される`Password:`へ、そのユーザーのパスワードを入力します。

> パスワードを入力しても、画面上には文字や記号が表示されません。そのまま入力してEnterキーを押してください。

認証に成功すると、RTX830の機種名、ファームウェアリビジョン、メモリ容量などが表示され、最後に`>`プロンプトが現れます。これでシリアルコンソールからのログインは完了です。

![RTX830へシリアルコンソールでログイン成功](./14-rtx830-login-success.png)

今回の環境では、`RTX830 Rev.15.02.33`が動作していることも確認できました。

## おわりに

USBコンソールケーブルを接続してもCOMポートが表示されない場合は、まずケーブルに対応したドライバーがインストールされているか確認してください。

ドライバーを導入してCOM番号を確認したあとは、PuTTYへRTX830のシリアル設定を登録すれば接続できます。さらにPlinkをWindows Terminalのプロファイルへ登録しておくと、次回から簡単にコンソールを開けます。

次は、ログアウト方法とMacBookから接続する方法も追記する予定です。
