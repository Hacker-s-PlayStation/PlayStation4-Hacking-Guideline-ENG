### Page Contents <!-- omit in toc -->
- [1. Introduction](#1-Introduction)
  - [1.1. What is WebKit?](#11-What-is-webkit)
- [2. Build WebKit](#2-Build-webkit)
  - [2.1. WebKit download](#21-webkit-download)
  - [2.2. Build JSC](#22-Build-jsc)
  - [2.3. Build GTK](#23-Build-gtk)
  - [2.4. Build PS4 Webkit](#24-Build-ps4-webkit)
  - [2.5. Docker Environment](#25-docker-environment)
- [3. Feature of PS4 WebKit](#3-Feature-of-ps4-webkit)
  - [3.1. NO JIT](#31-no-jit)
  - [3.2. NO Garbage Collector](#32-no-garbage-collector)
  - [3.3. NO WASM](#33-no-wasm)
  - [3.4. Return Oriented Programming, Jump Oriented Programming](#34-return-oriented-programming-jump-oriented-programming)
    - [3.4.1. FW 6.20 WebKit exploit](#341-fw-620-webkit-exploit)
- [4. Sanitizer](#4-sanitizer)
  - [4.1. Introduction](#41-Introduction)
    - [4.1.1. AddressSanitizer(ASan)](#411-addresssanitizerasan)
    - [4.1.2. MemorySanitizer(MSan)](#412-memorysanitizermsan)
    - [4.1.3. UndefinedBehaviorSanitizer(UBSan)](#413-undefinedbehaviorsanitizerubsan)
  - [4.2. Build](#42-Build)
    - [4.2.1. Step 1 - Compile flag setting](#421-step-1---Compile-flag-setting)
    - [4.2.2. Step 2 - Build Configuration](#422-step-2---Build-configuration)
    - [4.2.3. Step 3 - clang](#423-step-3---clang)
    - [4.2.4. Step 4 - Troubleshooting](#424-step-4---Troubleshooting)
    - [4.2.5. Step 5 - Test](#425-step-5---Test)
  - [4.3. Drawback](#43-Drawback)
- [5. 1-day Vulnerability Analysis Methodology](#5-1-day-Vulnerability-Analysis-Methodology)
  - [5.1. bugs.chromium](#51-bugschromium)
  - [5.2. exploit-db](#52-exploit-db)
  - [5.3. Bugzilla](#53-bugzilla)
  - [5.4. WebKit regression test](#54-webkit-regression-test)
- [6. Future Work](#6-future-work)
  - [6.1. PS4 window property](#61-ps4-window-property)
  - [6.2. openDatabase](#62-opendatabase)
- [7. Reference](#7-reference)

---
# WebKit <!-- omit in toc -->
## 1. Introduction
### 1.1. What is WebKit?

<!--
APPLE 에서 개발한 Safari, Chrome 등의 브라우저에서 사용되는 Open Source 렌더링 엔진이다. PS4 내부 브라우저에서도 WebKit을 사용한다. 그렇기에 우리는 해당 PS4의 웹킷을 attack vector로 삼았다.

그러나 WebKit에서 User-Agent에 나오는 버전을 <strong>Freezing</strong> 하고 있어서 정확한 버전을 확인 할 수 없었고, PS4 WebKit ChangeLog를 확인해 보니 <strong>2018-12-16</strong> 이후로 SONY에서 자체적으로 fork를 해서 커스터마이징 한 것으로 추정된다. <span id="webkit">PS4 Webkit은 [이 곳](https://doc.dl.playstation.net/doc/ps4-oss/webkit.html)에서 다운 받을 수 있다.</span>
-->

WebKit is an opensource browser engine used in Safari and Chrome. PS4 also uses WebKit, thus we tried to find vulnerability of WebKit. First, we tried to figure out the version of WebKit. We could not use the User-Agent information because Apple stopped updating the User-Agent information. Instead, we used the PS4 WebKit source code. You can download on [this site](https://doc.dl.playstation.net/doc/ps4-oss/webkit.html). We assumed that Sony had forked the source code from main repository around 16th December 2018, because the latest date in the ChangeLog of the source code is 16th December 2018.

## 2. Build WebKit
### 2.1. WebKit download

<!--
다음 명령어를 통해 Webkit 소스 코드를 다운 받을 수 있다.
-->

Environment : Ubuntu 18.04<br>
You can download WebKit using below command.

```bash
git clone git://git.webkit.org/WebKit-https.git WebKit
```

<!--
이후 분석하고자 하는 version으로 git checkcout 해주면 된다.
해당 실습은 2018-12-16일 기준으로 checkout을 했고, 우분투 18.04로 진행 하였다.
-->

Checkout to the commit that we assumed Sony had forked the code from main repository.

```bash
git log --before="2018-12-16"
git checkout (해당 커밋의 hash)
```

<!--
참고로 다음 Webkit을 빌드하기전에 다음 프로그램들이 사전에 설치되어 있어야 한다.
또한 version에 따라 설치에 필요한 것들이 다를 수 있으니 그때마다 설치해 주어야 한다.
-->



### 2.2. Build JSC

Before building webkit JSC, you need some prerequisite programs. The prerequisite programs may vary depending on the WebKit version.

```bash
sudo apt install cmake ruby libicu-dev gperf ninja-build
```

<!--
JSC 빌드 명령어는 다음과 같다. **빌드 명령어에 나오는 모든 경로는 웹킷 루트 디렉토리를 기준으로 한다.**
-->
JavaScriptCore(JSC) is a module that processes javascript. Building JSC command is like below. **All pathes in the command based on the WebKit root directory. In other words, present working directory is WebKit root directory.**

```bash
./Tools/Scripts/build-webkit --jsc-only --debug
```

- `jsc-only` : build only jsc
- `debug` : enable debug mode (You can use some functions that only can use in debug mode like `describe`)

<!--
이후 빌드에 성공하면 다음과 같이 JSC를 실행했을 때 command line이 뜨는 것을 확인 할 수 있다.
-->

After building complete, you can get the javascript interpreter as below.

```bash
$ ./WebKitBuild/Debug/bin/jsc 
>>> 1+2
3
>>> 
```

### 2.3. Build GTK

<!--
GTK를 빌드하기 위해서는 먼저 다음과 같은 선수 작업이 필요하다.
-->

If you build GTK, you can get mini-browser which is a browser program having minimum features of the browser. We can test WebCore, page rendering module, using mini-browser. Before building webkit GTK, you need some prerequisite programs. The prerequisite programs may vary depending on the WebKit version.

```bash
./Tools/gtk/install-dependencies

apt install libwoff-dev flatpak flatpak-builder python-pip

pip install pyyaml
```

`xdg-dbus-proxy` and `bwrap 0.3.1` are also needed.
- [xdg-dbus-proxy](https://github.com/flatpak/xdg-dbus-proxy)
- [bwrap 0.3.1](https://launchpad.net/ubuntu/+source/bubblewrap/0.3.1-1ubuntu1)

<!--
위 링크들을 참고하여 설치한 뒤 `./Source/WebKit/UIProcess/gtk/WaylandCompositor.cpp` 파일에 `#include <EGL/eglmesaext.h>` 헤더를 한 줄 추가해야 한다. EGL_WAYLAND_BUFFER_WL이 없다는 오류가 뜰 수 있기 때문이다.

모든 선수 작업을 마무리 한 뒤 `./Tools/Scripts/build-webkit --gtk` 를 실행하면 된다. (실행 할 때 RAM 16GB 정도 할당 권장)
-->

And, you have to add `#include <EGL/eglmesaext.h>` header to `./Source/WebKit/UIProcess/gtk/WaylandCompositor.cpp` file. If not, you may see a build error `EGL_WAYLAND_BUFFER_WL is not defined`. The below command is building command. We recommend your computer has at least 16GB ram.

```bash
./Tools/Scripts/build-webkit --gtk
```

You can run minibrowser using below command.

```bash
./Tools/Scripts/run-minibrowser --gtk
```

![image](https://user-images.githubusercontent.com/45416961/101721228-cbbf4400-3aea-11eb-8895-957472579115.png)

If you want to execute a html file, give the file as an argument of the script.

```bash
./Tools/Scripts/run-minibrowser --gtk index.html
``` 

### 2.4. Build PS4 Webkit

<!--
서론에서 [언급](#webkit)했던 PS4 WebKit을 다운 받은 뒤 열어 보면 다음과 같이 `WebKit-601.2.7-800`과 `WebKit-606.4.6-800` 두 개의 폴더가 있음을 확인 할 수 있다. (8.00 기준)
-->

Now, let's see the source code of the PS4 WebKit. As I mentioned in Introduction section, you can download the source code on [this site](https://doc.dl.playstation.net/doc/ps4-oss/webkit.html). When you extract the .zip file, you can see another two compressed file, `WebKit-601.2.7-800` and `WebKit-606.4.6-800` (based on 8.00 version file).

<img width="278" alt="webkit" src="https://user-images.githubusercontent.com/47859343/101600855-3adf5e80-3a3f-11eb-95d7-a170b238a0dc.png">

<!--
추정상 JSC 프로세스와 WebCore 프로세스로 분류해 둔 것 같다. [PS4 Dev Wiki](https://www.psdevwiki.com/ps4/Working_Exploits#JiT_removed_from_webbrowser)에 1.76이후 두개의 프로세스로 분할 되었다는 내용이 언급되어 있다.
-->

We think that there are two files because PS4 WebKit process has been split into two processes, JSC and WebCore. For more information, please refer [PS4 Dev Wiki](https://www.psdevwiki.com/ps4/Working_Exploits#JiT_removed_from_webbrowser).

> On FW <= 1.76, you could map RWX memory from ROP by abusing the JiT functionality and the sys_jitshm_create and sys_jitshm_alias system calls. This however was fixed after 1.76, as WebKit has been split into two processes. One handles javascript compilation and the other handles other web page elements like image rendering and DOM. The second process will request JiT memory upon hitting JavaScript via IPC (Inter-Process Communication). Since we no longer have access to the process responsible for JiT, we can no longer (at least currently), map RWX memory for proper code execution unless the kernel is patched.

<!--
또한, PC 상에서 GTK는 아예 빌드가 안 되고 JSC는 606 version만 빌드가 된다.
-->

We failed to build GTK on PC using the PS4 webkit source code. Only JSC in 606 version was built.

### 2.5. Docker Environment

<!--
[여기](https://hub.docker.com/r/gustjr1444/webkit/tags?page=1&ordering=last_updated)에 들어가면 그동안 우리가 취약점 분석을 위해 구축해둔 Webkit Docker 환경들을 다운 받을 수 있다. 여러 CVE 취약점, WebCore, ps4 Webkit의 분석을 목적으로 구축해 두었으니, 활용하면 좋을 것 같다.
-->

We made several docker containers that have various WebKit version to study several vulnerability and PS4 WebKit. If you are interested, check this [dockerhub site](https://hub.docker.com/r/gustjr1444/webkit/tags?page=1&ordering=last_updated).

## 3. Feature of PS4 WebKit
### 3.1. NO JIT

<!--
Browser exploit 에서 주로 사용하는 기법이 `JIT`을 활용해서 fake object 와 RWX 메모리 영역을 만들어서 공격을 시도하는 것이다. 그러나 PS4의 브라우저에서는 JIT이 비활성화 되어 있다.

다음과 같이 UART LOG로 확인해 보면 JIT이 비활성화 되있는 것을 알 수 있다.
-->

Many WebKit exploits uses JIT(Just In Time) compiler. However, JIT is disabled in PS4 WebKit. You can check this in UART Log.

<img width="1639" alt="스크린샷 2020-12-09 오후 7 43 46" src="https://user-images.githubusercontent.com/47859343/101619723-011a5200-3a57-11eb-9e6e-d2813fca28fb.png">

### 3.2. NO Garbage Collector

<!--
JIT과 마찬가지로 Browser exploit에서 활용되는 Garbage Collector도 `PS4`에서는 비활성화 되어 있다.

마찬가지로 UART LOG를 보면 비활성화 되어 있음을 확인 할 수 있다.
-->

Garbage Collector is a module that manage memory usage. Many WebKit exploits also uses GC, usually they allocate many objects to trigger garbage collecting. However, GC is disabled in PS4 WebKt. You can check this in UART Log.

<img width="732" alt="스크린샷 2020-12-09 오후 7 44 04" src="https://user-images.githubusercontent.com/47859343/101620296-b4834680-3a57-11eb-830f-620004bc519d.png">

### 3.3. NO WASM

<!--
`WebAssembly` 또한 PS4 브라우저에서 지원을 안한다. WebAssembly 객체를 만들 때 오류가 발생하면 alert로 메세지를 띄우게끔 테스트를 해 보았더니,
-->

Also, PS4 WebKit does not support `WebAssembly` module.

```javascript
<!DOCTYPE html>
<html>
  <head>
  </head>
  <body>
    <script>
        try{
          var memory = new WebAssembly.Memory({initial:10, maximum:100});
        }catch(error){
          alert(error);
        }
    </script>
  </body>
</html>
```

![nowasm](https://user-images.githubusercontent.com/47859343/101623898-73d9fc00-3a5c-11eb-85b9-52b1717e9d80.jpeg)

<!--
PS4에서 오류가 발생하여 `"ReferenceError: Can't find variable: WebAssembly"`이 출력 되는 것을 확인 할 수 있었다. 확실히 WebAssembly는 없다.
-->

### 3.4. Return Oriented Programming, Jump Oriented Programming

<!--
> 요약하자면 PS4 WebKit에는 JIT, GC, WebAssembly가 존재하지 않는다.

그렇기에 ROP, JOP 기법을 사용해서 exploit을 해야 한다. 우리는 이전 Jailbreak 사례들 중에서 이러한 기법이 사용된 케이스들을 분석하였다. 그 중 6.20 Jailbreak 사례에 대해서 간략히 언급하고 넘어가도록 하겠다.
-->

As We mentioned above, PS4 WebKit doesn't have JIT, GC, and WebAssembly. Thus, we have to use classical approach, JOP(Jump Oriented Programming) and ROP(Return Oriented Programming). We analyzed previous version(6.20) jailbreak exploit to study JOP, ROP technique. Let us briefly explain about this example.

<!--
- ROP(Return Oriented Programming) : ROP는 RET 명령어를 이용하여 프로그램의 흐름을 제어하는 Exploit 기술 이다. 공격자가 실행 공간 보호(NXbit) 및 코드 서명(Code signing)과 같은 보안 방어가 있는 상태에서 코드를 실행할 수 있게 해준다.<sup id="head1">[1](#foot1)</sup>
- JOP(Jump Oriented Programming) : JOP는 JMP 명령어를 이용하여 프로그램의 흐름을 제어하는 Exploit 기술 이다.<sup id="head2">[2](#foot2)</sup>
-->

- ROP(Return Oriented Programming) : ROP is a technique that control program's flow using RET command. Attacker can bypass mitigations like NXbit and Code signing.<sup id="head1">[1](#foot1)</sup>
- JOP(Jump Oriented Programming) : JOP is a technique that control program's flow using JMP command.<sup id="head2">[2](#foot2)</sup>

#### 3.4.1. FW 6.20 WebKit exploit

<!--
6.20 버전의 WebKit exploit에 CVE-2018-4441 취약점이 이용되었다. PoC는 아래와 같다.
-->

The 6.20 PS4 WebKit exploit code uses CVE-2018-4441 vulnerability. PoC is like below.

```javascript
function main() {
    let arr = [1];

    arr.length = 0x100000;
    arr.splice(0, 0x11);

    arr.length = 0xfffffff0;
    arr.splice(0xfffffff0, 0, 1);
}

main();
```
```
public length, vector length, m_numValuesInVector
```

<!--
- public length : array.length를 했을 때 출력되는 길이
- vector length : 실제로 할당된 공간의 길이
- m_numValuesInVector : 실제로 할당된 element의 개수
-->

- public length : Printed length when print array.length.
- vector length : The length of actually allocated memory space.
- m_numValuesInVector : The number of actually allocated elements.

<!--
해당 PoC에서`m_numValuesInVector` 값이 underflow로 인해 매우 큰 값으로 변경된다. 그리고 이후에 `length` 속성을 변경해줌으로써 `public length`와 `m_numValuesInVector`가 같은 값을 갖도록 한다.
-->

In the PoC, `m_numValuesInVector` value changes into very large value because of underflow. And then make `public length` and `m_numValuesInVector` have same value by changing `length` property.

```c++
if (storage->hasHoles() || storage->inSparseMode() || shouldUseSlowPut(indexingType()))
        return false;
```

<!--
이런 경우 `unshiftCountWithArrayStorage` 함수 내부에서 위 조건문을 우회하고 마치 배열에 hole이 없는 것처럼 `unshift` 작업을 진행할 수 있다. 이 과정에서 OOB access가 발생한다.
-->

In this case, we can bypass above conditional statement in `unshiftCountWithArrayStorage` function. This allows `unshift` task as if there is no hole in array. In this process, OOB access occurs.

![image](https://user-images.githubusercontent.com/40509850/101981224-995e4400-3cae-11eb-903d-438598f3cad9.png)

<!--
본 취약점을 이용한 exploit 기법은 [공개된 코드](https://github.com/Cryptogenic/PS4-6.20-WebKit-Code-Execution-Exploit)를 참고하여 분석했다. 그리고 해당 공격 기법에 대한 경험치를 쌓기 위해 PS4를 대상으로 한 exploit 코드를 PC에서 재현하고자 포팅을 진행했다. 그 과정에서 재현이 잘 되지 않는 부분이 있어서 취약점 트리거 로직을 리팩토링 하기도 했다. 추후 실습을 진행하고자 한다면 참고하자.
-->

We analyzed this exploit using [this code](https://github.com/Cryptogenic/PS4-6.20-WebKit-Code-Execution-Exploit). And We tried reproducing this PS4 WebKit exploit code on the PC to improve our attack skill. In this process, we did code refactoring to reproduce on the PC.

## 4. Sanitizer
### 4.1. Introduction

<!--
Sanitizer는 버그를 감지해 주는 도구이다. 종류에 따라 탐지할 버그의 대상이 달라지며, 목적에 맞게 사용할 수 있다. 일반적으로 clang을 이용하여 컴파일을 할 때 Sanitizer 관련 플래그를 함께 입력해 주면 Sanitizer를 쉽게 붙일 수 있다.
-->

Sanitizer is a tool that detects bugs. Depending on the sanitizer type, the target of the bug to be detected can be different. So, you can choose different sanitizer depending on the purpose. Generally, we can easily enable sanitizer giving sanitizer flags to clang.

#### 4.1.1. AddressSanitizer(ASan)

<!--
buffer-overflow 및 heap use-after-free를 포함한 메모리 액세스 버그는 C 및 C++과 같은 프로그래밍 언어의 심각한 문제로 남아 있다. AddressSanitizer는 힙, 스택 및 전역 객체에 대한 out-of-bounds 액세스와 use-after-free 버그를 탐지해 주는 도구이다.<sup id="head3">[3](#foot3)</sup>
-->

C and C++ programming language have severe problems about memory access bugs like buffer-overflow and use-after-free. AddressSanitizer detect use-after-free bug and out-of-bounds access of heap, stack and global objects. <sup id="head3">[3](#foot3)</sup>

#### 4.1.2. MemorySanitizer(MSan)

<!--
MemorySanitizer는 초기화 되지 않은 변수를 읽는 경우를 탐지해 주는 도구이다.<sup id="head4">[4](#foot4)</sup>
-->

MemorySanitizer is a tool that detects memory issues like using uninitialized variable.<sup id="head4">[4](#foot4)</sup>

#### 4.1.3. UndefinedBehaviorSanitizer(UBSan)

<!--
UndefinedBehaviorSanitizer는 undefined behavior를 탐지하는 빠른 도구이다. 컴파일 타임에 프로그램을 수정하며 프로그램 실행 중 정의되지 않은 다양한 행위들을 포착한다.<sup id="head5">[5](#foot5)</sup>
-->

UndefinedBehaviorSanitizer is a tool that detect various undefined behavior.<sup id="head5">[5](#foot5)</sup>

### 4.2. Build

<!--
WebKit 같은 경우는 빌드를 할 때 perl 기반의 스크립트를 이용하게 된다. 또한 스크립트 중에서 빌드 환경설정을 해주는 스크립트가 존재하는데, 여기에서 Sanitizer 옵션을 줄 수 있다. (아래 명령어 참고<sup id="head6">[6](#foot6)</sup>)
-->

When building WebKit, it uses perl script. And there is a script that sets build-configuration. We can set sanitizer option using the script.<sup id="head6">[6](#foot6)</sup>

```bash
./Tools/Scripts/set-webkit-configuration --release --asan
./Tools/Scripts/build-webkit
```
-  `build-webkit` : WebKit build script
-  `set-webkit-configuration` : WebKit set configuration script
-  release/debug : Release mode or debug mode.

<!--
이제 각 Step 별로 Sanitizer를 붙여서 빌드를 시도해 볼 차례이다.
-->

Now, It's time to try building webkit with sanitizer.

> **Environment** : Ubuntu 18.04 64bit

#### 4.2.1. Step 1 - Compile flag setting

<!--
컴파일 플래그를 설정할 수 있는 파일은 `./Source/cmake/WebKitCompilerFlags.cmake`이다. 이 파일의 내용을 수정하여 우리가 원하는 Sanitizer를 붙일 수 있다. 참고로 최신 버전의 `WebKitCompilerFlags.cmake` 에는 address, undefined, thread, memory, leak과 같은 flag를 적용할 수 있게끔 분기 로직이 존재한다. (아래 코드 참고)
-->

To change sanitizer, we have to modify `./Source/cmake/WebKitCompilerFlags.cmake` file. Like below code, latest version WebKit's script has conditional branch that helps us to select sanitizer.

```javascript
foreach (SANITIZER ${ENABLE_SANITIZERS})
    if (${SANITIZER} MATCHES "address")
        WEBKIT_PREPEND_GLOBAL_COMPILER_FLAGS("-fno-omit-frame-pointer -fno-optimize-sibling-calls")
        set(SANITIZER_COMPILER_FLAGS "-fsanitize=address ${SANITIZER_COMPILER_FLAGS}")
        set(SANITIZER_LINK_FLAGS "-fsanitize=address ${SANITIZER_LINK_FLAGS}")

    elseif (${SANITIZER} MATCHES "undefined")
        WEBKIT_PREPEND_GLOBAL_COMPILER_FLAGS("-fno-omit-frame-pointer -fno-optimize-sibling-calls")
        # -fsanitize=vptr is incompatible with -fno-rtti
        set(SANITIZER_COMPILER_FLAGS "-fsanitize=undefined -frtti ${SANITIZER_COMPILER_FLAGS}")
        set(SANITIZER_LINK_FLAGS "-fsanitize=undefined ${SANITIZER_LINK_FLAGS}")

    elseif (${SANITIZER} MATCHES "thread" AND NOT MSVC)
        set(SANITIZER_COMPILER_FLAGS "-fsanitize=thread ${SANITIZER_COMPILER_FLAGS}")
        set(SANITIZER_LINK_FLAGS "-fsanitize=thread ${SANITIZER_LINK_FLAGS}")

    elseif (${SANITIZER} MATCHES "memory" AND COMPILER_IS_CLANG AND NOT MSVC)
        set(SANITIZER_COMPILER_FLAGS "-fsanitize=memory ${SANITIZER_COMPILER_FLAGS}")
        set(SANITIZER_LINK_FLAGS "-fsanitize=memory ${SANITIZER_LINK_FLAGS}")

    elseif (${SANITIZER} MATCHES "leak" AND NOT MSVC)
        set(SANITIZER_COMPILER_FLAGS "-fsanitize=leak ${SANITIZER_COMPILER_FLAGS}")
        set(SANITIZER_LINK_FLAGS "-fsanitize=leak ${SANITIZER_LINK_FLAGS}")

    else ()
        message(FATAL_ERROR "Unsupported sanitizer: ${SANITIZER}")
    endif ()
endforeach ()
```

<!--
하지만 본 프로젝트에서는 이전에 언급했듯 2018-12-16 기준으로 fork 한 WebKit을 이용했기 때문에, 과정이 다소 복잡해진다. (아래 코드 참고)
-->

However, in this project, we use previous version WebKit, forked from 16th December 2018 commit, thus the process becomes slightly complicated. Below code is the compile flag script of old version WebKit.

```javascript
if (ENABLE_ADDRESS_SANITIZER)
    WEBKIT_PREPEND_GLOBAL_COMPILER_FLAGS(-fno-omit-frame-pointer
                                            -fno-optimize-sibling-calls)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=address")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address")
    set(CMAKE_EXE_LINKER_FLAGS "-lpthread ${CMAKE_EXE_LINKER_FLAGS} -fsanitize=address")
    set(CMAKE_SHARED_LINKER_FLAGS "-lpthread ${CMAKE_SHARED_LINKER_FLAGS} -fsanitize=address")
endif ()
```

<!--
해당 버전에서는 Asan만 디폴트로 제공하기 때문에 다른 플래그를 주려면 소스코드를 직접 수정해야만 한다. WebKit 하위 디렉토리 중 Source라는 디렉토리에서 `-fsanitize=address` 로 설정해주는 부분을 찾아서 모두 원하는 옵션으로 변경해 주면 된다. `Source` 디렉토리 내부에서 grep 명령어로 검색해도 되는데, 기타 에디터를 이용하는 경우 검색 기능을 이용하여 찾는 것이 당연히 좋다. 그리고 Asan 이외의 Sanitizer로 빌드하고자 하는 경우 옵션이 상이할 수 있으므로 어떤 옵션이 필요한지 체크한 후 추가해 주어야 한다.
-->

In this version, only Asan can be used by default. Thus, you have to modify the script to change sanitizer. We found all parts that set sanitizer option `-fsanitize=address` from the files in the WebKit source code directory and then changed them all. 

```c
WEBKIT_PREPEND_GLOBAL_COMPILER_FLAGS(-fno-omit-frame-pointer -fno-optimize-sibling-calls)
``` 

<!--
버전에 따라 다를 수 있겠지만 2018-12-16 기준으로는 `WebKitCompilerFlags.cmake`에서 위 코드의 인자를 다른 옵션으로 변경해 주었다.
-->

We changed all the `-fsanitize=address` options in `WebKitCompilerFlags.cmake` file. This may vary depending on WebKit version.

- AddressSanitizer(ASan) : `-fsanitize=address`
- MemorySanitizer(MSan) : `-fsanitize=memory`
  - Option : `-fno-omit-frame-pointer -fsanitize-memory-track-origins`
- UndefinedBehaviorSanitizer(UBSan) : `-fsanitize=undefined`

#### 4.2.2. Step 2 - Build Configuration

```bash
./Tools/Scripts/set-webkit-configuration --release --asan
```

<!--
모든 옵션을 변경했다면 Asan 빌드를 활성화 해야 한다. release/debug는 자유롭게 선택하면 된다. 해당 작업을 해줘야 Sanitizer를 붙여서 빌드가 되고, 그 과정에서 원래는 Asan이 적용되어야 했던 부분이 우리가 원하는 Sanitizer로 변경될 것이다.
-->

After changing all options, you have to enable Asan build option. Originally, WebKit will be built with Asan, but if you modify sanitize option, WebKit will be build with sanitizer that you chose.

#### 4.2.3. Step 3 - clang

<!--
만약 우분투에서 해당 작업을 수행한다면 디폴트로 gcc 컴파일러를 통해 빌드가 될 것이다. 아쉽게도 gcc에서 `-fsanitize=MemorySanitizer`로 빌드시 에러가 발생한다. 이러한 옵션은 clang 컴파일러를 이용해야 하는데 환경변수로 기본 컴파일러를 지정해 주면 간단히 해결되는 문제이다.
-->

Because the default compiler of Ubuntu is gcc, you have to change compiler to clang. Only clang compiler has various sanitizer options (Gcc has only addressSanitizer.). Thus, you have to set environment variable like below.

```bash
export CC=/usr/bin/clang
export CXX=/usr/bin/clang++
```

<!--
컴파일러가 clang으로 잘 세팅 되었는지, ASan 빌드가 잘 활성화 되었는지 확인하기 위해서는 `CMakeCache.txt` 파일을 확인해 보면 된다.
-->

If you want to check the build compiler is clang, and ASan option is activated, you have to check `CMakeCache.txt`.

```bash
❯ pwd
/home/lee/WebKit/WebKitBuild/Release
❯ code CMakeCache.txt
```
```c
24 //CXX compiler
25 CMAKE_CXX_COMPILER:FILEPATH=/usr/bin/clang++

49 //C compiler
50 CMAKE_C_COMPILER:FILEPATH=/usr/bin/clang

243 //Enable address sanitizer
244 ENABLE_ADDRESS_SANITIZER:BOOL=ON
```
> **Wrong Case**
> - C compiler : /usr/bin/cc
> - CXX compiler : /usr/bin/c++
> - ENABLE_ADDRESS_SANITIZER:BOOL=OFF

#### 4.2.4. Step 4 - Troubleshooting

<!--
아마도 빌드는 한 번에 되지 않을 것이다. `Could NOT find Threads (missing: Threads_FOUND)` 이런 메시지가 뜨면서 빌드에 실패할 수 있다. 이 경우 가장 최상위 디렉토리에 존재하는 `CMakeLists.txt` 파일을 수정해 주면 된다.
-->

If Building Webkit fail with `Could NOT find Threads (missing: Threads_FOUND)` error, you have to modify `CMakeLists.txt` in the WebKit root directory.

```bash
❯ pwd
/home/lee/WebKit
❯ code CMakeLists.txt
```

<!--
오류가 나는 부분이 로그에 남을텐데 해당 라인 넘버로 이동한 후 그 위에 4줄의 코드를 추가해 주면 된다.
-->

Find the part that error occured, then add four lines like below.

```javascript
set(THREADS_PREFER_PTHREAD_FLAG ON) // Where the error occurred
```
```javascript
//////////////// Added //////////////////
set(CMAKE_THREAD_LIBS_INIT "-lpthread")
set(CMAKE_HAVE_THREADS_LIBRARY 1)
set(CMAKE_USE_WIN32_THREADS_INIT 0)
set(CMAKE_USE_PTHREADS_INIT 1)
/////////////////////////////////////////

set(THREADS_PREFER_PTHREAD_FLAG ON)
```
#### 4.2.5. Step 5 - Test

<!--
마지막으로 PoC를 테스트해보거나 `testmasm` 바이너리를 한 번 실행해 보자.
-->

Finally, test WebKit using PoC or `testmasm`.

```bash
# Built with --jsc-only
❯ pwd
/home/lee/WebKit/WebKitBuild/Release/bin
❯ ./testmasm
```
![msan](https://user-images.githubusercontent.com/45416961/101595740-3adb6080-3a37-11eb-815c-8a8727ee3ee4.png)

<!--
위와 같이 메시지가 뜨면 빌드에 성공한 것이다. 만약 위 사진처럼 출력되지 않는다면 `CMakeCache.txt` 파일을 보면서 컴파일러나 Asan enable 설정이 잘 되었는지 점검해 보자.
-->

If you followed the process properly, you can see some logs like above picture.

### 4.3. Drawback
> Sanitizer other than Asan is too unstable to use.

![image](https://user-images.githubusercontent.com/45416961/101727066-1cd53500-3af7-11eb-858f-12b8f7ac1b7d.png)

<!--
아마 옛날 버전의 WebKit을 사용해서 그런 것일지도 모르겠다.*(본 프로젝트에서는 WebKit 최신 버전을 이용할 일이 없어서 빌드를 해보지 않았다.)* 2018-12-16 버전으로 Msan이나 UBSan을 붙여서 테스트를 해봤더니 오탐률이 거의 100%에 육박했다. 소위 말해 '개복치' 스럽다고도 할 수 있겠다. jsc에서 `print("hello world")`만 해줘도 Memory Leak이 발생하니 그 결과가 가히 실망스럽다. 
-->

We thought about the reason why old version WebKit only support ASan. And our conclusion is that the old version WebKit itself is too unstable to use other sanitizers like MSan and UBSan. MSan detected a lot of using uninitialized variable bug when we just asked JSC to execute `print("hello world")`. And UBSan has similar problem. Thus, You might as well use ASan because it does not print errors that unrelated with PoC, at least.

## 5. 1-day Vulnerability Analysis Methodology

<!--
다음으로는 본 프로젝트에서 1-day 취약점을 분석하기 위해 수립 및 시행한 방법론을 소개하고자 한다. 아래의 사이트들에서 PS4에 사용가능한 취약점을 찾아보았다. **PS4 WebKit에서는 JIT과 GC가 꺼져있기 때문에 JIT과, GC를 사용하지 않는지 가장 먼저 파악을 했다. 그리고 Sony에서 WebKit을 2018년 12월 쯤에 fork를 했다고 하더라도, Sony 내부에서 자체적으로 자신들의 WebKit을 패치해서 나가기 때문에 해당 취약점이 2018년 12월 이후 취약점이라도, PS4 WebKit에서 패치가 되었는지 확인해야했다.** [이 곳](https://doc.dl.playstation.net/doc/ps4-oss/webkit.html)에 있는 소스코드를 보며 패치가 되었는지 확인했다.
-->

We'd like to show you the methodology exploiting PS4 WebKit using 1 day vulnerabilities. We found some vulnerabilities on below websites. **We had to figure out whether the vulnerability uses JIT or GC, because PS4 WebKit didn't have JIT and GC. Even though Sony had forked the WebKit on December 2018, they patched their WebKit themselves. Thus, we had to check whether vulnerabilities were patched, even though they were the vulnerabilities found after 2018.** We checked whether the vulnerability is patched using the PS4 WebKit source code on [this website](https://doc.dl.playstation.net/doc/ps4-oss/webkit.html).

### 5.1. bugs.chromium
![image](https://user-images.githubusercontent.com/45416961/101621717-771fb880-3a59-11eb-9eca-bce3ecbad852.png)
![image](https://user-images.githubusercontent.com/45416961/101621198-c9aca500-3a58-11eb-9b20-12056b95fa12.png)

<!--
가장 먼저 [bugs.chromium](https://bugs.chromium.org/p/project-zero/issues/list?sort=-reported&q=webkit&can=1)에서 Project-zero 팀이 report 한 취약점들을 분석하고, PS4에 포팅하고, 코드 오디팅을 수행했다. 거의 모든 취약점들을 테스트해보았지만 이미 패치가 되었거나, PoC에 사용되는 모듈이 PS4에는 존재하지 않는 경우가 대부분이었다. 특히 JSC 취약점은 대개 JIT 컴파일러를 기반으로 하기 때문에 별다른 수확은 없었다.
-->

[bugs.chromium](https://bugs.chromium.org/p/project-zero/issues/list?sort=-reported&q=webkit&can=1) is a website collecting vulnerabilities discovered by the Project-Zero team. However, they were almost patched by Sony, or using JIT compiler, or using modules that do not exist in PS4 WebKit. Thus, we couldn't get any available vulnerabilities.

### 5.2. exploit-db
![image](https://user-images.githubusercontent.com/45416961/101621658-6707d900-3a59-11eb-8f9a-000d03573bc7.png)

<!--
[exploit-db](https://www.exploit-db.com/) 또한 Chromium과 함께 초반에 취약점을 찾고자 부단히 방문했던 사이트이다. 아무래도 이미 존재하는 exploit들을 모아 놓은 사이트이기 때문에 Chromium에서 이미 봤던 코드들이 대부분이었고, 비교적 최신 exploit은 존재하지 않았다. 결론적으로 exploit-db에서도 원하는 바를 달성하지는 못했다.
-->

[exploit-db](https://www.exploit-db.com/) is a vulnerability archive site. We could not get results because most of the codes were duplicated codes on [bugs.chromium](https://bugs.chromium.org/p/project-zero/issues/list?sort=-reported&q=webkit&can=1).

### 5.3. Bugzilla
![image](https://user-images.githubusercontent.com/45416961/101606281-89dcc200-3a46-11eb-9fa9-0e962243c136.png)

<!--
[WebKit Bugzilla](https://bugs.webkit.org/)에 report 되는 버그들을 모니터링 하면서 실제로 exploit에 사용될만한 취약점이 있는지 탐색할 수 있다. 다만 간단한 description과 패치 내역만 보고 특정 버그가 exploitable한지 판단할 수 있는 경험치가 요구된다. 그리고 Security issue의 경우 일반 사용자들에게 액세스 권한을 주지 않는 경우가 많다. 따라서 Bugzilla만 살펴보면서 취약성이 존재하는 케이스를 찾아내기란 모래알 속 진주 찾기와도 같다. 물론 시작하는 단계에서는 말이다.
-->

[WebKit Bugzilla](https://bugs.webkit.org/) is a website reporting bugs of WebKit. However, most of them are just simple bugs, we need skill that can figure out whether this bug is exploitable by reading simple description and patch code. Moreover, general users cannot access some bug description pages if the bug is related to security issue. Thus, looking only at Bugzilla and finding exploitable bugs is like looking for a needle in a haystack.

### 5.4. WebKit regression test

<!--
> 아직 경험치가 많이 부족하다면 ChangeLog가 버그 탐색을 위한 좋은 입문 경로가 될 수 있다.
-->

> If you are not familiar with bug searching, we recommend this method.

<!--
WebKit repositorty를 조금만 들여다 보면 ChangeLog 상에 패치 내역이 아주 잘 정리되어 있다는 사실을 알 수 있다. `JSTests, LayoutTests` 디렉토리는 JavaScriptCore와 WebCore 각각에 대한 테스트 코드를 담고 있는데, 물론 이 폴더 내부에도 ChangeLog가 존재한다. ChangeLog의 패턴을 살펴보고 넘어가자.
-->

ChangeLog files in WebKit repository have patch logs. `JSTests`, `LayoutTests` directory have test codes for JavaScriptCore and WebCore respectively. And there are ChangeLog files in the directories. Let's check the ChangeLog's content.

```
2020-10-28  Robin Morisset  <rmorisset@apple.com>

  DFGIntegerRangeOptimization is wrong for Upsilon (as 'shadow' nodes are not in SSA form)
  https://bugs.webkit.org/show_bug.cgi?id=218073

  Reviewed by Saam Barati.

  The only testcase I managed to get for this bug loops forever when not crashing.
  So I use a 1s timeout through --watchdog=1000.

  * stress/bounds-checking-in-cold-loop.js: Added.
  (true.vm.ftlTrue):
```

<!--
취약점에 대한 간략한 설명 및 Bugzilla 주소, 그리고 해당 버그를 테스트 하기 위한 Regression test 코드의 경로까지 상세히 기술되어 있다. 이 패턴을 이용하여 테스트 코드만 뽑아내면 탐색 범위를 상당히 줄일 수 있을 것으로 판단했다. 그래서 자동화 스크립트를 작성한 후 코드 수집 및 테스트까지 일사천리로 진행했다.
처음에는 ASan을 붙여서 빌드한 WebKit GTK 미니브라우저에서 로그들을 수집한 후 에러가 발생한 케이스들만 분석을 진행했다. 하지만 대부분이 Null Dereferencing이나 exploit에 사용하기는 애매한 취약점들이었고, 또 Sony 측에서 자체적으로 패치를 진행한 경우들이 대부분이었다. 그래서 ASan 로그 상에서 별다른 반응이 없었던 경우들에 대해서도 패치 개핑을 하면서 분석 과정을 이어 나갔다.
-->

It has simple description of bug, address of Bugzilla site, and the test code that checks whether patch is done well. Thus, we made a script that collects and tests the test codes. First, we collected ASan logs of minibrowser. However, most of them were unexploitable bugs like null dereferencing, or patched by Sony. Thus, we also analyzed the bugs that ASan Log was empty.

![image](https://user-images.githubusercontent.com/45416961/101624972-1e065380-3a5e-11eb-943e-2a6ed5273672.png)
![image](https://user-images.githubusercontent.com/45416961/101624881-f911e080-3a5d-11eb-98da-c4ceef700f57.png)

<!--
그러던 도중 WebCore 엔진에서 pc 레지스터 컨트롤이 가능한 취약점 하나를 발견할 수 있었다. 6.72 버전에서 디버깅을 통해 레지스터 값이 임의의 값으로 변경되는 것을 확인했고, 8.01 버전에서도 레지스터 값을 확인할 순 없었지만 에러 반응이 나타나는 것을 볼 수 있었다. 다만 해당 취약점 하나만으로는 exploit을 할 수 없기에 info leak 취약점을 탐색해야만 했다. 프로젝트 기간 동안 쓸만한 info leak 취약점은 발견하지 못했다. 비록 프로젝트는 끝났지만 Future work로 시도해보면 좋을 것 같다.
-->

Finally, we found a vulnerability that can control program counter register. We checked changing rip value into arbitrary value using UART Log on PS4 6.72 version. The vulnerability was not patched in the latest PS4 WebKit source code, thus we checked that PS4 latest version(8.01 version) also crashed. However, this vulnerability just can change pc register value, thus we can not exploit the WebKit without information leak. Unfortunately, we failed to find information leak vulnerability during project term.

## 6. Future Work

<!--
PS4 브라우저에서 사용하는 property 중에서 Safari에서 사용하지 않는 property가 있는지 알아봤다.
-->

We checked some properties that used in PS4 Webkit but not in Safari.

### 6.1. PS4 window property

```html
<body>
    <form action="http://172.30.1.59:8000" method="get">
      <input type="text" id="test" name="test" value="" />
      <input type="button" value="Set" onclick=object() />
      <input type="submit" value="Submit" />
    </form>
</body>
<script>
    var a;
    function object(){
      keys = Object.keys(window);
      for (let i = 0; i < keys.length; i++) {
          a += keys[i] + " ";
      }
      document.getElementById('test').value = a;
    }
</script>
```

![image](https://user-images.githubusercontent.com/45416961/101769433-4b203800-3b2a-11eb-979e-a71af77875dc.png)

<!--
Set 버튼을 누른 후 Submit을 누르면, PS4에서 사용하는 window property들을 서버로 전송해 주는 소스코드이다.
-->

The above code gets the window properties in the PS4 WebKit. You can get window properties by clicking set button and submit button. Before using the above code, you have to change the address in form tag to your server address.

```
"GET /?test=undefineda+document+object+window+self+name+location+history+locationbar+menubar+personalbar+scrollbars+statusbar+
toolbar+status+closed+frames+length+top+opener+parent+frameElement+navigator+applicationCache+sessionStorage+localStorage
+screen+innerHeight+innerWidth+scrollX+pageXOffset+scrollY+pageYOffset+screenX+screenY+outerWidth+outerHeight+devicePixelRatio
+event+defaultStatus+defaultstatus+offscreenBuffering+screenLeft+screenTop+clientInformation+styleMedia+onabort+onblur+
oncanplay+oncanplaythrough+onchange+onclick+oncontextmenu+oncuechange+ondblclick+ondrag+ondragend+ondragenter+ondragleave+
ondragover+ondragstart+ondrop+ondurationchange+onemptied+onended+onerror+onfocus+oninput+oninvalid+onkeydown+onkeypress+
onkeyup+onload+onloadeddata+onloadedmetadata+onloadstart+onmousedown+onmouseenter+onmouseleave+onmousemove+onmouseout+
onmouseover+onmouseup+onmousewheel+onpause+onplay+onplaying+onprogress+onratechange+onrejectionhandled+onreset+onresize
+onscroll+onseeked+onseeking+onselect+onstalled+onsubmit+onsuspend+ontimeupdate+ontoggle+onunhandledrejection+onvolumechange
+onwaiting+ontransitionend+ontransitionrun+ontransitionstart+ontransitioncancel+onanimationend+onanimationiteration+
onanimationstart+onanimationcancel+crypto+performance+onbeforeunload+onhashchange+onlanguagechange+onmessage+onoffline
+ononline+onpagehide+onpageshow+onpopstate+onstorage+onunload+origin+close+stop+focus+blur+open+alert+confirm+prompt+
print+requestAnimationFrame+cancelAnimationFrame+postMessage+captureEvents+releaseEvents+getComputedStyle+matchMedia+
moveTo+moveBy+resizeTo+resizeBy+scroll+scrollTo+scrollBy+getSelection+find+webkitRequestAnimationFrame+webkitCancelAnimationFrame+
webkitCancelRequestAnimationFrame+getMatchedCSSRules+showModalDialog+webkitConvertPointFromPageToNode+webkitConvertPointFromNodeToPage+
openDatabase+setTimeout+clearTimeout+setInterval+clearInterval+atob+btoa+customElements+visualViewport+isSecureContext+fetch+ HTTP/1.1" 200 -
```

<!--
위 로그는 서버로 전송된 PS4가 사용하는 window property 정보인데, 이를 Safari에서 사용하는 window property와 비교해서 PS4에서만 사용하는 것이 존재하는지 확인해보았다. 그 중 `openDatabase`가 Safari에서는 지원하지 않는 property라는 사실을 알게 되었다.
-->

The above result is the information of window property in PS4 WebKit. We compared this result with the result of Safari. We found that `openDatabase` is the property that only used in PS4 WebKit.

### 6.2. openDatabase

<!--
`openDatabase`는 Web SQL Database를 사용하여 Database를 만들어주는 property이다. Safari에서는 race condition 문제가 자주 발생해서 더이상 지원하지 않는다고 한다. 이 [링크](https://caniuse.com/sql-storage)에 들어가서 보면 Safari 13버전부터 Web SQL Database를 지원하지 않는 것이 확인된다.

Safari에서는 race condition 문제 때문에 지원하지 않기도 하고 DEFCON에서 발표된 취약점인 Magellan 같은 경우는 Chrome에서 RCE까지 가능했다. 이러한 점들을 미루어 봤을 때 상당히 해볼만한 Attack Vector라고 생각한다. (DEFCON 발표자료 참고<sup id="head7">[7](#foot7)</sup>)
-->

`openDatabase` is a property that makes an Database using Web SQL Database. Safari does not use it anymore before openDatabase has many race condition issues. According to this [website](https://caniuse.com/sql-storage), Safari has not supported openDatabase since version 13. Moreover, Magellan, the vulnerability released in DEFCON<sup id="head7">[7](#foot7)</sup>, can do RCE in Chrome. Thus, we think `openDatabase` is quite attractive attack vector.

## 7. Reference
><b id="foot1">[[1](#head1)]</b> [ROP(Return-Oriented Programming)](https://en.wikipedia.org/wiki/Return-oriented_programming#cite_note-2)<br>
><b id="foot2">[[2](#head2)]</b> [JOP(Jump-Oriented Programming)](https://www.lazenca.net/pages/viewpage.action?pageId=16810290)<br>
><b id="foot3">[[3](#head3)]</b> Konstantin Serebryany; Derek Bruening; Alexander Potapenko; Dmitry Vyukov. ["AddressSanitizer: a fast address sanity checker"(PDF)](https://www.usenix.org/system/files/conference/atc12/atc12-final39.pdf). Proceedings of the 2012 USENIX conference on Annual Technical Conference.<br>
><b id="foot4">[[4](#head4)]</b> [MemorySanitizer - Clang 12 Documentation](https://clang.llvm.org/docs/MemorySanitizer.html)<br>
><b id="foot5">[[5](#head5)]</b> [UndefinedBehaviorSanitizer - Clang 12 Documentation](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html)<br>
><b id="foot6">[[6](#head6)]</b> [Building WebKit with Clang Address Sanitizer(ASan)](https://trac.webkit.org/wiki/ASanWebKit)<br>
><b id="foot7">[[7](#head7)]</b> [Breaking Google Home: Exploit It with SQLite(Magellan)](https://media.defcon.org/DEF%20CON%2027/DEF%20CON%2027%20presentations/DEFCON-27-Wenxiang-Qian-Yuxiang-Li-Huiyu-Wu-Breaking-Google-Home-Exploit-It-with-SQLite-Magellan.pdf)<br>


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