# UPDI_HV_WRITER-w-RESET
This is a new AVR ATTINY series UPDI programmer with HV pulse injection avility on power on reset timing.
<!-- ![TINY202_IR_REMOTE 2024-05-02 233458]https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/7ae2ed78-f5d1-4399-9f40-e7d13989e965-->
<img src="https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/7ae2ed78-f5d1-4399-9f40-e7d13989e965"  width = "480">  

## はじめに
### USB Serial変換基板を使ったUPDI対応プログラマー
AVR New ATTINY 0/1/2 シリーズは旧ATTINY13,85等より性能が上がって、値段も手ごろなので
これを気軽に使いたいと思いました。  
しかし、書き込み方式が今までの3線ICSPから1線のUPDI方式にかわり、いままでのAVR ISP MK2とか
の安価な書き込み機が使えなくなりました。ATMEL ICEとかメッチャ高価ですし。  
また、手持ちの中華製のパラレルプログラマーTL866-IIも対応していないので、高電圧プログラム
によるFUSE書き替えもできなくなり、不便になりました。  
そのような背景で、高電圧プログラムに対応したUPDI対応のプログラマーを作りました。
下のステップで開発を行いました。  
1. 中華製USB Serial変換モジュールを使ったUPDI対応のプログラマーを作る。  
2. UPDI線にPOR(パワーオンリセット)時に12Vパルスを印加できるタイミング回路を設計する。    
3. 手作り試作で動作を検証した後、基板を起こして正式に製作する。  
4. 
### このプロジェクトの概要
HV対応のUPDIプログラマーは各種市場に出ています。代表的なのは  
Adafruitさんの [Adafruit High Voltage UPDI Friend](https://blog.adafruit.com/2024/02/29/coming-soon-adafruit-high-voltage-updi-friend-usb-serial-updi-programmer/)  
wagiminatorさんの[SerialUPDI_HV_Programmer](https://github.com/wagiminator/AVR-Programmer/tree/master/SerialUPDI_HV_Programmer)  
technoblogyさんの [Tiny UPDI-HV Programmer](http://www.technoblogy.com/show?48MP)  
といったところでしょうか？
  
これらの方式の共通的なところは、プログラマーをターゲット機器にUPDIで接続して、非同期にUPDI線にHVP(12Vパルス)を与える
所だと思います。  

これに対して、今回採用した名張市つつじが丘おもちゃ病院の大泉院長の方式では、POR(パワーオンリセット)後に HVPを与えるところが違います。  
[ｔｉｎｙＡＶＲプログラマー（インタフェース製作）](http://tutujith.blog.fc2.com/blog-entry-729.html)  

さて、TINY202/402のデータシートを見てみましょう。    
<img src= "https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/d541ac43-7fc5-4661-946c-c52dff27c6bc" width = "640">  
**日本語版 30.3.2.1.3. RESETﾋﾟﾝの高電圧無効化でのUPDI許可**   

HVPの印加の際は先にリセットすること。/RESET信号が"H"に戻ってからで、HVPのパルス幅は100～1ｍS以内とのことですね。  

次の章には、下記のようにHVPとPORのタイミングについて明確に記されています。  
**30.3.2.1.4. 汎用入出力(GPIO)に対する出力許可計時器保護  
ｼｽﾃﾑ構成設定0(FUSE.SYSCFG0)ﾋｭｰｽﾞのRESETﾋﾟﾝ構成設定(RSTPINCFG)ﾋﾞｯﾄが’00’の時に、RESETﾋﾟﾝは汎用入出力(GPIO)と
して構成設定されます。GPIOが出力を活動的に駆動するのとUPDI高電圧(HV)許可手順開始の間での潜在的な衝突を避けるた
め、GPIO出力駆動部はｼｽﾃﾑ ﾘｾｯﾄ後に最小8.8ms間禁止されます。
HVﾌﾟﾛｸﾞﾗﾐﾝｸﾞ手順に入るのに先立って常に電源ONﾘｾｯﾄ(POR)を発行することが推奨されます。**

前の2方式は、USBSerialモジュールのRTS/DTR信号がプログラム前に "L" に下がるタイミングでHVP印加しています。  
したがってこの記述とは関係なく、リセットとは非同期にATTINYのRESETピンに高電圧パルスが加わります。  

結論として、このデータシートの条件を守っている大泉院長の方式の方がよいと思われます。 
実際にPA0 RESETピンがGPIOに設定されていた場合、CPUの入出力と衝突する恐れがないPOR後の8.8ｍS Hi-Zの間に  
12VのHVP高電圧パルスを短い数百uS加えるのが安全な方式になるかと思います。

## **TINY UPDI_HV_WRITER-ｗ-RESETの基本方針**
以上の検討内容で、今回のリセット付きnewATTINY用UPDI HV対応プログラマーの基本方式を決めました。
* 基本は安価な中華製USBserial基板を使用したUPDIserialプログラマー
* HVPのための12V電源は簡易のDC-DC電源回路をオンボードで作り、電源はUSBserialの3.3V-5Vから供給する。
* RESETスイッチを付けて、RESET終了後の8mS以内に2～400us幅の12V高電圧パルス(HVP)を印加できるようにする
* RESET時にターゲットにHVPを印加するためのHVPスイッチを付ける（常に印加する時はHVP‗SWはショートする)
* CPUをパワーオンリセットするために、ターゲットにはserialUSBモジュールからの3.3V/5Vを供給する
  (3.3V電源時は50ｍA程度の供給能力。不足する場合は、USBserial基板に3.3V LDOを追加改造する)

  このような仕様にして、使い方は下記のようになります。
* 通常のUPDI書き込みは、どのスイッチも押さずに書き込みが可能
* FUSEでPA0 RESETピンがUPDI以外に設定の場合はプログラム前に高電圧(HVP)を印加して、UPDIでプログラムする。
  操作方法は、1.RESETスイッチとHVPスイッチを同時に押す 2.RESETスイッチを解除後、続けてHVPスイッチを開放
  PA0 RESETピンがUPDI通信可能になって、プログラムできます。次のRESETでプログラムが有効になります。
  ※ 常にRESET時にHVPを印加してよい場合は、HVPスイッチを線でショートしてください。RESET押しのみでOKになります。　  

## 回路図
<img src="https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/d5f86092-47d6-4fc4-9925-b66a0f7be015" width="640">

**UPDI_HVP_PROGRAMMER_V18回路図**  

回路図と基板ははKiCadで設計しました。  
接続は、USBserialモジュールと2.54ｍｍピッチのピンソケット6Pを使って接続します。  
向きが逆になるので、ピン配列も逆になるので注意してくださいね。
1. GND
2. DTS
3. Vcc
4. TX
5. RX
6. DTR

ターゲット機器へは同じ2.54ｍｍピッチのピンソケット3Pで接続します。
1. UPDI
2. TG_Vdd(3.3Vか5V）
3. GND
Arduino用のジャンパーワイヤでターゲット機器と接続します。

## 部品リストと購入先
大半は秋月電機通商で購入可能です。
一部の部品はAliExpressで購入しています。
serialUSB変換モジュールと一緒に注文してくださいね。

特に特殊な部品はないと思いますが、12Vを生成するためのDC-DC電源をオンボードで
組んでいます。 そのICとか、インダクタ、スイッチング用のショットキーダイオードが
少し入手が難しいかもしれません。
電源完成モジュールが安いので、買ってばらして部品を使うのも手かもしれません。

またチップ抵抗は秋月でも扱ってないので、Aliからチップ抵抗のセットを買うのが安上がりです。
定数がない場合は、違う値の抵抗を2段重ねにして抵抗値を合わせて使うのも生活の知恵(笑)です。

基板は安くて速い、JLCPCBに発注しました。　
5枚で送料込み約550円、2～3年前に比べるとOCSを使っての送料が安くなっています。

<img src="https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/e4f9ec98-439f-4367-9a6f-9f7e3a40f345" width="640">


## 実装の注意点
実装もとくに注意する点はないかと思います。  
ほとんどチップ部品を使ってますので、慣れてない人には一部大変かもしれません。
SMD(表面実装)の練習と思って頑張ってみてくださいね。 

部品は1608サイズのチップ部品がほとんどです。ICやFETはピッチが細かいですが  
少し練習すればなんとかなると思います。DC-DC用のICのピッチが0.65ｍｍと細かく 
インダクタも熱容量が大きいので、ハンダ付けは十分注意して取り付けて下さいね。  

半田付けスタンドもあると便利ですよ。  
<img src="https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/e88c25ff-851e-4a96-863b-56bd53ab7973" width="480">  

## 使い方
今回はPanasonicの天井灯のリモコンを制御します。  
・タクトSWを押すとリモコンコードの発信が始まります。  

・リモコンは"mode切り替え", "UP", "DOWN" の3種類のコードを発信します。  

・リモコンコード発信の動作中は赤LEDが点滅して動作中を知らせます。  

・全部のコードが発振されるまで（テレビが消えるまで）押し続けてください。    


## スリープ時電流
ATTINY202のスリープ時電流を測定しました。  
0.1uAで、ATTINY85の0.2～0.3uAより明らかに小さいです。  
<!--![IMG_0471](https://github.com/todopapa/TINY202_IR_REMOTE_ISR/assets/16860878/f6b65397-b703-45a6-a5ef-4505950a0ba0)-->
<img src="https://github.com/todopapa/TINY202_IR_REMOTE_ISR/assets/16860878/f6b65397-b703-45a6-a5ef-4505950a0ba0" width="320">  

## クロックの設定 (8MHz)
今回は、ATTINY85のクロックに合わせてクロックは8MHz設定にします。  
（assmblerのNOPでタイミングを調整した delay_ten_us() という10uS単位のディレイ関数があるためです）

## 他のATTINY202開発参考資料

pin change Interrupt  
https://www.avrfreaks.net/s/topic/a5C3l000000UaC5EAK/t152923  

Technoblogy New ATtiny Low Power  
http://www.technoblogy.com/show?2RA3  

New ATTINY コード記述方法　Direct Port Manipulation  
https://github.com/SpenceKonde/megaTinyCore/blob/master/megaavr/extras/Ref_DirectPortManipulation.md  

Microchip  Getting Started with GPIO TB3229  
https://www.microchip.com/en-us/application-notes/tb3229  

ATTINY202/402/802 datasheet  
https://ww1.microchip.com/downloads/aemDocuments/documents/MCU08/ProductDocuments/DataSheets/ATtiny202-204-402-404-406-DataSheet-DS40002318A.pdf  
