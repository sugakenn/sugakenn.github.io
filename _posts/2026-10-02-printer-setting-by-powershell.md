---
layout: post
title: "PowerShellでプリンタ設定"
date: 2026-10-01 22:00:00 +0900
last_modified_at: 2026-10-01
description: Windowsのプリンタ設定をPowerShellで自動化できないか考えてみました
img: 2026/2026-10-01-1.gif # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [windows,printer]
categories: 
  - windows
---


Windowsでの印刷では、用紙サイズ毎に専用の論理プリンターを作っておくと便利ですが、初期設定が面倒です。
そこでPowerShellを使って自動化できないかAIと相談しながら考えてみました。

## プリンタの取得

まずは現状把握の方法です。PowerShellでは`Get-Printer`とすることで現在設定されている、プリンタ一覧を出すことができます。


```powershell
PS > Get-Printer

Name                           ComputerName    Type         DriverName                PortName        Shared   Published  DeviceType
----                           ------------    ----         ----------                --------        ------   ---------  ----------
Microsoft Print to PDF                         Local        Microsoft Print To PDF    PORTPROMPT:     False    False      Print
OneNote for Windows 10                         Local        Microsoft Software Pri... Microsoft.Of... False    False      Print
```

ここで表示されるプリンタ名を使って、プリンタを取得することができます。
```powershell
$printer = Get-Printer -Name "Printer Name"
```
Where-Objectを使ってドライバー単位で取得することも可能です。
```powershell
$driverName="Printer Driver"
$printers = Get-Printer | Where-Object { $_.DriverName -eq $driverName }
```
そのようにして取得したプリンタに対して印刷設定を取得するには次のようにします。
```powershell
$printConf = $printer | Get-PrintCofiguration
```
設定から`プリントチケット`を取得できます。プリントチケットとはどういう設定で印刷するかを保持したものでWindowsではXML形式で保存されています。
```powershell
$printConf.PrintTicketXML
...
<psf:Feature name="PageMediaSize">
    ...
</psf:Feature>

<psf:Feature name="PageOrientation">
    ...
</psf:Feature>
...
```
この設定にはWindowsとして共通定義されている項目とメーカーやプリンタ独自で定義しているものが混在しています。
Windowsの共通項目でない部分を設定をしたい場合にはここを事前に確認します。

## ドライバのインストール

まずはドライバのインストール方法です。GUI環境を操作できるなら手動でインストールしてしまった方が早いと思いますが、自動化したりする場合には有効な方法です。

```powershell
$DriverDir="driverdir"

if (-not (Test-Path -LiteralPath $DriverDir)) {
    throw "ドライバフォルダが見つかりません: $DriverDir"
}

$infFiles = @(Get-ChildItem -LiteralPath $DriverDir -Filter '*.inf' -File -Recurse)
if ($infFiles.Count -eq 0) {
    throw "INF ファイルが見つかりません: $DriverDir"
}

foreach ($inf in $infFiles) {
    Write-Host "Driver Store へ登録: $($inf.FullName)"
    & pnputil.exe /add-driver $inf.FullName /install
    if ($LASTEXITCODE -ne 0) {
        throw "pnputil に失敗しました: $($inf.FullName) (ExitCode=$LASTEXITCODE)"
    }
}
```
ここで出てくる`pnputil.exe`はWindows標準のドライバーパッケージ管理コマンドで、`add-driver`を使って.infファイルを指定することでドライバーのインストールができます。他には削除コマンドの`delete-driver`をはじめ、ドライバ―をフォルダにエクスポートする`export-driver`など様々なオプションがあります。-hをつけて実行するとヘルプが出てきますので興味があれば確認してください。

問題は最近のプリンタードライバーはインストーラーの形式で配布されていることが多いので、どうやって.infを含んだフォルダ形式にするかという部分になると思います。


## 論理プリンタドライバ―の追加
プリンタドライバ―のインストールがおわって、ドライバ―名を取得できるようになればそれを使って論理プリンタドライバ―を追加することができます。

ここでは任意の名前とポート名を設定したプリンタを追加したいと思います。

ポート名はUSBの場合は"USB001"等の名称、スタンダードTCP/IPで作成した場合はその時に付けた名前`IP_192.168.1.100など`を指定します。
```powershell
#インストールされたドライバ名をセット
DriverName="Driver Name"
# 任意のプリンタ名
$PrinterName="Printer Name"
# 用意されているポートの中で使いたいポート名
$PortName="USB001"

$PrinterClass=New-Object System.Management.ManagementClass("Win32_Printer")
$NewPrinter=$PrinterClass.CreateInstance()
$NewPrinter.drivername=$DriverName
$NewPrinter.PortName=$PortName
$NewPrinter.DeviceID=$PrinterName
$NewPrinter.Put()
```

## 設定の変更

Windowsで共通設定になっているものなら、`Set-Printer`や`Set-PrintConfiguration`を使います。
プリンタ名やポート名を変更するなら、Set-Printerを使います。
この時、プリンタの指定には`-Name`を使い、名前を変えたい場合は`-NewName`で指定します。
```powershell
Set-Printer -Name "Printer" -NewName "New-Printer" -PortName "USB002"
```
どのような項目があるかは、Parameters.Keysを取得することでおおよそ分かります。
```powershell
(Get-Command Set-Printer).Parameters.Keys
```
正式なヘルプが欲しいのなら、初回ダウンロード処理が走りますが、
```powershell
Get-Help Set-Printer -Full
```
で確認できます。

Set-PrintConfigurationの方は、用紙サイズ、方向、カラー、両面、などを指定できます。
こちらは-PrinterNameがキーになっています。
```powershell
Set-PrintConfiguration `
    -PrinterName $printerName `
    -PaperSize A4 `
    -Orientation Portrait
```
メーカー固有の設定を変更する場合は、前述のようにプリントチケットをXMLとして取得し、特定キーの値を変更した後で差し替えるようです。なかなか難しい気がします。
先の要領で取得したプリントチケットを、別名のプリンタドライバにセットするぐらいならエラーにならずにセットできました。
```powershell
$ticket = Get-Content "c:\path\to\printticket.xml" -Raw

Set-PrintConfigratoin -PrinterName "Printer" -PrintTicketXml $ticket
```



