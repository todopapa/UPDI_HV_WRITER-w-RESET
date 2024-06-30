# UPDI_HV_WRITER-w-RESET
This is a new AVR ATTINY series UPDI programmer with HV pulse injection ability on power on reset timing.
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
4. 要望があれば、不具合点を改善して改版し、ほかのおもちゃドクターにも頒布する。  
5. 
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

## 設計内容
###  **TINY UPDI_HV_WRITER-ｗ-RESETの基本方針**
以上の事前検討内容で、今回のリセット付きnewATTINY用UPDI HV対応プログラマーの基本方式を決めました。
* 基本は安価な中華製USBserial基板を使用したUPDIserialプログラマー
* HVPのための12V電源は簡易のDC-DC電源回路をオンボードで作り、電源はUSBserial基板の3.3V-5Vから供給する
* RESETスイッチを付けて、RESET終了後の8mS以内に2～400us幅の12V高電圧パルス(HVP)を印加できるようにする
* RESET時にターゲットにHVPを印加するためのHVPスイッチを付ける ※1
* CPUをパワーオンリセットするために、ターゲットにはUSBserial基板からの3.3V/5Vを供給する
  (3.3V電源時は50ｍA程度の供給能力。不足する場合は、USBserial基板に3.3V LDOを追加改造する)

  このような仕様にして、基本的な使い方は下記のようになります。
* 通常のUPDI書き込みは、どのスイッチも押さずに書き込みが可能
* FUSEでPA0 RESETピンがUPDI以外に設定の場合はプログラム前に高電圧(HVP)を印加して、UPDIでプログラムする。  
  操作方法は、1.RESETスイッチとHVPスイッチを同時に押す 2.RESETスイッチを解除後、続けてHVPスイッチを開放  
  PA0 RESETピンがUPDI通信可能になって、プログラムできます。次のRESETまで、プログラムが有効になります。  
  ※1 常にRESET時にHVPを印加してよい場合は、HVPスイッチを線でショートしてください。RESET押しのみでOKになります  　  


### 回路図
<img src="https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/d5f86092-47d6-4fc4-9925-b66a0f7be015" width="640">

**UPDI_HVP_PROGRAMMER_V18回路図**  

### 基板パターン図

<img src="https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/e09b60c3-313e-40db-b92a-36d270b87985" width="640">

回路図と基板ははKiCadで設計しました。  
接続は、USBserial基板と2.54ｍｍピッチのピンソケット6Pを使って接続します。  
USBserial基板と向きが逆になるので、ピン配列も逆になるので注意してくださいね。
1. GND
2. DTS
3. Vcc
4. TX
5. RX
6. DTR

ターゲット機器へは同じ2.54ｍｍピッチのピンソケット3Pで接続します。
ここはユーザで縦型ピンヘッダを付けるなり、自由にしてください。
1. UPDI
2. TG_Vdd(3.3Vか5V）
3. GND
　Arduino用のジャンパーワイヤでターゲット機器と接続します。

### 部品リストと購入先
大半は秋月電機通商で購入可能です。
一部の部品はAliExpressで購入しています。
USBserial変基板と一緒に注文してくださいね。

特に特殊な部品はないと思いますが、12Vを生成するためのDC-DC電源をオンボードで
組んでいます。 そのICとか、インダクタ、スイッチング用のショットキーダイオードが
少し入手が難しいかもしれません。
電源完成モジュールが安いので、買ってばらして部品を使うのも手かもしれません。

またチップ抵抗は秋月でも扱ってないので、Aliからチップ抵抗のセットを買うのが安上がりです。
定数がない場合は、違う値の抵抗を2段重ねにして抵抗値を合わせて使うのも生活の知恵(笑)です。

基板は安くて速い、JLCPCBに発注しました。　
5枚で送料込み約550円　発注から10日くらいで到着します。2～3年前に比べるとOCSを使っての送料が安くなっています。

<img src="https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/e4f9ec98-439f-4367-9a6f-9f7e3a40f345" width="640">

### 実装の注意点
実装もとくに注意する点はないかと思います。  
ほとんどチップ部品を使ってますので、慣れてない人には一部大変かもしれません。 
SMD(表面実装)の練習と思って頑張ってみてくださいね。  
<img src="https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/e88c25ff-851e-4a96-863b-56bd53ab7973" width="480">    

部品は1608サイズのチップ部品がほとんどです。ICやFETはピッチが細かいですが  
少し練習すればなんとかなると思います。DC-DC用のICのピッチが0.65ｍｍと細かく 
インダクタも熱容量が大きいので、そこのハンダ付けは十分注意して取り付けて下さいね。  

半田付けスタンドもあると便利ですよ。  
<img src="https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/a31cd522-f385-4a7f-8ce8-3de96c2f7f1c" width="480">  

### 使い方
つつじが丘おもちゃ病院の大泉さんの[TINY402電子オルゴールV1.2](http://tutujith.blog.fc2.com/blog-entry-741.html)  のテスト回路を実装した時を例にして使い方を説明します。    
手順としては下記のようになります。  
・テストプロジェクトをATMEL STUDIOでコンパイルします。生成したHEXファイルはプロジェクトのreleaseに入ります。  


<img src="https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/686991d0-5f53-46f4-b4ff-daec3d81dd3b" width="640"> 

・ターゲットとしてブレッドボード上にATTINY402を使ったテスト回路を組みます。動作確認なのでシンプルなオルゴールを作ります。  


<img src="https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/2af494f8-31a1-4eba-bee7-77438e3d14d5" width="480">    

・このUPDI_HV書き込み機をターゲットに接続します。赤 電源線(TG_Vdd)。黒 グランド線(GND)、白 UPDI線の3本です。  


<img src="https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/e2da1b93-806f-4e62-9946-2d508ed946c6" width="480">    

・GUIのAVRDUDESSをPCにインストールしておきます。  
　さきほどのテスト用HEXファイルをターゲットのATTINY402に書き込みます。  

 
<img src="https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/c2430fee-ec31-4e97-b804-334140cab901" width="480">   

 
　このときは、まだFUSEビットが書き込まれていないので、そのまま書けるはずです。  
  もし、FUSEが書き込まれていた場合は、書き込み機の黄色ボタンと赤ボタンを上にある手順で押してから書いて下さい。  
  
・次にFUSEの書き込みを行います。AVRDUDESSが新ATTINYシリーズのFUSE書き込みに対応していないので、  
　今度はコマンドラインでAVRDUDEを使用します。AVRDUD.exeはAVRDUDESSをインストールしたフォルダにあります。 
 今回のプロジェクトでは、クロック 16MHzでPA0 RESETピンをGPIOにする設定で、  
　OSCCFG（FUSE2）を20MHｚ→ 16ＭＨｚ、SISYCFG0（FUSE５）をUPDI→ GPIOに変更するので、 
　avrdude -Cavrdude.conf -c serialupdi -p t402 -P COM7 -U fuse2:w:0x01:m -U fuse5:w:0xC0:m を書き込みます  
 （com7はご自分の環境にて変えてください
 ）   
 <img src="https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/656d9b48-e459-4484-8cec-75ec6dbe481c" width="480">  

・書き込めたら、書き込み機のRESETスイッチを押してください。（あるいはUSBケーブルをいったん抜き/差ししてください)  
　ターゲットのトグルSWを押すと、オルゴールの曲（シャボン玉とんだ）がスピーカからなるはずです。   
 　アンプがないので小さい音ですが、きれいな曲が聞こえると思います。 SWを長押しすると停止します。 

[![オルゴール演奏テスト](https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/775e3ddb-164a-46eb-9a81-e4c0a9cf4ea9)](https://youtu.be/el-AeCiNPmo)   **<-- サムネイル画像クリックでYoutubeでの動画に飛びます。ブラウザの戻る、で戻ります**    

### タイミング確認  
・オシロでタイミングを確認しました。   
**ターゲットのパスコンが１０uFの時**　上がUDPI信号　下がTG_Vdd(ターゲット電源)電圧 1ｍS/div  
 <img src="https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/af174b25-3709-483d-86ac-424630ab51ce" width="480"> 

**ターゲットのパスコンが１００uFの時** 上がUDPI信号　下がTG_Vdd(ターゲット電源)電圧 1ｍS/div  
 <img src="https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/cab0e20f-a592-41f3-ac10-f1b539ac5599" width="480">  

USBserial基板の3.3V電源でも、RESETの立ち上がり後２～3ｍS十分余裕をもって12Vパルスがでているのがわかると思います。  
また、R9 1kΩはTG_Vddのブリーダ抵抗ですが、最初の10kΩでは100uF時のRESET時の立下りが遅かったので1kΩに変更しました。
ターゲットのパスコンの容量がもっと大きくなるとと立下りが遅くなりますので、その時はリセットボタンを長く押してくださいね。  

以上の内容になります。  

## 他のATTINY202/402 UPDIプログラマ(書き込み機）開発参考資料  
ATTINY202/402/802 datasheet    
https://ww1.microchip.com/downloads/aemDocuments/documents/MCU08/ProductDocuments/DataSheets/ATtiny202-204-402-404-406-DataSheet-DS40002318A.pdf   
つつじが丘おもちゃ病院tinyAVRプログラマー  
http://tutujith.blog.fc2.com/blog-entry-726.html  
つつじが丘おもちゃ病院tinyAVR電子オルゴールVer1_2（tiny402をサポート）  
http://tutujith.blog.fc2.com/blog-entry-741.html  
糸魚川「おもちゃクリニックゆりかご」Dr.わたなべ氏 おもちゃ修理「電子カルテ」  
https://blog.canpan.info/charts/   
トドお父さん通信 「ATTINY402に電子オルゴールのHVPプログラム書き込みテストをしました」  
https://ameblo.jp/powpher/entry-12852703185.html  
