### Page Contents <!-- omit in toc -->
- [1. Overview](#1-overview)
  - [1.1. Uart Log](#11-uart-log)
  - [1.2. File Decryption](#12-file-decryption)
- [2. Prepare Source Code Auditing](#2-prepare-source-code-auditing)
  - [2.1. Restore Library Function Symbol](#21-restore-library-function-symbol)
- [3. Prepare Fuzzing](#3-prepare-fuzzing)
  - [3.1. PS4 Library](#31-ps4-library)
  - [3.2. Transform sprx into so](#32-transform-sprx-into-so)
  - [3.3. Encrypt / Decrypt env](#33-encrypt--decrypt-env)
  - [3.4. SPRX / ELF Header](#34-sprx--elf-header)
  - [3.5. Craft Program Header](#35-craft-program-header)
  - [3.6. Craft Dynamic Entries](#36-craft-dynamic-entries)
  - [3.7. Creating DT_STRTAB](#37-creating-dt_strtab)
    - [3.7.1. Creating DT_SYM](#371-creating-dt_sym)
  - [3.8. Creating Relocation Table](#38-creating-relocation-table)
  - [3.9. Creating Section Header](#39-creating-section-header)
  - [3.10. Testing the Library Converted to so file](#310-testing-the-library-converted-to-so-file)
- [4. Script](#4-script)
  - [4.1. Limit](#41-limit)
    - [4.1.1 Imperfect Connection of plt and got](#411-Imperfect-Connection-of-plt-and-got)
    - [4.1.2 Not Working dlsym](#412-Not-working-dlsym)
- [5. Fuzzing (Not completed)](#5-Fuzzing-not-completed)

---
# Library <!-- omit in toc -->
## 1. Overview
### 1.1. Uart Log
<!--Uart Log를 보다가 PS4에서 외부 서버에서 주기적으로 파일을 가져오는 것을 확인했다<br> -->
Looking at the UART log, we noticed that PS4 periodically requests file from external server.
![image](https://user-images.githubusercontent.com/39231485/101589311-86880d00-3a2b-11eb-9906-aafc7b2b666e.png)
```
SERVER_URL={http://ps4-system.sec.np.dl.playstation.net/ps4-system/hid_config/np/v00/hidusbpower.env}
```
<!--서버에서 전달받는 파일은 env파일이다. env파일은 PS4 서버와 기기사이에서 데이터를 주고받는 포맷으로 이에 대한 자세한 정보는 [여기](https://www.psdevwiki.com/ps4/Envelope_Files)에서 확인 가능하다. UART Log에서 출력된 해당 env파일의 처리루틴을 살펴본 결과, env파일의 복호화를 통해 xml파일이 생성되었고, 이를 libxml2 라이브러리에서 처리한다. **따라서 서버로부터 받아오는 env파일을 통신 중간에 임의로 변조하여 공격하는 새로운 시나리오를 생각해냈다. 그리고 이를 확장하여 기존에 WebKit, Freebsd 취약점을 사용하는 Jailbreak와는 다르게 프로세스 안에 있는 라이브러리의 취약점을 사용하는 것을 생각해봤다. 라이브러리 내의 취약점을 찾기 위해서 소스 코드 오디팅, 라이브러리 함수를 대상으로 퍼징을 시도한다.**-->
The file received from the externel server is a 'env' file. The env file is a format for exchaining data between PS4 server and the device, and more detailed information about this can be found at [here](https://www.psdevwiki.com/ps4/Envelope_Files). As a result of examining the processing routine of the env file in UART log, we found that an XML file was created through the decryption of the env file, and it was processed in the `libxml2` library. Therefore, we came up with a new senario in which the env file received from the server is arbitrarily altered and attacks in the middle of communication. And by extending this, we considered using vulnerability of the library in the process, unlike Jailbreak, which uses the existing WebKit and FreeBSD vulnerabilities. To find vulnerability in the library, we will try to audit the source code and fuzz to the library functions.

### 1.2. File Decryption
<!-- PS4 안의 파일들은 모두 암호화가 되어있기 때문에 복호화를 진행해야 한다. 복호화된 내용물을 모아둔 [사이트](https://darthsternie.net/ps4-decrypted-firmwares/)가 존재하여 이를 이용했다. 아쉽게도 2020년 12월 기준, 가장 최신 버전은 8.03이지만, 위 사이트에는 7.00버전까지 존재했고, 7.00버전을 분석했다.<br> -->
Since all files in PS4 are encrypted, decryption is required. There is a [Site](https://darthsternie.net/ps4-decrypted-firmwares/) that collects decrypted contents, and we used it. Unfortunately, as of December 2020, the latest firmware version is 8.03, but there were up to 7.00 version on the above site, so we analyzed the 7.00 version.

## 2. Prepare Source Code Auditing
### 2.1. Restore Library Function Symbol
<!--복호화된 sprx를 아이다로 열었을 때, 심볼은 존재하지 않는다.<br>-->
When the decoded sprx file was opened with IDA, the symbol did not exist.

![image](https://user-images.githubusercontent.com/39231485/101623622-0f1ea180-3a5c-11eb-8f63-9687c8a3624d.png)

<!--하지만 심볼 대신 NID라는 것을 통하여 함수 주소와 매치시키는데, 만약 특정 NID가 어떤 함수명인지 안다면 심볼을 복구 할 수 있을 것이다.<br>
NID와 함수명을 매치한 약 38000개의 데이터를 모아놓고, 이를 매칭시켜주는 아이다 플러그인을 만든 [사이트](https://github.com/SocraticBliss/ps4_module_loader)가 존재한다. 해당 플러그인을 사용하면 많은 함수들의 심볼들을 구할 수 있다.<br>-->
Howerver, it is possible to match the function address through the NID instead of the symbol. If we know what function name a specific NID is, we can recover the symbol.
There is a [site](https://github.com/SocraticBliss/ps4_module_loader) that IDA plug-in that collects about 38,000 data and matches them. We can get the symbols of many functions by using this plug-in.

![image](https://user-images.githubusercontent.com/39231485/101710935-d9b69a00-3ad5-11eb-9326-ff45cc95335b.png)<br>

## 3. Prepare Fuzzing
### 3.1. PS4 Library
![image](https://user-images.githubusercontent.com/39231485/101594750-8e4caf00-3a35-11eb-891e-3102d8be47be.png)

  <!-- * ps4 라이브러리는 소니가 자체적으로 만든 sprx라는 포맷을 사용한다. -->
  - The PS4 library uses a format called `sprx` made by Sony.
  - You can check the detail of the sprx format on this [website](https://www.psdevwiki.com/ps4/SELF_File_Format)
### 3.2. Transform sprx into so
<!--퍼징을 진행할 때, MITM기법으로 env파일을 변조하여 기기에 전달하는 방식은 속도가 매우 느리고, 콘솔 기기내의 code coverage를 분석하는데도 어려움이 있다. 따라서 xml 처리 루틴을 PC에서 재현한 후에 이를 이용하여 PC상에서 퍼징을 하려고 한다. sprx는 PS4 전용 포맷이기 때문에 이를 PC에서 사용할 수 있도록 하기 위해 elf 포맷으로 변경하는 것을 시도했다.-->
When fuzzing, the method of modulating the env file using the MITM technique and transmitting it to the device is very slow, and it is difficult to analyze the code coverage in the console device. Therefore, after reproducing the xml processing routine on the PC, we will fuzz on the PC using it. Since sprx is PS4 only format, we tried to transform it into elf format in order to be able to use it on PC.

### 3.3. Encrypt / Decrypt env
<!-- 변조된 xml데이터를 전달하기 위해서는, env파일 암복호화를 임의로 할 수 있도록 해야한다. [여기](https://github.com/SocraticBliss/ps4_env_decryptor)에서 env파일 복호화 코드를 구할 수 있다. 우리는 이를 참고하여 아래와같이 env파일 암호화 코드를 구현했다. -->
To deliver the altered xml data, the env file must be able to be encrypted and decrypted arbitrarily. [Here](https://github.com/SocraticBliss/ps4_env_decryptor), you can get the code for decrypting the env file. We referred to this and implemented the code for encrypting the env file as follow.

```python
from binascii import unhexlify as uhx, hexlify as hx
from Crypto.Cipher import AES
from pwn import *
import struct
import sys
import hashlib
import binascii
# Replace the 0's with the actual keys! :)

KEYS = {
     0x1 : 'C68A9B40493577E7543A2D95599F7E96', # beta_updatelist
     0x2 : '566BDD67C3B6B504EF1A39C0CCAC4BE2', # timezone
     0x3 : '8C811754ABE72C8A1B4DDCA232B8CC2A', # system_log_config
     0x4 : '8A7A15CE5D2162197CFA83B1DC773CC7', # system_log_unknown
     0x5 : 'DAAF3B0699F76E048F3C27BF0BE8951C', # bgdc_config
     0x6 : '67EEDAF267E043BEB5AD5C7A6F54C537', # wctl_config
     0x7 : '5A1F16C347F8246BF6D78BE3C4D6F2D1', # morpheus_updatelist
     0x8 : '6601692E461BBDCC856EA8DCCE063CF1', # netev_config
     0x9 : '89D5DD4B3AE3891EA307D562FF7A94D2', # gls_config
     0xA : '5CC692D06CCF5288575992292C8AB979', # hid_config
     0xC : '61762FAFA75069F616BA9143F698680B', # hidusbpower
     0xD : '9B12DA7C22443BCD2BBCF6C0185E7770', # patch_hmac_key
     0xE : 'F11B1925D93CE77086F49CCC50964556', # bgft
    0x11 : 'CEA474201F0CA2368DD837BFA1B4BD78', # system_log_privacy
    0x12 : 'CA4A06AD3C098DAB6B30972CBC4900BD', # webbrowser_xutil
    0x13 : '49B61A0B8F33BC286F856DA8CB04B8B4', # entitlementmgr_config
    0x15 : '51AE12B0CBD8EFD3598BC5118DE1A30C', # jsnex_netflixdeckeys
    0x16 : '9C4EE3E6DC82A18AA21233D535B108EC', # party_config
}

# Big Thanks to Flatz

def aes_encrypt_cbc_cts(key, iv, data):
  result = ''
  data_size = len(data)

  context = AES.new(key, AES.MODE_ECB)
  num_data_left = data_size
  block_size = 16
  offset = 0
  tmp = data
  print(hexdump(iv))
  while num_data_left >= block_size:
    inp = data[offset:offset + block_size]
    enc = ''.join(chr(ord(inp[i]) ^ ord(iv[i])) for i in xrange(block_size))
    enc = context.encrypt(enc)
    num_data_left -= block_size
    offset += block_size
    result += enc
    iv = enc
    print(hexdump(iv))

  if num_data_left > 0 and num_data_left < block_size:
    inp = result[offset - block_size:offset]
    out = context.encrypt(inp)
    for i in xrange(num_data_left):
      result += chr(ord(tmp[offset + i]) ^ ord(out[i]))


  return result

def aes_decrypt_cbc_cts(key, iv, data):
  result = ''
  data_size = len(data)

  if data_size == 0:
    return result

  context = AES.new(key, AES.MODE_ECB)
  num_data_left = data_size
  block_size = 16
  offset = 0

  while num_data_left >= block_size:
    input = data[offset:offset + block_size]
    output = context.decrypt(input)
    output = ''.join(chr(ord(output[i]) ^ ord(iv[i])) for i in xrange(block_size))
    num_data_left -= block_size
    offset += block_size
    result += output
    iv = input
    print(hexdump(iv))

  if num_data_left > 0 and num_data_left < block_size:
    input = data[offset - block_size:offset]
    output = context.encrypt(input)

    for i in xrange(num_data_left):
      result += chr(ord(data[offset + i]) ^ ord(output[i]))

  return result


def main(argc, argv):
    if argc != 3:
        raise SystemExit('\nUsage: %s <originalfile> <evilfile>' % argv[0])

    with open(argv[1], 'rb') as tmp, open("output.env", 'wb') as output, open(argv[2], 'rb') as input:

        data = input.read()
        t = tmp.read()


        id = struct.unpack('<I', t[0x8:0xC])[0]

        try:
            key = uhx(KEYS[id].replace(' ', ''))
        except:
            raise SystemExit('\nError: Invalid File!')

        size = len(data)
        iv   = t[0x20:0x30]


        print(hexdump(iv))
        data1 = aes_encrypt_cbc_cts(key, iv, data)


# setting env file format

        evil = ''
        evil += t[:0x4]
        evil += p32(0)
        evil += p32(id)
        evil += p32(0)
        evil += p64(size)
        evil += p64(0)
        evil += iv
        tmp = hashlib.sha256(data)
        evil += binascii.unhexlify(tmp.hexdigest())
        evil += t[0x50:0x150]
        evil += data1

        output.write(evil)
    print('\nSuccess!')

if __name__ == '__main__':
    main(len(sys.argv), sys.argv)
```
### 3.4. SPRX / ELF Header

<!-- - 두 포맷의 헤더 필드는 거의 동일하다. 각각의 요소만 조금씩 변형시켜주면 된다. -->
- The header fields of both formats are almost the same.

<!-- 우리가 리눅스에서 사용하는 ELF의 헤더이다.  -->
The following is the header of ELF used in Linux.

```python
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Shared object file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x4000
  Start of program headers:          64 (bytes into file)
  `:          165424 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         7
  Size of section headers:           64 (bytes)
  Number of section headers:         6
  Section header string table index: 2
```

<!-- 다음은 `SPRX`의 헤더를 읽어온 것이다. -->
And the following is the header of SPRX.

```python
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 09 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - FreeBSD
  ABI Version:                       0
  Type:                              OS Specific: (fe18)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          64 (bytes into file)
  Start of section headers:          0 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         9
  Size of section headers:           0 (bytes)
  Number of section headers:         0
  Section header string table index: 0
```

<!-- 변경해야할 필드들과 변경할 값들을 쌍으로 나열해보면 다음과 같다. -->
Let's list the fields and values to be changed in pairs.

<!-- - `OS/ABI` → 대상 운영 체제 ABI를 구별하는 필드.
    - `UNIX FreeBSD` 로 설정되어있는  값을 `0x00(System V)` 로 변경시켜준다.
- `TYPE`
    - 해당 파일이 어떤 파일인지 명시하는 필드
    - `sprx` 에서는 파일을 구분하는 값이 조금 다른데 신경쓰지 않아도 된다. 우리는 sprx파일을 so 파일로 바꾸는 것이 목적이므로 해당 필드값을 `3(shared object file)` 로 설정해주면 된다.
- `Entry point`
    - so파일이라 0으로 설정해주면 된다.
- `Start of section headers`
    - 추후에 섹션헤더를 추가한 뒤에 설정해줘야하는 offset. sprx에서는 이상하게 섹션 헤더를 사용하지 않았지만, 우리는 다른 프로그램에서 로딩할 수 있도록 몇몇 섹션들을 추가해주어야한다.
- `Number of program headers`
    - 프로그램 헤더의 갯수. 우리가 추가하거나 뺀만큼 조정해주면 된다.
- `Number of section headers`
    - 섹션 헤더들의 갯수. 추가한 만큼 나중에 늘려주면 된다. -->
- `OS/ABI`
    - Field that identifies the target OS ABI
    - It changes the value set in `UNIX FreeBSD` to `0x00(System V)`
- `TYPE`
    - Field that specify the file type
    - In `sprx`, the value for classifying files is slightly different, but don't have to worry about it. Our goal is to convert the `sprx` file into the `so` file, so we just need to set the field value to `3(shared object file)`.
- `Entry point`
    - For the `so` file, we set it to `0`.
- `Start of section headers`
    - Offset that should be set after adding the section header later.
    - Strangely, `sprx` didn't use section header, but we need to add some sections so that other program can load them.
- `Number of program headers`
    - Just adjust it as much as we add or subtract it.
- `Number of section headers`
    - Just increase it as much as we add it later.

### 3.5. Craft Program Header

<!-- - elf(.so)의 프로그램 헤더(참고용) -->
  <!-- - GNU_ 가 붙은 타입들은 필수적이지 않은 요소들이라 일단 배제하고 보아도 된다. -->
- Program header of `elf(.so)` *(for reference only)*
  - Types starting with `GNU_` are not essential, so we have excluded them for now.

```python
Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000b08 0x0000000000000b08  R      0x1000
  LOAD           0x0000000000001000 0x0000000000001000 0x0000000000001000
                 0x0000000000000b25 0x0000000000000b25  R E    0x1000
  LOAD           0x0000000000002000 0x0000000000002000 0x0000000000002000
                 0x00000000000002b8 0x00000000000002b8  R      0x1000
  LOAD           0x0000000000002e08 0x0000000000003e08 0x0000000000003e08
                 0x0000000000000440 0x0000000000000448  RW     0x1000
  DYNAMIC        0x0000000000002e18 0x0000000000003e18 0x0000000000003e18
                 0x00000000000001c0 0x00000000000001c0  RW     0x8
  NOTE           0x00000000000002a8 0x00000000000002a8 0x00000000000002a8
                 0x0000000000000020 0x0000000000000020  R      0x8
  NOTE           0x00000000000002c8 0x00000000000002c8 0x00000000000002c8
                 0x0000000000000024 0x0000000000000024  R      0x4
  GNU_PROPERTY   0x00000000000002a8 0x00000000000002a8 0x00000000000002a8
                 0x0000000000000020 0x0000000000000020  R      0x8
  GNU_EH_FRAME   0x000000000000206c 0x000000000000206c 0x000000000000206c
                 0x000000000000007c 0x000000000000007c  R      0x4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10
  GNU_RELRO      0x0000000000002e08 0x0000000000003e08 0x0000000000003e08
                 0x00000000000001f8 0x00000000000001f8  R      0x1
```

<!-- - sprx의 프로그램 헤더
    - 프로그램 헤더 타입 코드가 달라서 `readelf`로 읽어올시에 타입 이름이 이상하게 나타나지만, 대략적인 형태는 알아볼 수 있어서 다음과 같이 읽어왔다.
    - sprx에서는 elf 헤더 부분을 메모리에 로딩하지 않는다. -->
  - Program header of `sprx`
    - The type name appears strange when reading with `readelf` because the code of program header type is different, but the approximate form is recognizable, so we read it as follows.
    - sprx does not load the part of elf header into memory.

```python
Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000004000 0x0000000000000000 0x0000000000000000
                 0x000000000001ebf0 0x000000000001ebf0  R E    0x4000
  LOOS+0x1000010 0x0000000000024000 0x0000000000020000 0x0000000000020000
                 0x0000000000000430 0x0000000000000430  R      0x4000
  LOAD           0x0000000000028000 0x0000000000024000 0x0000000000024000
                 0x0000000000000198 0x00000000000001f9  RW     0x4000
  LOOS+0x1000002 0x0000000000028000 0x0000000000024000 0x0000000000024000
                 0x0000000000000018 0x0000000000000018  R      0x8
  DYNAMIC        0x000000000002b260 0x0000000000000000 0x0000000000000000
                 0x0000000000000270 0x0000000000000270  RW     0x8
  TLS            0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  R      0x1
  GNU_EH_FRAME   0x00000000000217e4 0x000000000001d7e4 0x000000000001d7e4
                 0x000000000000140c 0x000000000000140c  R      0x4
  LOOS+0x1000000 0x00000000000281a0 0x0000000000000000 0x0000000000000000
                 0x0000000000003330 0x0000000000000000  R      0x10
  LOOS+0xfffff01 0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000         0x10
```

<!-- sprx의 프로그램 헤더를 불필요한 부분을 전부 제거한 뒤에 다음과 같이 바꿀 것이다. -->
After removing all unnecessary parts of sprx's program header, change it as follow.

```python
LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000003ff0 0x0000000000003ff0  RW     0x1000
  LOAD           0x0000000000004000 0x0000000000004000 0x0000000000004000
                 0x000000000001ebf0 0x000000000001ebf0  RWE    0x4000
  LOAD           0x0000000000024000 0x0000000000024000 0x0000000000024000
                 0x0000000000003730 0x0000000000003730  RW     0x4000
  LOAD           0x0000000000028000 0x0000000000028000 0x0000000000028000
                 0x00000000000001f9 0x00000000000001f9  RW     0x4000
  DYNAMIC        0x0000000000028400 0x0000000000228400 0x0000000000228400
                 0x0000000000000200 0x0000000000000200  RW     0x10
  LOAD           0x0000000000028400 0x0000000000228400 0x0000000000228400
                 0x0000000000001000 0x0000000000001000  RW     0x1000

....
```

<!-- 추가한 부분은  -->
The part we added is below.

```python
LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000003ff0 0x0000000000003ff0  RW     0x1000
```

<!-- 위 세그먼트는 elf header를 메모리에 로딩하기 위해 추가한 세그먼트다. 기존 세그먼트들 위에 새로운 세그먼트를 추가한 것이므로 파일 `offset, virtual/physical address`에 각각 해당 크기만큼을 더해줘야한다.

세그먼트를 추가한 뒤에는 각각 원래 `LOAD` 세그먼트들의 offset + 추가한 세그먼트의 크기에다가 

기존의 세그먼트 컨텐츠를 다 옮겨주면 된다. (코드영역, 문자열 정보 등)

이제 여기서부터 기존의 elf와 다른 부분을 차근차근 고쳐나가면 되는데, 자세한 내용은 후술하면서 같이 언급할 예정이다. -->

The segment above is added to load the elf header into memory. Since a new segment is added on top of existing segments, the corresponding size must be added to the file `offset, virtual/physical address`.

After adding segments, we changed the virtual address and physical address by adding the size of added segment to the offset of original `LOAD` segment respectively.

From here on, it is enough to fix the parts that are different from the existing `elf` step by step, and details will be mentioned later.

### 3.6. Craft Dynamic Entries

```python
LOAD:0000000000228410                 Elf64_Dyn <5, 2000h>    ; DT_STRTAB
LOAD:0000000000228420                 Elf64_Dyn <6, 24580h>   ; DT_SYMTAB
LOAD:0000000000228430                 Elf64_Dyn <0Ah, 1FF0h>  ; DT_STRSZ
LOAD:0000000000228440                 Elf64_Dyn <0Bh, 18h>    ; DT_SYMENT
LOAD:0000000000228450                 Elf64_Dyn <15h, 0>      ; DT_DEBUG
LOAD:0000000000228460                 Elf64_Dyn <9, 18h>      ; DT_RELAENT
LOAD:0000000000228470                 Elf64_Dyn <18h, 0>      ; DT_BIND_NOW
LOAD:0000000000228480                 Elf64_Dyn <7, 25B58h>   ; DT_RELA
LOAD:0000000000228490                 Elf64_Dyn <8, 818h>     ; DT_RELASZ
LOAD:00000000002284A0                 Elf64_Dyn <0>           ; DT_NULL
```

<!-- DYNAMIC 엔트리에서 필요한 정보들을 저장할 오프셋들을 미리 지정해둔 뒤에 해당 테이블을 옮겨오거나 새로 생성할 것이다.  -->
After specifying the offsets to store necessary information in the DYNAMIC entry, the table will be moved or created.

### 3.7. Creating DT_STRTAB

<!-- 일반적으로 ELF에서는 심볼 테이블에서 함수 이름이 위치한 string table의 인덱스를 가지고 있지만 

sprx에서는 함수 이름을 가진 테이블 대신에 함수 고유의 코드인 `nid` 를 가진 table이 존재한다. 

심볼 테이블은 이 nid table의 인덱스를 사용한다.

자세한 내용은 [여기](https://blog.madnation.net/ps4-nid-resolver-ida-plugin/)에 설명되어있다.

전부는 아니지만 이 nid가 각각 무슨 함수들을 가리키는지에 대한 데이터베이스가 존재하므로 

이 데이터베이스를 이용하여 함수 문자열을 얻어낸 다음 함수 스트링 테이블을 만들어주면 될 것이다. 그런데 이 스트링 테이블을 위한 세그먼트의 공간이 충분하지 않다면 직접 세그먼트를 추가해주면 된다. 

그리고 해당 문자열의 크기만큼 `DT_STRSZ` 을 설정해주면 된다. -->

Generally, ELF has the index of string table where the function name is located in the symbol table, but in sprx, instead of the table with the function name, there is a table with `nid`, which is a function specific code. Symbol table uses the index of this nid table.

More information is [here](https://blog.madnation.net/ps4-nid-resolver-ida-plugin/).

Although not all, there is a database of what functions each nid refers to, so we can use this database to get a function string and then create a function string table. However, if there is not enough space for the segment for this string table, add the segment yourself. And, you can set `DT_STRSZ` as much as the size of the string.

#### 3.7.1. Creating DT_SYM

- In sprx

```python
SCE_DYNLIBDATA:0000000001029250                 Symbol <240h, 12h, 0, 3, 7270h, 8> ; _ZN3sce3Xml10SimpleDataC1EPKcm | Global : Function
SCE_DYNLIBDATA:0000000001029268                 Symbol <250h, 12h, 0, 3, 7260h, 9> ; _ZN3sce3Xml10SimpleDataC1Ev | Global : Function
SCE_DYNLIBDATA:0000000001029280                 Symbol <260h, 12h, 0, 3, 7270h, 8> ; _ZN3sce3Xml10SimpleDataC2EPKcm | Global : Function
SCE_DYNLIBDATA:0000000001029298                 Symbol <270h, 12h, 0, 3, 7260h, 9> ; _ZN3sce3Xml10SimpleDataC2Ev | Global : Function
SCE_DYNLIBDATA:00000000010292B0                 Symbol <280h, 12h, 0, 3, 4830h, 12Ch> ; _ZN3sce3Xml11Initializer10initializeEPKNS0_13InitParameterE | Global : Function
SCE_DYNLIBDATA:00000000010292C8                 Symbol <290h, 12h, 0, 3, 47B0h, 72h> ; _ZN3sce3Xml11Initializer9terminateEv | Global : Function
SCE_DYNLIBDATA:00000000010292E0                 Symbol <2A0h, 12h, 0, 3, 4730h, 8> ; _ZN3sce3Xml11InitializerC1Ev | Global : Function
```

<!-- sym table의 첫번째 필드값은 nid table에서 각각 함수들이 대응하는 nid의 오프셋을 가지고 있다.

4번째 필드값은 해당 함수가 위치한 섹션의 인덱스값이다. 나중에 .text섹션을 생성한뒤에 

.text섹션이 몇번째에 위치해있는지를 적어주면 된다. 

대략 어떻게 바뀌는지를 보여주면 다음과 같다. -->

The first field value of the sym table has an offset of the nid corresponding to each function in the nid table.
And the fourth field value is the index value of the section where the function is located. After creating `.text` section later, let's specify where this section is located.

Here's how it changes roughly.

```python
LOAD:0000000000024820                 Elf64_Sym <offset aZn3sce3xml10si - offset unk_2000, 12h, 0, 3, \ ; "_ZN3sce3Xml10SimpleDataC1EPKcm" ...
LOAD:0000000000024820                            offset _ZN3sce3Xml10SimpleDataC2EPKcm, 8>
LOAD:0000000000024838                 Elf64_Sym <offset aZn3sce3xml10si_0 - offset unk_2000, 12h, 0, 3, \ ; "_ZN3sce3Xml10SimpleDataC1Ev" ...
LOAD:0000000000024838                            offset _ZN3sce3Xml10SimpleDataC2Ev, 9>
LOAD:0000000000024850                 Elf64_Sym <offset aZn3sce3xml10si_1 - offset unk_2000, 12h, 0, 3, \ ; "_ZN3sce3Xml10SimpleDataC2EPKcm" ...
LOAD:0000000000024850                            offset _ZN3sce3Xml10SimpleDataC2EPKcm, 8>
LOAD:0000000000024868                 Elf64_Sym <offset aZn3sce3xml10si_2 - offset unk_2000, 12h, 0, 3, \ ; "_ZN3sce3Xml10SimpleDataC2Ev" ...
LOAD:0000000000024868                            offset _ZN3sce3Xml10SimpleDataC2Ev, 9>
LOAD:0000000000024880                 Elf64_Sym <offset aZn3sce3xml11in - offset unk_2000, 12h, 0, 3, \ ; "_ZN3sce3Xml11Initializer10initializeEPK"... ...
LOAD:0000000000024880                            offset _ZN3sce3Xml11Initializer10initializeEPKNS0_13InitParameterE,\
LOAD:0000000000024880                            12Ch>
LOAD:0000000000024898                 Elf64_Sym <offset aZn3sce3xml11in_0 - offset unk_2000, 12h, 0, 3, \ ; "_ZN3sce3Xml11Initializer9terminateEv" ...
LOAD:0000000000024898                            offset _ZN3sce3Xml11Initializer9terminateEv, 72h>
LOAD:00000000000248B0                 Elf64_Sym <offset aZn3sce3xml11in_1 - offset unk_2000, 12h, 0, 3, \ ; "_ZN3sce3Xml11InitializerC1Ev" ...
LOAD:00000000000248B0                            offset _ZN3sce3Xml11InitializerC2Ev, 8>
LOAD:00000000000248C8                 Elf64_Sym <offset aZn3sce3xml11in_2 - offset unk_2000, 12h, 0, 3, \ ; "_ZN3sce3Xml11InitializerC2Ev" ...
LOAD:00000000000248C8                            offset _ZN3sce3Xml11InitializerC2Ev, 8>
```

### 3.8. Creating relocation table

<!-- relocation table은 그대로 copy해오면 되는데, 몇가지 유의할 점이 있다.

- 세그먼트를 새로 추가해주었으므로  `offset` 값과 `addend` 값에 새로 추가해준 세그먼트 값을 더해줘야 할 뿐만 아니라, 해당 offset에 위치한 포인터에 대해서도 값을 더해줘야 한다.
- 함수 포인터의 경우 sprx에서는 심볼 테이블과 같은 형태 비슷한 형태로 info를 나타내고 있는데 이 경우 다른 relocation 값들과 같이 구성해주면 된다. -->

You can copy the relocation table as it is, but there are a few things to note.

- Since the segment has been newly added, not only the newly added segment value must be added to the `offset` and `addend` value, but also the value for the pointer located at the corresponding offset must be added.
- In the case of function pointer, sprx displays info in a form similar to the symbol table. In this case, it can be configured with other relocation values.

```python
SCE_DYNLIBDATA:000000000102AAB0                 Relocation <offset sce::Xml::MemAllocator::~MemAllocator(), \ ; R_X86_64_64
SCE_DYNLIBDATA:000000000102AAB0                             2A00000001h, 0> 

->

LOAD:0000000000026170                 Elf64_Rela <28068h, 8, 1C357h> ; R_X86_64_RELATIVE +1C357h
```

### 3.9. Creating Section Header

<!-- 섹션 헤더를 만드는 부분은 그냥 일반적인 elf 포맷에 대한 이해도만 있으면 된다.

원래 sprx 포맷에서는 섹션 헤더를 사용하지 않는데, 리눅스에서 dlopen으로 메모리에 로딩되기 위해서는 몇몇 필수적인 섹션들이 있다. 이는 몇가지 테스트를 통해 알아낸 내용이다.

섹션에 정보를 적어주는건 간단하다. 

섹션 이름을 표기해주기 위하여 여유있는 공간에( 그리 큰 공간이 필요하지도 않다.) 섹션 이름을 저장하고 저장한 곳의 정보를 `.shstrndx` 섹션 헤더에 담아주면 된다. 그리고 `.shstrndx` 가 몇번째에 위치해있는지 elf header의 `Section header string table index` 에 그 정보를 적어주면 된다.

단 여기서 `.dynstr` 이 `.shstrndx` 보다 먼저 와야한다.

`.dynstr` 은 동적 심볼을 위한 string들이 저장되어있는 섹션이다. 즉 함수 또는 전역 변수의 이름, 때에 따라서는 라이브러리의 이름등이 적혀있기도 하다. 필드값에 위에서 생성했었던 string table 주소, 오프셋, 사이즈등을 적어주면 된다.

`.dynamic` 은 dynamic entries(위에서 생성했던) 의 정보를 적어주면 된다.

필드값에  entry size, 주소, 오프셋, 사이즈등을 적어주면 된다.

`.dynsym` 또한 똑같다. 필드값에 이전에 생성했었던 symbol table의 entry size, 주소, 오프셋, 사이즈등을 적어주면 된다.

sprx에서 코드 영역은 항상 첫번째 세그먼트( 헤더가 로딩되지 않으므로 항상 첫번째 세그먼트에 코드가 위치한다고 생각해도 된다.)에 있으므로 .text 섹션에는 해당 세그먼트의 정보들을 옮겨주면 될 것이다.(offset, address, 권한, type등등) -->

The part of creating the section header just needs to understand the general elf format.

The original sprx format does not use section headers, but there are some essential sections required to be loaded into memory with dlopen in Linux. This is what we found out through several tests.

It's easy to write down the information in the section.

To mark the section name, you can save section name in a free space(not much space is required), and put the saved information in the `.shstrndx` section heade. And you can write the position of `.shstrndx` in `Section header string table index` of elf header.

Howerver, here `.dynstr` must come before `.shstrndx`.

`.dynstr` is a section where strings for dynamic symbols are stored. That is, the name of the function or global variable, and sometimes the name of the library are written there. In the field value, write the string table address, offset, size, etc. create above.

In `.dynamic`, you can write the information of dynamic entries(created above). Enter the entry size, address, offset, size, etc. in the field value.

The same is true for `.dynsym`. In the field value, write the entry size, address, offset, size, etc. of the previously created symbol table.

In sprx, the code section is always in the first segment(the header is not loaded, so you can think that the code is always in the first segment), so you just need to move the information of that segment to the `.text` section. (offset, address, authority, type, etc.)

```python
[Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .dynstr           STRTAB           0000000000002000  00002000
       0000000000001ff0  0000000000000000   A       0     0     1
  [ 2] .shstrndx         STRTAB           0000000000000000  00028600
       0000000000000030  0000000000000000           0     0     1
  [ 3] .dynamic          DYNAMIC          0000000000228400  00028400
       0000000000000170  0000000000000010  WA       1     0     8
  [ 4] .dynsym           DYNSYM           0000000000024580  00024580
       0000000000001368  0000000000000018   A       1     2     8
  [ 5] .text             PROGBITS         0000000000004000  00004000
       000000000001ebf0  0000000000000000  AX       0     0     16
```
### 3.10. Testing the library converted to so file
```c
#include <stdio.h>
#include <dlfcn.h>

int main(){

    long long *handle;
    double *(*func)();

    handle = dlopen("./hello_output2", RTLD_LAZY);
    func = *handle+0x98c0;

    func();
    return 0;
}
```
<!-- 위 소스코드를 사용하여 함수가 잘 실행되는지 테스트해볼 것이다. -->
Using the above source code, we tested whether the function works well.

![image](https://user-images.githubusercontent.com/39231485/101734324-70e61680-3b03-11eb-8315-cca6132f0dfe.png)

<!-- dlsym이 작동하지 않아서 이 함수의 오프셋을 넣고 함수 포인터를 호출시켰다. -->
Since dlsym didn't work, we put the offset of this function and called the function pointer.

```
 ► 0x7ffff7b978c0 <sce::Xml::Dom::NodeList::clear()>       mov    rdi, qword ptr [rdi]
   0x7ffff7b978c3 <sce::Xml::Dom::NodeList::clear()+3>     test   rdi, rdi
   0x7ffff7b978c6 <sce::Xml::Dom::NodeList::clear()+6>     je     sce::Xml::Dom::NodeList::clear()+13 <sce::Xml::Dom::NodeList::clear()+13>
    ↓
   0x7ffff7b978cd <sce::Xml::Dom::NodeList::clear()+13>    ret
```
<!-- 해당 함수가 잘 호출 된 것을 확인할 수 있다. 그러므로 만약 퍼징을 돌린다고 하였을 때, 특정 함수에 여러 값들을 넣어보며 테스트 하는 것이 가능하다. -->

As a result, it was confired that the function was called well. Therefore, if fuzzing, it is possible to test by putting various values in specific function.

## 4. Script
We made a script that convert sprx file to so file. Even though it makes imperfect elf file, it enables the target file to be loaded on memory using dlopen. After loading the library, we can execute a function by adding offset to base address of the library. Even though there are still many limitations, it will reduce your effort to convert sprx file to so file.

You can check the script [here](https://github.com/Hacker-s-PlayStation/PlayStation4-Hacking-Guideline-ENG/tree/main/2_methodology/sprx_to_so).
> Usage : download two script files and execute script.py.<br>
> Command : python3 script.py (sprx library)
<br>The reference for writing the script is as follows.<br>
* env file decrypt : https://github.com/SocraticBliss/ps4_env_decryptor
* ps4 elf loader parsing : https://github.com/SocraticBliss/ps4_module_loader

### 4.1. Limit
#### 4.1.1 Imperfect Connection of plt and got

<!--
plt와 got가 연결되어있지 않기 때문에, 다른 라이브러리에서 import 하여 사용하는 함수는 실행시킬 수 없다.<br>
만약 퍼징을 돌리려는 함수 안에 외부 라이브러리의 함수가 사용된다면 자체적으로 연결시켜줘야한다.<br><br>
우리가 퍼징을 시도할 xml 라이브러리에서는 libc 함수를 import 시키기 때문에 해당 함수들을 프로그랜 내의 plt 주소로 점프 하는 방식을 사용하여 연결시켜주었다.<br>
우리가 적용한 방법은 아래와 같다.<br>
1. xml 라이브러리에서 사용되는 libc 함수(ex. free, malloc)를 퍼징할 프로그램에도 호출한다. 이를 통해 프로그램 내에 해당 함수들의 plt, got가 생기게 된다.
2. dlopen을 한 이후, 프로그램 내부 주소를 라이브러리에 저장한다.
-->

Since plt and got are not connected, the functions that imported from another libraries are not executable. If our target function uses external functions, we have to connect it manually. Because XML library, our target library, uses some libc functions, we called the libc functions that need in XML library to make plt of the functions in the main program binary, and connected the plt in main program binary with functions in XML library manually. We describes how we connected them below.

1. Call the libc functions that need in XML library to make plt of the functions in the main program binary.
2. After open the target library using dlopen, store the address of main program in the target library. We stored address of _start function and this will be used when we find plt in the main binary. We didn't write the address of plt functions directly because the address may change(using PIE or something like that).
```c
    handle = dlopen("./xml_library", RTLD_LAZY);

    asm("pushq %rax;"
    "movq (%rax), %rax;"        // rax has the base address of XML library.
    "movq %r12, 0x28900(%rax);" // We found that r12 register has _start plt address by debugging. Of course this may be different depending on the case.
                                // We write the _start address in the XML library. 0x28900 is just an arbitrary offset that point to one of the pending space.
    "popq %rax;");
```

<!--
3. 저장된 프로그램 내부 주소를 사용하여 프로그램 내의 plt 주소로 jmp 해주는 코드를 xml 라이브러리의 text 섹션에 적는다.<br>아래는 위에서 설명한 코드를 xml 라이브러리에서 사용하는 libc 함수 모두에게 만들어주는 코드이다.
-->

3. We manually modified the XML library binary to insert routines that jump to the plt in main function using _start address value. Below is simple script that makes the routine binary.

```python
# mov rax, qword ptr [rip + offset]  [rip + offset] has address of _start.
# sub rax, 0xe0
# jmp rax

plt_offset_list = []
function_list = []
mov = "\x48\x8b\x05"
mov_offset = 0x5D30-7 # minus 7 because $rip is containing next command's address
small_sub = "\x48\x83\xE8" # 1byte
big_sub = "\x48\x2d" # 4byte
jmp = "\xFF\xE0"

code_offset_list = []
code_offset = 0

with open("input.txt", "r") as f:
    for line in f.readlines():
        plt_offset = line.split()[0]
        function_list.append(line.split()[1])
        plt_offset_list.append(int(plt_offset, 16))

with open("result.bin", "w") as f:
    for plt_offset in plt_offset_list:
        if plt_offset == 0:
            code_offset_list.append(-1)
            continue

        f.write(mov)
        temp = mov_offset
        for _ in range(4):
            f.write(chr(temp%0x100))
            temp //= 0x100

        code_offset_list.append(code_offset)
        if plt_offset < 128:
            code_offset += 13
            mov_offset -= 13
            f.write(small_sub)
            f.write(chr(plt_offset))
        else:
            code_offset += 15
            mov_offset -= 15
            f.write(big_sub)
            for _ in range(4):
                f.write(chr(plt_offset%0x100))
                plt_offset //= 0x100
        f.write(jmp)

for i in range(len(function_list)):
    print(function_list[i] + " " + str(hex(code_offset_list[i])))
```

<!--
4. 라이브러리 내에 있는 고장난 plt에 text 섹션에 적은 각각 함수에 맞는 코드들의 주소를 넣어준다.
-->

4. Modify the broken plt section in the XML library to connect them to our routine described step 3.

```c
    int arr_code[] = {0,0xf,0x1e,0x2d,0x3a,-1,0x49,0x58,0x67,0x76,0x85,0x94,0xa3,0xb0,0xbf,0xcc,-1,0xdb,0xea,0xf9,0x108,0x115,0x124,0x133,0x142}; // Offset of connecting routines that we inserted.
    long long target;
    long long *target_save;

    for(int i = 0;i<23;i++){
        if(arr_code[i] == -1){
            continue;
        }
        target = (long long)*handle + (long long)arr_code[i] + 0x22bd0; // the location of connecting routines in the library.
        target_save = (long long *)((long long)*handle + (long long)0x24368 + (8*i)); // the location of broken plt in XML library. 
        *target_save = target;
    }
```

Below is sample code.

```c
#include <stdio.h>
#include <dlfcn.h>
#include <string.h>
#include <stdlib.h>
#include <iostream>

int main(int argc, char *argv[]){

    char text[30];
    const char test[30] = "AAAAAAA";

    if(argv[0]==0){
        int *a;
        int* c = new int;
        delete c;
        strncpy(text,text,10);
        memmove(test+15,test+10,11);
        a = strnlen(text,30);
        a = strlen(text);
        a = malloc(30);
        free(a);
        a = memcpy(text,test,10);
        a = memset(text,test,10);
        a = strcmp(text,test);
        a = strncmp(text,test,10);
        a = memcmp(text,test,10);
        a = fclose(0);
        FILE* b;
        b  = fopen(test, test);
        a = fread(0, 0,0,0);
        a = fseek(0,0,0);
        a = ftell(0);
        rewind(0);
        a = snprintf(0,0,0);
    }

    int arr_code[] = {0,0xf,0x1e,0x2d,0x3a,-1,0x49,0x58,0x67,0x76,0x85,0x94,0xa3,0xb0,0xbf,0xcc,-1,0xdb,0xea,0xf9,0x108,0x115,0x124,0x133,0x142};
    long long target;
    long long *target_save;

    long long *handle;
    double *(*func)();

    handle = dlopen("./xml_library", RTLD_LAZY);

    asm("pushq %rax;"
    "movq (%rax), %rax;"
    "movq %r12, 0x28900(%rax);"
    "popq %rax;");

    for(int i = 0;i<23;i++){
        if(arr_code[i] == -1){
            continue;
        }
        target = (long long)*handle + (long long)arr_code[i] + 0x22bd0;
        target_save = (long long *)((long long)*handle + (long long)0x24368 + (8*i));
        *target_save = target;
    }

    func = (long long)*handle+0x4118; // execute target function

    func();
    return 0;
}
```

#### 4.2.2 Not Working dlsym
Maybe there are some problems at the symbol table, but we failed to figure out. Thus, we called functions using function pointer.

## 5. Fuzzing (Not completed)

<!--
우리가 구현한 함수를 실행하는 기능을 사용하면 특정 기능에 대한 루틴을 재현하여 해당 부분을 퍼징해볼 수 있다.<br>
하지만, 제약사항이 존재한다.<br><br>
심볼이 완벽하게 복구된게 아니라서 인스턴스 생성이 일반적인 방법으로는 불가능하다. <br>
그래서 조금 분석해본 결과, 클래스의 크기를 맞춰 동적할당을 한 후, 반환된 주소를 인자로 클래스 생성자 함수에 넣어주면 할 수 있지 않을까 여겨진다.<br>
-->

We can try fuzzing the target routine reproduced on PC. However, we faced some problems in this step. Even though we managed to call function, the symbol problem remained. If the target function needs some objects that exists in target library where symbol is not fully recovered, it is hard to use object in the usual way. For example, if we want to make an object, we cannot use new operator. Instead, we may have to allocate memory and call constructor function explicitly. There may be more problems due to not fully recovered symbol. We need to think about this problem.

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
