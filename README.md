# UPDI_HV_WRITER-w-RESET
This is a new AVR ATTINY series UPDI programmer with HV pulse injection avility on power on reset timing.
<!-- ![TINY202_IR_REMOTE 2024-05-02 233458]https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/7ae2ed78-f5d1-4399-9f40-e7d13989e965-->
<img src="https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/7ae2ed78-f5d1-4399-9f40-e7d13989e965"  width = "480">  

## はじめに
### 中華製USB Serial変換モジュール基板を使ったUPDI対応のプログラマーについて
AVR New ATTINY 0/1/2 シリーズは旧ATTINY13,85等より性能が上がって、値段も手ごろなので
これを使いたいと思うのですが、書き込み方式が今までの3線ICSPから、1線のUPDI方式にかわり
いままでのAVR ISP MK2とかが使えなくなりました。
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
<img src= "https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/d541ac43-7fc5-4661-946c-c52dff27c6bc" width = "480">  
**日本語版 30.3.2.1.3. RESETﾋﾟﾝの高電圧無効化でのUPDI許可**   

HVPの印加は、/RESET信号が"H"に戻ってから、HVPのパルス幅は100～1ｍS以内とのことですね。  

次の章には、下記のようにHVPとPORのタイミングについて明確に記されています。  
**30.3.2.1.4. 汎用入出力(GPIO)に対する出力許可計時器保護  
ｼｽﾃﾑ構成設定0(FUSE.SYSCFG0)ﾋｭｰｽﾞのRESETﾋﾟﾝ構成設定(RSTPINCFG)ﾋﾞｯﾄが’00’の時に、RESETﾋﾟﾝは汎用入出力(GPIO)と
して構成設定されます。GPIOが出力を活動的に駆動するのとUPDI高電圧(HV)許可手順開始の間での潜在的な衝突を避けるた
め、GPIO出力駆動部はｼｽﾃﾑ ﾘｾｯﾄ後に最小8.8ms間禁止されます。
HVﾌﾟﾛｸﾞﾗﾐﾝｸﾞ手順に入るのに先立って常に電源ONﾘｾｯﾄ(POR)を発行することが推奨されます。**

前の2方式は、USBSerialモジュールのRTS/DTR信号がプログラム前に "L" に下がるタイミングでHVP印加していますので、非同期にATTINYのRESETピンに高電圧パルスが加わります。
結論として、大泉院長の方式の方がデータシートの条件を守っているのがいいと思います。 
実際にPA0 RESETピンがGPIOに設定されていた場合、POR後の8.8ｍS Hi-Zになっている間にHVPを加えるのが、ターゲット機器に損傷を与えにくい良い方法だと思います。

## **TINY UPDI_HV_WRITER-ｗ-RESETの基本方針**
以上の内容で、今回のリセット付きnewATTINY用UPDI HV対応プログラマーの基本方式を決めました。
* 基本は安価な中華製serialUSBモジュールを使用したUPDIserialプログラマー　
* RESETスイッチを付けて、RESET終了後の8mS以内に2～400us幅の12V高電圧パルス(HVP)を印加できるようにする
* RESET時にターゲットにHVPを印加するためのHVPスイッチを付ける（常に印加する時はSWをショートすればOK)
* ターゲットにserialUSBモジュールからの3.3V/5Vを印加できる。(3.3Vは50ｍAくらいしか流せないので注意）

  このような仕様にして、使い方は下記のようになります。
* 通常のUPDI書き込みは、どのスイッチも押さずに書き込みが可能
* FUSEでPA0 RESETピンがUPDI以外に設定の場合はプログラム前に高電圧(HVP)を印加して、UPDIでプログラムする。
  操作方法は、1.RESETスイッチとHVPスイッチを同時に押す 2.RESETスイッチを解除後、続けてHVPスイッチを開放
  PA0 RESETピンがUPDI通信可能になって、プログラムできます。次のRESETでプログラムが有効になります。　　  

## 回路図
<img src="https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/d5f86092-47d6-4fc4-9925-b66a0f7be015" width="640">

**UPDI_HVP_PROGRAMMER_V18回路図**  

回路図と基板ははKiCadで設計しました。  
接続は、serialUSBモジュールからは6P  
1. GND
2. DTS
3. Vcc
4. TX
5. RX
6. DTR

ターゲット機器へは3Pで
1. UPDI
2. TG_Vdd(3.3Vか5V）
3. GND
になります。

## 部品リスト　
大半は秋月電機通商で購入可能です。
一部の部品はAliExpressで購入しています。
serialUSB変換モジュールと一緒に注文してくださいね。

基板はJLCPCBに発注しました。　

5枚で送料込み約560円、2～3年前に比べるとOCS使った送料が安くなっています。
* main.c : メインプログラム  
* IRcodes.c : 元は北米/アジア仕向け 各社テレビのON/OFF 赤外線コードの構造体  
  　　　　　　　→ 自分が使う装置の赤外線コードに書き換える,TINY202は容量が少ないのが　5～10くらいが入る  
* util.c : デバッグ用ソフトUart (未使用）  
* Debugフォルダ：生成したHEXファイル、ELFファイルが入る  
<img src="https://github.com/todopapa/UPDI_HV_WRITER-w-RESET/assets/16860878/775e85a9-cd0d-40b4-8827-e8228541acea" width="640">


## 使い方
今回はPanasonicの天井灯のリモコンを制御します。  
・タクトSWを押すとリモコンコードの発信が始まります。  

・リモコンは"mode切り替え", "UP", "DOWN" の3種類のコードを発信します。  

・リモコンコード発信の動作中は赤LEDが点滅して動作中を知らせます。  

・全部のコードが発振されるまで（テレビが消えるまで）押し続けてください。    

・ボタンを離してIRコードの発信が終わると、最後に4回LEDが短く点灯した後、動作が終わります。  

・動作が終わると、TINY202はSleepモードに入り、動作電流は < 1uA になります。 (約0.1uA)  

・このリモコンの動作モードは、ボタンを押している間連続で同じIRコードを発信します。  

・単発でよい場合は、コードの226行目、252行目をコメントアウトしてください。  
　　226　// while ((~PORTA.IN & SW0_PIN) | (~PORTA.IN & SW1_PIN) | (~PORTA.IN & SW2_PIN)){ //while SW0 or SW1 or SW2 is pressed then continue IR LED flashing  
　　252 //	}  

## リモコンコード
<img src="https://github.com/todopapa/TINY202_IR_REMOTE_ISR/assets/16860878/f1b21625-a868-4491-9a7b-8a4c05b9795c" width="360">   

**パナソニック天井灯リモコン　HK9337**   
このリモコンはM5060という三菱のリモコン用ICを使用しています。   
もうこのICは手に入りませんが、コンパチの[PT2560](https://pdf.dzsc.com/60-/PT2560-002.pdf)という中華ICのデータシートが見つかりましたので、これを参考に    
必要なコードを解析していきます。（実際はオシロスコープによる波形解析を行った）  

<!<!----![PT2560 CODE 2024-05-03 011738](https://github.com/todopapa/TINY202_IR_REMOTE_ISR/assets/16860878/00e489bc-4ffc-4ae1-a2fb-4cbaf296b434)  -->
<img src="https://github.com/todopapa/TINY202_IR_REMOTE_ISR/assets/16860878/00e489bc-4ffc-4ae1-a2fb-4cbaf296b434" width="640">   

この図でLeader Codeは3.57ｍSと1.80ｍS、”0”は420/480uS、"1"は420/1320uSでした。  
コードは下記でした。　キャリア周波数は38KHzがICの仕様ですが、440kHzのセラミック発振器を使って36.37kHzでした。  
これに従って、IRcodes.c　を修正しています。  

mode切り替え：custom code 34h、4Ah、90h　/　data code   14h、84h END  
UP　　　　　：custom code 34h、4Ah、90h　/　data code 　54h、C4h END  
DOWN　　　　：custom code 34h、4Ah、90h　/　data code 　D4h、44h END  

## スリープ時電流
ATTINY202のスリープ時電流を測定しました。  
0.1uAで、ATTINY85の0.2～0.3uAより明らかに小さいです。  
<!--![IMG_0471](https://github.com/todopapa/TINY202_IR_REMOTE_ISR/assets/16860878/f6b65397-b703-45a6-a5ef-4505950a0ba0)-->
<img src="https://github.com/todopapa/TINY202_IR_REMOTE_ISR/assets/16860878/f6b65397-b703-45a6-a5ef-4505950a0ba0" width="320">  

## クロックの設定 (8MHz)
今回は、ATTINY85のクロックに合わせてクロックは8MHz設定にします。  
（assmblerのNOPでタイミングを調整した delay_ten_us() という10uS単位のディレイ関数があるためです）

ATTINY202のクロック系統の構成は下記のように、MAIN_CLOCK源の設定とプリスケーラになっています。  
この出力CLK_PERがTIMER/COUNTER等に入力されます。    
<img src="https://github.com/todopapa/TINY202_IR_REMOTE_ISR/assets/16860878/0daeca7c-4309-414f-91b6-8611f89cafcf" width="480">  

起動RESET後のクロック系統の動作は、下記データシートに書かれているように、Main Clock 20/16MHz　Prescaler 6分周になっています。  
これを今回は　Main clock 16MHz Prescaler 2分周にして、クロックを8MHzに設定しします。
<img src="https://github.com/todopapa/TINY202_IR_REMOTE_ISR/assets/16860878/a853f67f-75dc-4714-aa42-43a20f36e04d" width="480">  
<!--![TINY202 default clock 2024-05-03 10
2832](https://github.com/todopapa/TINY202_IR_REMOTE_ISR/assets/16860878/a853f67f-75dc-4714-aa42-43a20f36e04d)-->

プログラムのこの部分です。データシートにあるようにプロテクトされている部分ですから_PROTECTED_WRITE() で当該レジスタに書き込みます。
```
void SYSCLK_init(void) {
	// SYSCLK 8MHz (16MHz /2)
	/* Set the Main clock to internal 16MHz oscillator*/
	_PROTECTED_WRITE(CLKCTRL.MCLKCTRLA, CLKCTRL_CLKSEL_OSC20M_gc);
	/* Set the Main clock divider is / 2 and Main clock prescaler enabled. */
	_PROTECTED_WRITE(CLKCTRL.MCLKCTRLB, CLKCTRL_PDIV_2X_gc | CLKCTRL_PEN_bm); //0x01
}
```
これだけでは、内部クロックに20MHzが出力されるので、(FUSE.OSCCFG) fuse で16MHzの設定をする必要があります。

<img src="https://github.com/todopapa/TINY202_IR_REMOTE_ISR/assets/16860878/2330607a-ac0a-46a9-95c3-d84d2e2e9961" width="640"> 

FUSE.OSCCFGレジスタの設定は下記、デフォールトは0x02（20MHz）、これを0x01（16MHz）に変更します。
<img src="https://github.com/todopapa/TINY202_IR_REMOTE_ISR/assets/16860878/407393ac-5878-47cb-8575-22901651c689" width="640"> 

FUSEの設定は、プログラムに構造体で記載してHEXファイルにFUSEデータを入れて、書き込み器でFLashデータと一緒にFUSEデータを  
書き込む方法もあるのですが、これはなぜかうまくいきませんでした。
困りはててAVR Freakで調べて、AVRDudeで書き込む方法があると知ったので、もっぱらこちらでFUSEを書き込んでいます。  
fuse2がFUSE.OSCCFGレジスタになります。  

Fuses on ATtiny1614 with AVRDUDE  
https://www.avrfreaks.net/s/topic/a5C3l000000BqV9EAK/t391706  
WDTCFG = fuse0, BODCFG = fuse1, OSCCFG = fuse2.....  
ディレクトリ　D:\Program Files (x86)\AVRDUDESS　にて  (fuse2の指定だけでOKです。COM5は自分の環境に合わせて下さい）  
avrdude -Cavrdude.conf -c serialupdi -p t202 -P COM5 -U fuse0:w:0x00:m -U fuse1:w:0x00:m -U fuse2:w:0x1:m   

## TIMER/COUNTERの使い方
ここがATTINY85と大きく変わっているので、やっぱり説明せんばなりませんね。
MicroChipの技術資料 [TB3217 Getting Started with Timer/Counter Type A (TCA)](https://www.microchip.com/en-us/application-notes/tb3217)) を参考にしました。
TINY85ではTCNT0とTCNT1という２つの8bitカウンタ/タイマがあって、OCRnX, OCRnBに設定した値と比較して任意の周波数やディレイを作ってました。
これに対しTINY202ではTCA0とTCB0という2つの16bitカウンタ/タイマがあって前段のPrescaler、CNTレジスタ、PERレジスタ、CMPnレジスタで制御しています。

```
void xmitCodeElement(uint16_t ontime, uint16_t offtime ) {
	// start TCA0 outputting the carrier frequency to IR emitters on CMP0 WO0 (PA3, pin 7)
	/* set waveform output on PORT A */
	TCA0.SINGLE.CTRLB = TCA_SINGLE_CMP0EN_bm // enable compare channel 0
	| TCA_SINGLE_WGMODE_FRQ_gc;	// set Frequency mode
	/* disable event counting */
	TCA0.SINGLE.EVCTRL &= ~(TCA_SINGLE_CNTEI_bm);
	/* set frequency in FRQ mode */
	// TCA0.SINGLE.CMP0 = PERIOD_EXAMPLE_VALUE;   //the period time (FRQ) already set in main()
	/* set clock source (sys_clk/div_value) */
	TCA0.SINGLE.CTRLA = TCA_SINGLE_CLKSEL_DIV1_gc   // set clock source (sys_clk/1)
	| TCA_SINGLE_ENABLE_bm; /* and start timer */
	// keep transmitting carrier for onTime
	delay_ten_us(ontime);
	//for debug continue emitting test
	// while(1);
	
	// turn off output to IR emitters on 0C1A (PB1, pin 6) for offTime
	TCA0.SINGLE.CTRLA &= ~(TCA_SINGLE_ENABLE_bm); /* stop timer to set bit "0" */
	TCA0.SINGLE.CTRLB = 0;	// CTRLB register RESET add for forced OUTPUT "L"
	PORTA.OUTCLR = IRLED_PIN;  // turn off IR LED

	delay_ten_us(offtime);
}
```
1.CTRLBの設定：TCA0をFRQモード (WOnに出力を出す)、コンペアレジスタにCMP0を選択（WO0に出力される）  
2.EVCTRLの設定：EVENT入力を止める    
3.CMP0レジスタに比較値を入れ出力周波数を決める（これはmain()で設定するので、ここはコメントアウト）    
4.CTRLAの設定：プリスケーラの設定1/1、TCA0をEnable（起動）する    
止めるときは、CTRLAでTCA0をDisableするだけでなく、CTRLBも0x00を入れてPA3へのWO0の出力を止めないと  
PORTA.OUTCLR = IRLED_PIN(PIN3_bm);　だけでは”L"にならず、IR LEDが光りっぱなしになるので注意してください。  

なお、コンペアレジスタ CMPnを選択すると出力はWOnになります。　下記の出力ピンリストで参照ください。
PORTMUXで代替ピンの使用も可能になります。WO0のデフォールトの出力はPA3ですが、PA7にも出力できます。
<img src="https://github.com/todopapa/TINY202_IR_REMOTE_ISR/assets/16860878/922acc81-6f77-4c1d-9545-99a526dc1c91" width="640">

## Pin Chnge Interruptの使い方
  詳細は工事中です。すみません,
  コードを見ていただけると、これで動いています。

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
