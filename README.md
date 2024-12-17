이 프로젝트는 auterion의 [Embedded Debug Tools](https://github.com/Auterion/embedded-debug-tools/tree/main)을 개조해 만들어졌습니다.
기존 앱과 협업하여 만들어졌던 관계로, DWT 기능을 쓰기 위해서는 본인의 DWT app에 ITM.h 파일을 include한 뒤 EMDBG_LOG_DWT_CATCH 함수를 선언하여 구현해야 합니다.

사용 방법은 아래와 같습니다. 라즈베리파이4에서의 실행 방법을 기술했으며, 다른 환경에서의 사용법은 auterion 공식 문서를 활용해주시면 감사드리겠습니다.

## 의존 파일 설치

먼저 우리가 앞에서 시도한 대로 xpack으로 arm-none-eabi-gcc를 설치해주었다.

```bash
sudo apt-get update
sudo apt-get upgrade

# python virtual env 설치 및 실행
sudo apt-get install python3-venv
python3 -m venv .venv
source .venv/bin/activate

# xpack 설치
sudo apt-get install curl
sudo install -d -o $USER /opt/xpack

curl -L https://github.com/xpack-dev-tools/arm-none-eabi-gcc-xpack/releases/download/v13.2.1-1.1/xpack-arm-none-eabi-gcc-13.2.1-1.1-linux-arm64.tar.gz | \
        tar -xvzf - -C /opt/xpack/
```

xpack 설치 후, 원래는 gcc-multilib이 설치되어 있어서, 그냥 이 xpack의 arm-none-eabi-gdb-py3만을 심볼릭 링크를 걸어주었지만, 우린 라즈베리파이 환경이어서 설치되어 있지 못하므로 아예 이 xpack을 path에 추가해주었다.

```bash
# 영구적 방법
sudo vi ~/.bashrc

# 파일 맨 마지막 줄에 다음 문장 추가
export PATH="$PATH:/opt/xpack/xpack-arm-none-eabi-gcc-13.2.1-1.1/bin"
# esc 후 저장
:wq

# 실행해서 꼭 적용해주어야 함
source ~/.bashrc

# 만약 venv가 비활성화되었다면 다시 활성화
```

## px4_fmu-v6x_default 빌드

이제 xpack을 통해 arm-none-eabi-gcc를 수동으로 설치했으므로, 라즈베리파이에서 px4_fmu-v6x_default 빌드까지 가능하다.

```bash
git clone https://github.com/PX4/PX4-Autopilot.git --no-recursive

# dependency 설치
bash ./PX4_Autopilot/Tools/setup/ubuntu.sh --no-nuttx --no-sim-tools

# make
make px4_fmu-v6x_default
```

이로써 자체적으로 build도 가능하다.

## Jlink 설치

```bash
wget --post-data "accept_license_agreement=accepted" https://www.segger.com/downloads/jlink/JLink_Linux_V796n_arm64.deb
sudo dpkg -i JLink_Linux_V796n_arm64.deb

# 위 명령어를 입력하면 여러 의존 파일들이 없어서 실패했다는 오류 화면이 뜨는데, 아래 명령어를 통해 의존 파일들을 설치해주고
# 다시 dpkg 명령어를 실행하면 제대로 설치됨을 확인할 수 있었다.
sudo apt --fix-broken install
sudo dpkg -i JLink_Linux_V796n_arm64.deb
```

## emdbg 설치

```bash
pip3 install emdbg
# 가상환경에서 이 명령어를 실행해야 한다!
```

위 명령이면 깔끔하게 설치될 것이다. 파이썬 venv에서 실행해야 하고, sudo를 붙이지 않아야함을 명심하자.

## orbetto 설치

대망의 orbetto다.

```bash
sudo apt-get install -y libncurses-dev libusb-1.0-0-dev libzmq3-dev meson libsdl2-dev libdwarf-dev libdw-dev libelf-dev libcapstone-dev python3-pip ninja-build protobuf-compiler
pip3 install meson==1.2.0
```

의존 파일들을 설치했다면 빌드를 진행해주면 된다.

```bash
cd ./embedded-debug-tools/ext/orbetto
meson setup build
ninja -C build
```

- TroubleShooting
    - subprojects/orbuculum/meson.build:12:0: ERROR: Dependency "ncursesw" not found, tried pkgconfig
        
        meson setup build 명령어를 그냥 실행했을 때, 아래의 오류를 맞닥뜨렸다.
        
        <aside>
        💡 subprojects/orbuculum/meson.build:12:0: ERROR: Dependency "ncursesw" not found, tried pkgconfig
        
        </aside>
        
        이 문제를 해결하기 위해, 아래의 명령어를 입력해주면 된다.
        
        ```bash
        sudo apt install libncurses-dev
        ```
        

## px4_fmu-v6x_default trace on

이제 빌드를 진행했던 PX4-Autopilot 디렉토리로 돌아가 emdbg에서 제공하는 패치 기능을 사용한다.

```bash
# cd to PX4-Autopilot/
python3 -m emdbg.patch nuttx_tracing_itm_v11 --apply -v
```

![Untitled%202.png](https://file.notion.so/f/f/49d5a777-35f2-40db-9a89-86307bbfd66f/1805b7a0-a3d8-4904-ad41-73fb30b8e3e8/Untitled.png?table=block&id=db5a00b7-ff54-4453-9d45-b1d2a8f6be66&spaceId=49d5a777-35f2-40db-9a89-86307bbfd66f&expirationTimestamp=1734508800000&signature=3PqZ60m4bkY0DsUdjcVPhNrkd-nZEtu0KB3tp-K8lvY&downloadName=Untitled.png)

![Untitled](https://file.notion.so/f/f/49d5a777-35f2-40db-9a89-86307bbfd66f/6d73844d-e488-4d76-aaac-9031936a688d/Untitled.png?table=block&id=bb9da039-5fa1-414a-802c-30cc27acd402&spaceId=49d5a777-35f2-40db-9a89-86307bbfd66f&expirationTimestamp=1734508800000&signature=wzZ49arNkk1kiJ8X_0HqLYScHmHFPHGiUloYoIZcGMo&downloadName=Untitled.png)

적용이 되었다면 이와 같이 한 줄만 딸랑 나온다. 확인 되었다면 make 후 upload를 진행한다.

```bash
make px4_fmu-v6x_default
sudo make px4_fmu-v6x_default upload
```

- TroubleShooting
    - make 후 region ‘FLASH’ overflowed 문제 발생 시
        
        ![Untitled](https://file.notion.so/f/f/49d5a777-35f2-40db-9a89-86307bbfd66f/65c576cd-752b-44c6-8dd9-f29d77e57853/Untitled.png?table=block&id=5b8d9efb-8d63-4e86-9744-809b3cdebd73&spaceId=49d5a777-35f2-40db-9a89-86307bbfd66f&expirationTimestamp=1734508800000&signature=p4B0AKGwyON0ylp44jPBC0_2e-2U2HLvpDrEjs4LxKQ&downloadName=Untitled.png)
        
        이런 식으로 FLASH에 너무 많은 프로그램이 들어가 flash overflow 문제가 발생할 수 있다. 이를 해결하기 위해선 우리 테스트에 쓸모없는 모듈을 비활성화해주어야 한다. 우린 추후 GPS와 모터를 instrument할 계획이므로 일단 제일 필요없어 보이는 HEATER의 드라이버 모듈을 비활성화해주었다.
        
        비활성화 방법은 다음과 같다. 
        
        ```bash
        # cd PX4-Autopilot
        vi boards/px4/fmu-v6x/default.px4board
        ```
        
        이 파일에서 모듈을 키고 끌 수 있다. 여러 모듈 중 원하는 모듈을 꺼도 되고, 원한다면 내가 끈 CONFIG_DRIVERS_HEATER=n으로 설정해주어도 된다. 단, 그 모듈이 overflow를 해결할 정도의 크기인지 알 수 없으므로 이왕이면 앞에서 끈 모듈을 끄는 것을 권한다.
        
        그 뒤 다시 make를 진행하면 
        
        ![Untitled](https://file.notion.so/f/f/49d5a777-35f2-40db-9a89-86307bbfd66f/f146b114-624c-4dfa-99bf-4f51bf0abc2b/Untitled.png?table=block&id=e7717f09-bd1f-4143-8cb9-7ac4df870383&spaceId=49d5a777-35f2-40db-9a89-86307bbfd66f&expirationTimestamp=1734508800000&signature=hc8o59HOOB6KGHV0cLqILJF-soo_WW-9weKdnhwMWeM&downloadName=Untitled.png)
        
        문제없이 진행되었음을 확인할 수 있다.
        
    - Waiting for bootloader… 후 진행 안 됨
        
        이 과정을 진행할 때 bootloader를 기다린다는 말만 뜨고 굳어버릴 때가 있는데, 이럴 땐 기다리던가, 아니면 pc와 연결된 선을 뺐다가 다시 꽂으면 빠르게 진행됨을 확인할 수 있었다.
        
    

업로드까지 마쳤다면 모든 준비가 끝났다.

## emdbg.bench.fmu 실행으로 trace.swo 획득

이제 똑같이 PX4-Autopilot 디렉토리에서 아래 명령어를 실행해 gdb를 실행한다.

```bash
python3 -m emdbg.bench.fmu --target px4_fmu-v6x_default --jlink
```

![Untitled](https://file.notion.so/f/f/49d5a777-35f2-40db-9a89-86307bbfd66f/3308c1e4-777b-42a7-811b-a6b4c3f8a64f/Untitled.png?table=block&id=72cc7199-7eaf-4c10-958a-eda465a92bc4&spaceId=49d5a777-35f2-40db-9a89-86307bbfd66f&expirationTimestamp=1734508800000&signature=A76THOMN-tQtZgcrO67u2S4xwa7shNVoHXnZ27YIwcE&downloadName=Untitled.png)

제대로 실행이 됐다면 위와 같은 화면이 뜨고 (gdb)로 우리의 입력을 기다릴 것이다. 이제 아래 명령들을 통해 trace를 시작한다.

```bash
(gdb) px4_trace_swo_stm32h7
(gdb) continue
# 계측할만큼 했다고 생각되면, ctrl-c 입력 후 종료!
^C
(gdb) quit
```

![Untitled](https://file.notion.so/f/f/49d5a777-35f2-40db-9a89-86307bbfd66f/f6f9ed56-b462-4f3a-8fe8-f428fcc7a21b/Untitled.png?table=block&id=b9510b7c-1e91-4552-a95b-6ff7c3d500eb&spaceId=49d5a777-35f2-40db-9a89-86307bbfd66f&expirationTimestamp=1734508800000&signature=rFFuAGKW4Af1FzSz1YhTpMK22vKIGaybdUMXMZUMUr0&downloadName=Untitled.png)

px4_trace_swo_stm32h7을 입력하면 resets core, 주변 기기를 진행하고, 시작 부분의 breakpoint를 걸어 시작 전 준비를 마친다. 이 상태에서 continue를 누르면 그때부터 ctrl-c를 입력할 때까지 trace를 해준다. 이 과정까지 진행했다면 quit을 입력해 gdb에서 나간다.

![Untitled](https://file.notion.so/f/f/49d5a777-35f2-40db-9a89-86307bbfd66f/4015c6e0-b37e-48ba-aa9a-ff53cafc6ea4/Untitled.png?table=block&id=51e76e15-a75b-4c20-8c94-a3aac60dc2d0&spaceId=49d5a777-35f2-40db-9a89-86307bbfd66f&expirationTimestamp=1734508800000&signature=pVxzgG5SYr1HiqktGSroA1rsfkyv-L_hfGIyE6f-Flo&downloadName=Untitled.png)

그렇게 나가고 ls를 통해 살펴보면, 빨간색 상자에 보이는 것처럼 trace.swo를 얻을 수 있다.

## trace.swo를 perfetto.perf로 변환

이 파일을 그냥 읽을 순 없고, perfetto로 볼 수 있도록 지원 형식을 맞춰 변형을 가해줘야 한다. 이를 위해 embedded-debug-tools/ext/orbetto로 이동한 뒤 아래의 명령을 입력해준다.

```bash
# cd embedded-debug-tools/ext/orbetto
cd /embedded-debug-tools/ext/orbetto
build/orbetto -C 480000 -f path/to/trace.swo -e path/to/elf
```

이 과정을 마치면 아래 처럼 orbetto.perf가 생겨났음을 확인할 수 있다.

![Untitled](https://file.notion.so/f/f/49d5a777-35f2-40db-9a89-86307bbfd66f/27996ec3-8995-40ce-99be-798b91c89a38/Untitled.png?table=block&id=92da4387-cc6e-43a7-b7ef-13d892ddb365&spaceId=49d5a777-35f2-40db-9a89-86307bbfd66f&expirationTimestamp=1734508800000&signature=5OyoUAqgLn5ZYdfM7YmYWrEgeGy0YbZFzeJtDdumc-w&downloadName=Untitled.png)

이걸 라즈베리파이에서 다른 컴퓨터로 옮겨서 [perfetto](https://ui.perfetto.dev/)에 가져다주면 결과가 제대로 나옴을 확인할 수 있다.

![Untitled](https://file.notion.so/f/f/49d5a777-35f2-40db-9a89-86307bbfd66f/0fd59cbc-fe4d-404b-ac6e-0fc081dadc50/Untitled.png?table=block&id=1bce290b-b0a7-45ab-9610-7e5b6df83525&spaceId=49d5a777-35f2-40db-9a89-86307bbfd66f&expirationTimestamp=1734508800000&signature=Z_99yWj9s-ShOlpjGBHqs6_22vMUYZnZ68xotBbuUbY&downloadName=Untitled.png)

---

# Embedded Debug Tools

The emdbg library connects several software and hardware debugging tools
together in a user friendly Python package to more easily enable advanced use
cases for ARM Cortex-M microcontrollers and related devices.

The library orchestrates the launch and configuration of hardware debug and
trace probes, debuggers, logic analyzers, and waveform generators and provides
analysis tools, converters, and plugins to provide significant insight into the
software and hardware state during or after execution.

The main focus of this project is the debugging of the PX4 Autopilot firmware
running the NuttX RTOS on STM32 microcontrollers on the FMUv5x and FMUv6x
hardware inside the Auterion Skynode. However, the library is modular and the
tools are generic so that it can also be used for other firmware either
out-of-box or with small adaptations.

emdbg is maintained by [@niklaut](https://github.com/niklaut) from
[Auterion](https://auterion.com).

## Features

- Debug Probes: SWD and ITM/DWT over SWO.
    - [SEGGER J-Link](https://www.segger.com/products/debug-probes/j-link/): BASE Compact (3MB/s), EDU Mini.
    - [STMicro STLink-v3 MINIE](https://www.st.com/en/development-tools/stlink-v3minie.html): (2MB/s).
    - [Orbcode ORBTrace mini](https://orbcode.org/orbtrace-mini): SWO (6MB/s).
    - Coredump via [CrashDebug](https://github.com/adamgreen/CrashDebug) with support for PX4 Hardfault logs.
- Trace Probes: ITM/DWT/ETM over TRACE.
    - [Orbcode ORBTrace mini](https://orbcode.org/orbtrace-mini): (50MB/s).
    - [SEGGER J-Trace](https://www.segger.com/products/debug-probes/j-trace/models/j-trace/): (>100MB/s) (planned).
- [GDB Debugger](https://developer.arm.com/Tools%20and%20Software/GNU%20Toolchain).
    - Automatic management of debug probe drivers.
    - Remote interfacing via [GDB/MI](https://github.com/cs01/pygdbmi) and RPyC.
    - Plugins via [GDB Python API](https://sourceware.org/gdb/onlinedocs/gdb/Python-API.html).
    - [User commands for PX4 and NuttX](https://github.com/Auterion/embedded-debug-tools/blob/main/src/emdbg/debug/gdb.md#user-commands).
    - Hardfault trapping with immediate backtrace.
- [Real-time instrumentation using ITM/DWT](https://github.com/Auterion/embedded-debug-tools/blob/main/ext/orbetto).
    - Visualization of entire RTOS state via [perfetto](https://perfetto.dev).
    - Nanosecond resolution with very little runtime overhead.
- Patch Manager for out-of-tree modifications.
- Power Switch.
    - Yocto USB Relay.
- Logic Analyzer and Waveform Generator.
    - [Digilent Analog Discovery 2](https://digilent.com/reference/test-and-measurement/analog-discovery-2/start) via Python API.
    - [Sigrok](https://sigrok.org/wiki/Main_Page) (prototyped).
    - Visualization via [perfetto](https://perfetto.dev) (prototyped).
- Serial Protocols.
    - NuttX NSH command prompt.
- Hardware configuration.
    - [FMUv5x and FMUv6x](https://docs.px4.io/main/en/flight_controller/autopilot_pixhawk_standard.html).

A number of GDB and NSH scripting examples for test automation can be found in
the `scripts` folder.


## Presentations

Sorted in reverse chronological order.

### Analyzing Cortex-M Firmware with the Perfetto Trace Processor

Presented at [emBO++](http://embo.io) by Niklas Hauser on 2024-03-15.

[![](https://i.ytimg.com/vi/FIStxUz2ERY/maxresdefault.jpg)](https://www.youtube.com/watch?v=FIStxUz2ERY&t=108s)

[Slides with Notes](https://salkinium.com/talks/embo24_perfetto.pdf).

### Debugging PX4

Presented at the [PX4 Developer Summit](https://events.linuxfoundation.org/px4-developer-summit) by Niklas Hauser on 2023-10-22.

[![](https://i.ytimg.com/vi/1c4TqEn3MZ0/maxresdefault.jpg)](https://www.youtube.com/watch?v=1c4TqEn3MZ0)

[Slides with Notes](https://salkinium.com/talks/px4summit23_debugging_px4.pdf).

### Debugging and Profiling NuttX and PX4

Presented at the [NuttX International Workshop](https://events.nuttx.apache.org/index.php/nuttx-international-workshop-2023) by Niklas Hauser on 2023-09-29.

[![](https://i3.ytimg.com/vi/_k1f4F2JVBA/maxresdefault.jpg)](https://www.youtube.com/watch?v=_k1f4F2JVBA)

### Debugging Microcontrollers

Presented at [Chaos Communication Camp](https://events.ccc.de/camp/2023/) by Niklas Hauser on 2023-08-18.

[![](https://static.media.ccc.de/media/conferences/camp2023/57321-4a4f8363-865f-52b7-b236-3b9b73aa2ad7_preview.jpg)](https://media.ccc.de/v/camp2023-57321-debugging_microcontrollers)

[Slides with Notes](https://salkinium.com/talks/cccamp23_debugging_microcontrollers.pdf).

## Installation

The latest version is hosted on PyPi and can be installed via pip:

```sh
pip3 install emdbg
```

You also need to install other command line tools depending on what you use:

- Debugger: [`arm-none-eabi-gdb-py3`](https://github.com/Auterion/embedded-debug-tools/blob/main/src/emdbg/debug/gdb.md#installation).
- Debug probes: [J-Link](https://github.com/Auterion/embedded-debug-tools/blob/main/src/emdbg/debug/jlink.md#installation),
                [OpenOCD](https://github.com/Auterion/embedded-debug-tools/blob/main/src/emdbg/debug/openocd.md#installation),
                [CrashDebug](https://github.com/Auterion/embedded-debug-tools/blob/main/src/emdbg/debug/crashdebug.md#installation).
- Analysis: [graphviz](https://github.com/Auterion/embedded-debug-tools/blob/main/src/emdbg/analyze/callgraph.md#installation)


## Usage

Most modules have their own command-line interface. This library therefore has
many entry points which can be called using `python3 -m emdbg.{module}`.
The individual command line usage is documented in each module.
The most important modules are `emdbg.debug.gdb` and `emdbg.bench.fmu`:

For example, launching GDB with TUI using a J-Link debug probe:

```sh
python3 -m emdbg.debug.gdb --elf path/to/firmware.elf --ui=tui jlink -device STM32F765II
```


## Documentation

Most important user guides are available as Markdown files in the repository.
You can browse the API documentation locally using the `pdoc` library:

```sh
pdoc emdbg
# pdoc server ready at http://localhost:8080
```


## Development

For development, checkout the repository locally, then install with the `-e`
flag, which symlinks the relevant files into the package path:

```sh
cd embedded-debug-tools
pip3 install -e ".[all]"
```
