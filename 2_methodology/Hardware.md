### Page Contents <!-- omit in toc -->
- [1. Hardware Overview](#1-hardware-overview)
- [2. UART Log](#2-uart-log)
  - [2.1. Supplies](#21-supplies)
  - [2.2. Step 1 - UART Port Location](#22-step-1---uart-port-location)
  - [2.3. Step 2 - UART Port Soldering](#23-step-2---uart-port-soldering)
  - [2.4. Step 3 - USB to TTL Serial Cable Connection](#24-step-3---usb-to-ttl-serial-cable-connection)
  - [2.5. Step 4 - UART Log](#25-step-4---uart-log)
  - [2.6. Result](#26-result)
- [3. syscon dump](#3-syscon-dump)
  - [3.1. Supplies](#31-supplies)
  - [3.2. Step 1 - Syscon Desoldering](#32-step-1---syscon-desoldering)
  - [3.3. Step 2 - Connect Syscon with Teensy 4.0 board](#33-step-2---connect-syscon-with-teensy-40-board)
  - [3.4. Step 3 - Teensy4.0 programming](#34-step-3---teensy40-programming)
  - [3.5. Step 4 - Syscon Dump](#35-step-4---syscon-dump)
  - [3.6. Dump Result](#36-dump-result)
- [4. Sflash Dump](#4-sflash-dump)
  - [4.1. Step 1 - Connect Sflash with Teensy 2.0 board](#41-step-1---connect-sflash-with-teensy-20-board)
  - [4.2. Step 2 - NORway Environment Setting](#42-step-2---norway-environment-setting)
  - [4.3. Step 3 - Teensy Loader](#43-step-3---teensy-loader)
  - [4.4. Step 4 - SPIway - info](#44-step-4---spiway---info)
  - [4.5. Step 5 - SPIway - dump](#45-step-5---spiway---dump)
  - [4.6. Dump Result](#46-dump-result)
  - [4.7. Plus](#47-plus)
- [5. Reference](#5-reference)

---

# Hardware <!-- omit in toc -->

## 1. Hardware Overview
<!--
> 본 프로젝트에서 하드웨어적으로 syscon dump와 sflash dump를 진행하게된 계기는 다음과 같다.

시스템 펌웨어 버전과 제조 모드 정보는 syscon의 snvs에 저장되고, syscon은 보드의 거의 모든 항목에 대한 클럭/전원 관리, 대부분의 프로세서 부팅, 기타 저속 주변 장치에 대한 프록시 역할 등 다양한 역할을 하기 때문에 dump를 해보았다. 또한 Aeolia용 펌웨어 업데이트 패키지는 sflash에 저장되기 때문에 sflash도 함께 dump를 진행해봤다. 추가적으로 새로운 칩에 dump한 내용을 백업해 두면, 추후 버전을 업데이트 하더라도 다시 백업해둔 칩으로 교체하여 다운그레이드를 할 수 있다.
-->
![syscon and sflash](https://user-images.githubusercontent.com/40509850/101976312-7cfae100-3c87-11eb-8383-a48ca297c669.png)


Syscon is a chip in the PS4 that has information such as system firmware version and manufacturing mode information, and has features such as clock/power management, booting processors and proxy. Thus, we tried dumping syscon. And 
we tried dumping sflash which has Aeolia firmware update package. Moreover, we can downgrade PS4 firmware by changing syscon chip to backup chip. you can see more details on the [fail0verflow's blog](https://fail0verflow.com/blog/2018/ps4-aeolia/).

## 2. UART Log

### 2.1. Supplies
- Soldering iron, Solder, JumperWire
- USB to TTL Serial Cable

### 2.2. Step 1 - UART Port Location

<img width="433" alt="UART" src="https://user-images.githubusercontent.com/48618245/101594879-bcca8a00-3a35-11eb-925c-f4f1cb90cc11.jpeg">
<!--
![UART](https://user-images.githubusercontent.com/48618245/101594879-bcca8a00-3a35-11eb-925c-f4f1cb90cc11.jpeg)
-->
<!--
PS4를 분해하여 메인보드를 보면 위 사진과 같은 곳에 UART 포트가 있다.
-->

There are UART ports on the motherboard.

<img width="433" alt="UART_point" src="https://user-images.githubusercontent.com/48618245/101595248-690c7080-3a36-11eb-9b79-cee09075332b.png">

<!--
위 사진은 우리가 사용한 SAB-001 보드의 UART 포트이다.
-->

The above picture shows the UART ports of SAB-001.

### 2.3. Step 2 - UART Port Soldering

<img width="164" alt="UART_납땜" src="https://user-images.githubusercontent.com/48618245/101595305-7f1a3100-3a36-11eb-904c-093f6788bf5e.png">

<!--
JumperWire를 위에서 확인한 UART 포트에 납땜해서 연결을 해준다.
-->

Connect the UART ports with jumperwire.

### 2.4. Step 3 - USB to TTL Serial Cable Connection

<img width="600" alt="UART_to_Serial" src="https://user-images.githubusercontent.com/48618245/101595351-92c59780-3a36-11eb-9aa6-d385ec93f6bc.jpeg">
<!--
![UART_to_Serial](https://user-images.githubusercontent.com/48618245/101595351-92c59780-3a36-11eb-9aa6-d385ec93f6bc.jpeg)
-->
<!--
USB to TTL Serial Cable에 Step 2에서 납땜한 JumperWire를 연결해준다. GND는 GND끼리 연결해주고 UART 포트의 TX는 USB to TTL Serial Cable의 RX에 연결해준다.
-->

Connect the jumperwire with USB to TTL Serial Cable. Connect GND UART port with GND Serial Cable, and connect TX UART port with RX Serial Cable.

### 2.5. Step 4 - UART Log
<img width="723" alt="UART_Log1" src="https://user-images.githubusercontent.com/48618245/101595832-62322d80-3a37-11eb-97b5-927d3e629647.png">
<img width="179" alt="UART_Log_Blank" src="https://user-images.githubusercontent.com/48618245/101595863-6f4f1c80-3a37-11eb-9ef2-00663cae257f.png">

<!--
소니에서 UART Log를 확인하지 못하도록 공백으로만 출력하도록 해놨다.
-->

However, UART Log is blocked. Sony has taken steps to prevent the UART log from being printed. All we can see is just space.

### 2.6. Result
<!--
UART Log를 보고 싶으면 Jailbreak 해놓은 PS4에서 ps4debug.bin 파일을 bin loader로 올리거나 mira 기능을 이용해 jailbreak를 하면 UART Log가 활성화된 것을 확인할 수 있다. 하지만 이는 하드웨어적으로 연결 안해도 nc를 이용하여 포트 접속만 해도 확인할 수 있으니 하드웨어적인 성과는 없었다. 하드웨어적인 연결로 UART Log를 확인하는 것이 아닌 nc를 사용해 PS4에서 UART Log 활성화된 포트로 접속하는 방법은 Jailbreak 목차에서 확인하면 될 것 같다.
-->

If you want to see the log, you have to jailbreak first using Mira. However, we can see the log without connecting UART physically, we can just use nc. Thus, connecting UART port seems to be meaningless. You can see how to see the UART log using nc in [Jailbreak section](https://github.com/Hacker-s-PlayStation/PlayStation4-Hacking-Guideline-ENG/blob/main/1_introduction/Jailbreak.md#23-UART-Log-Activation).

## 3. syscon dump

### 3.1. Supplies
- Hot air blower, Solder, Solder Wick, Flux)
- Jumper Wire, Resistance, USB to TTL Serial Cable, Pin Header, Capacitor
- Teensy 4.0

### 3.2. Step 1 - Syscon Desoldering

<img width="600" alt="desoldering" src="https://user-images.githubusercontent.com/48618245/101596857-3021cb00-3a39-11eb-8ede-48a42172527b.JPG">
<!--
![desoldering](https://user-images.githubusercontent.com/48618245/101596857-3021cb00-3a39-11eb-8ede-48a42172527b.JPG)
-->
<!--
syscon dump를 하기 위해서는 우선 syscon 칩을 디솔더링 해야하는데 사실 이 부분이 제일 힘들었다. 처음에 사용하던 열풍기로는 디솔더링이 잘 안됐다. 
-->
We have to desolder the syscon chip to dump the data. However, performance of our hot air blower was not quite good, thus desoldering didn't go well.

![Broken syscon chip](https://user-images.githubusercontent.com/48618245/101596946-50ea2080-3a39-11eb-9d13-8e457a331a46.PNG)

<!--
이 과정에서 무리하게 디솔더링을 하다가 syscon칩이 망가져서 PS4를 새로 하나 더 구입하게 됐다. 

더 높은 온도가 가능한 열풍기를 구입한 후 syscon 주변에 플럭스를 뿌려주고 `450℃`로 열을 가해주니 디솔더링이 쉽게 되었다. sflash도 이 방법으로 디솔더링하니 쉽게 되었다.

만약 가지고 있는 열풍기로 디솔더링이 되지 않는다면 무리하게 디솔더링하려고 시도하여 우리처럼 시행착오를 겪지말고 열풍기를 더 좋은 것으로 구입하는 것을 추천한다.
-->

Eventually, we bought another PS4 because the syscon chip was broken. And we desoldered syscon using flux and more powerful(about 450℃) air blower. If your air blower's performance is not good, do not desolder the chip by force. We recommend just buying more powerful air blower.


### 3.3. Step 2 - Connect Syscon with Teensy 4.0 board

<img width="414" alt="SYSGLITCH wiring diagram by Wildcard" src="https://user-images.githubusercontent.com/48618245/101595983-a6bdc900-3a37-11eb-8e23-2c50a206643a.png">

<!--
`Wildcard`가 제공해준 sysglitch diagram대로 `Teensy 4.0 보드(이하 Teensy4.0)`와 `syscon`을 연결하여 덤프를 진행하였다. diagram과 똑같이 연결을 해야 정상적으로 dump가 가능하니 이 부분을 잘 신경써서 해야한다.
-->

Connect desoldered syscon with Teensy 4.0 board. The above diagram is made by Wildcard.

### 3.4. Step 3 - Teensy4.0 programming

<img width="116" alt="Teensy_loader" src="https://user-images.githubusercontent.com/48618245/101596124-e389c000-3a37-11eb-9168-cee58b65fc63.png">

<!--
syscon glitch 하기 위해 Teensy4.0에서 동작하도록 만들어 놓은 hex 파일을 받고 `Teensy Loader`에 올린 후 Teensy4.0에 있는 버튼을 누르면 프로그래밍이 된다. 프로그래밍이 완료되면 후에 덤프가 가능해진다.
-->

Program the teensy board to dump the syscon. Connect teensy board with PC and then click the button on the board to enter programming mode. Download the hex file and load it on Teensy Loader. Then we can program the board by programming button on Teensy Loader.

[https://www.pjrc.com/teensy/loader.html](https://www.pjrc.com/teensy/loader.html) - Teensy Loader program download site<br>
[https://github.com/VV1LD/SYSGLITCH/releases/tag/T4-1.0](https://github.com/VV1LD/SYSGLITCH/releases/tag/T4-1.0) - SYSGLITCH_TEENSY4.0.hex download site

### 3.5. Step 4 - Syscon Dump

<img width="284" alt="realterm" src="https://user-images.githubusercontent.com/48618245/101596511-8c381f80-3a38-11eb-8fd5-58499a3e25e7.png">

<!--
realterm 프로그램을 사용하여 덤프를 뜬다. 

- pulldown 연결을 해제해놓은 상태로 시리얼 포트 및 baudrate을 115200 으로 설정하고 change 버튼을 누른다.
- Capture 탭에서 Start Overwrite 버튼을 누르면 화면이 빨갛게 변한다. 
- pulldown을 다시 연결하면 덤프가 떠진다. 4Mb 이상 덤프되면 다 됐다고 보면 된다.
-->

We need the program Realterm to dump the syscon.
- Set the serial port properly and set the baudrate to 115200 with the pulldown connection disconnected and press the change button.
- Press Start Overwrite button on the Capture tab. The window will change into red.
- Connect pulldown to dump the syscon. Read the dump file until it is 4MB in size.

[https://sourceforge.net/projects/realterm/](https://sourceforge.net/projects/realterm/) - realterm download site

### 3.6. Dump Result

<img width="885" alt="syscon_dump" src="https://user-images.githubusercontent.com/48618245/101596616-c5708f80-3a38-11eb-92d5-e2f0c6ddf6a8.png">

<!--
실제 덤프 뜬 파일 확인해보면 `Sony Computer Entertainment`가 나와있으면 덤프가 성공한 것이다.

만약 덤프가 정상적으로 되지 않고 실패했을 때는 계속 `Not Used`만 반복해서 떴었다.
-->

If you do the process correctly, you can see `Sony Computer Entertainment` in the dump file. Otherwise, you can not see the word and there is just a lot of `Not Used`.

## 4. Sflash Dump

### 4.1. Step 1 - Connect Sflash with Teensy 2.0 board

<img width="433" alt="sflash_diagram" src="https://user-images.githubusercontent.com/48618245/101597589-3feddf00-3a3a-11eb-9cda-7937a48d611c.png">

<img width="433" alt="sflash" src="https://user-images.githubusercontent.com/48618245/101597473-1634b800-3a3a-11eb-8be7-0a5d24554cd8.jpg">

<!--
![sflash](https://user-images.githubusercontent.com/48618245/101597473-1634b800-3a3a-11eb-8be7-0a5d24554cd8.jpg)

디솔더링은 syscon과 똑같이 진행하면 된다. syscon dump를 참고하면 된다. sflash를 디솔더링 하고, Teensy 2.0 보드(이하 Teensy2.0)에 연결해줬다. 연결할 때는 Wildcard가 제공해준 daigram과 위 사진을 보면서 연결하였다.

처음에 Teensy2.0을 연결하면 시리얼 포트로 인식이 되지 않을 것이다. Teensy를 COM 포트로 인식되게끔 하려면 아두이노 프로그램으로 한 번 쯤은 프로그래밍을 해줘야 한다.
자세한 내용은 [여기](https://www.pjrc.com/teensy/troubleshoot.html)에 잘 나와 있다.
-->

After desoldering the sflash, connect the sflash with Teensy 2.0 board. Above diagram is made by Wildcard.

If Teensy 2.0 is not recognized as serial port, You have to program Teensy board at least once from arduino. You can get more information in [here](https://www.pjrc.com/teensy/troubleshoot.html).

![com_serial](https://user-images.githubusercontent.com/48618245/101597743-79bee580-3a3a-11eb-9f5f-af34885ec829.png)

<!--
이렇게 포트 정보가 나타나야 툴을 제대로 돌릴 수 있다.
-->

### 4.2. Step 2 - NORway Environment Setting

<!--
SPIway라는 툴을 이용하여 덤프를 진행하면 된다. git clone을 해 주면 그 안에 `SPIway.py` 스크립트가 존재한다.
-->
SPIway tool is needed to dump sflash.  Python2.7.2 and pyserial2.5 are needed to execute `SPIway.py` script.

```bash
git clone https://github.com/hjudges/NORway
```

<!--
이 스크립트를 실행하기 위해서는 Python 2.7.2 버전 및 pyserial 2.5가 필요하다. 아래 링크로 접속하여 설치해 주자.
-->

- Python 2.7.2 ([http://www.python.org/ftp/python/2.7.2/python-2.7.2.msi](http://www.python.org/ftp/python/2.7.2/python-2.7.2.msi))
- pyserial 2.5 ([http://pypi.python.org/packages/any/p/pyserial/pyserial-2.5.win32.exe](http://pypi.python.org/packages/any/p/pyserial/pyserial-2.5.win32.exe))

### 4.3. Step 3 - Teensy Loader

![Teensy_loader2](https://user-images.githubusercontent.com/48618245/101597891-abd04780-3a3a-11eb-9335-e2424986820f.png)

<!--
TeensyLoader를 다운로드 받은 뒤 실행해 준다. git clone 받은 폴더에서 `.\NORway\SPIway\Release` 이 경로로 접속하면 `SPIway.hex` 라는 파일이 존재하는데 이 파일을 TeensyLoader를 이용하여 Teensy2.0에 프로그래밍 해준다. 그래야 SPIway 툴을 이용할 수 있다.
-->
Program Teensy 2.0 using Teensy Loader. You can program the board using `git_NORway_folder/SPIway/Release/SPIway.hex` file.

### 4.4. Step 4 - SPIway - info

![SPIWay_info](https://user-images.githubusercontent.com/48618245/101597993-d28e7e00-3a3a-11eb-9eb9-4ee60e5c0cfe.png)


```bash
.\python.exe .\NORway\SPIway.py COM12 info
```

<!--
위 명령어를 입력했을 때 info가 제대로 출력되면 모든 준비가 완료된 것이다!
-->
If you can see information properly, the Teensy is ready for dumping.

### 4.5. Step 5 - SPIway - dump

![SPIWay_dump](https://user-images.githubusercontent.com/48618245/101598053-e89c3e80-3a3a-11eb-8b33-8e291ecb27e0.png)


```bash
.\python.exe .\NORway\SPIway.py COM12 dump C:\Users\sugar\Desktop\orig1.bin
```

<!--
위 명령어를 입력하여 덤프를 수행한다. 한 3~5분 정도 기다리면 덤프가 완료된다.
-->
Dump the sflash using above command. It takes 3~5 minutes.

### 4.6. Dump Result

![sflash_dump](https://user-images.githubusercontent.com/48618245/101598102-ff429580-3a3a-11eb-915f-7ae364a14720.png)

<!--
`SONY COMPUTER ENTERTAINMENT INC` 가 나온다면 정상적으로 덤프가 완료된 것이다.
-->
If you do the process correctly, you can see `SONY COMPUTER ENTERTAINMENT INC`.

### 4.7. Plus

```
-SPIway.py

		* Flash Teensy with "\SPIway\Release\SPIway.hex"
		At the command prompt enter "SPIway.py" to display help.

		first make sure that you are able to read the SPI chip info. Do this by using
		the info command.

		get information:
		SPIway.py COMx info

		dump:
		SPIway.py COMx dump filename

		write:
		SPIway.py COMx write filename

		write with verify:
		SPIway.py COMx vwrite filename

		erase entire chip:
		SPIway.py COMx erasechip
```

<!--
dump 뿐만 아니라 write도 가능하다. 실제로 PS4에서 NOR 칩의 일부 섹션이 손상되어 BLOD (Blue Light of Death) 문제가 발생한 경우, sflash를 덤프하고 `00 00 00 00..` 영역을 `FF FF FF FF...` 로 덮어 쓴 뒤 write 해 줌으로써 수리를 하기도 했다. 이 write 기능을 추후에 이용할 수 있지 않을까 싶다.
-->

We can write data to chip as well as dump data. Maybe we can use this write function.

## 5. Reference
> - [SYSGLITCH_DOWNGRADE (2).pdf](https://gofile.io/d/sCK68r)
> - [PS4 SysGlitch Tool and SysCon Glitching Pinout by VVildCard777](https://www.psxhax.com/threads/ps4-sysglitch-tool-and-syscon-glitching-pinout-by-vvildcard777.7545/)
> - [PS4 NOR chip repair that displays signs of a BLOD](https://gbatemp.net/threads/ps4-nor-chip-repair-that-displays-signs-of-a-blod.569955/)
> - [PS4 Aux Hax 2: Syscon](https://fail0verflow.com/blog/2018/ps4-syscon/)


---

### Contents <!-- omit in toc -->
[Main](https://github.com/Hacker-s-PlayStation/PlayStation4-Hacking-Guideline-ENG/blob/main/README.md)<br>

#### Introduction <!-- omit in toc -->
[1. Jailbreak](https://github.com/Hacker-s-PlayStation/PlayStation4-Hacking-Guideline-ENG/blob/main/1_introduction/Jailbreak.md)<br>
[2. PS4 Open Source](https://github.com/Hacker-s-PlayStation/PlayStation4-Hacking-Guideline-ENG/blob/main/1_introduction/PS4_Open_Source.md)<br>
[3. Tools](https://github.com/Hacker-s-PlayStation/PlayStation4-Hacking-Guideline-ENG/blob/main/1_introduction/Tools.md)<br>
[4. Related Work](https://github.com/Hacker-s-PlayStation/PlayStation4-Hacking-Guideline-ENG/blob/main/1_introduction/Related_Work.md)<br>

#### Methodology <!-- omit in toc -->
[1. WebKit](https://github.com/Hacker-s-PlayStation/PlayStation4-Hacking-Guideline-ENG/blob/main/2_methodology/WebKit.md)<br>
[2. Hardware](https://github.com/Hacker-s-PlayStation/PlayStation4-Hacking-Guideline-ENG/blob/main/2_methodology/Hardware.md)<br>
[3. Library](https://github.com/Hacker-s-PlayStation/PlayStation4-Hacking-Guideline-ENG/blob/main/2_methodology/Library.md)<br>

#### Conclusion <!-- omit in toc -->
[Conclusion](https://github.com/Hacker-s-PlayStation/PlayStation4-Hacking-Guideline-ENG/blob/main/3_conclusion/Conclusion.md)
