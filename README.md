KTRW
===================================================================================================

KTRW는 iPhone 8과 같은 A11 SoC가 탑재된 기기를 위한 iOS 커널 디버거입니다. 이 문서에서는
이러한 디바이스에 있는 디버그 레지스터를 사용하여 KTRR을 우회하고, 커널을 쓰기 가능으로 리매핑하고,
GDB 스텁을 구현하는 커널 확장을 로드하여 모든 기능을 갖춘 커널 디버깅을 허용하는 방법을 보여줍니다.
표준 Lightning-USB 케이블을 통해 LLDB 또는 IDA Pro를 사용할 수 있습니다.

checkra1n] 탈옥 및 [pongoOS] 사전 부팅 환경이 출시됨에 따라 다음을 수행할 수 있습니다.
커널 확장으로 KTRW를 iOS 커널캐시에 직접 로드할 수 있으므로 원래의 KTRR 바이패스는
더 이상 사용되지 않습니다.


[checkra1n]: https://checkra.in
[pongoOS]: https://github.com/checkra1n/pongoOS


Bypassing KTRR
---------------------------------------------------------------------------------------------------



KTRR은 중요한 커널 데이터(모든 실행 코드 포함)를 잠그는 수단으로 A10과 함께 도입되어 커널 메모리를 가진 공격자도 수정할 수 없도록 하는
실행 코드를 포함하여) 커널 메모리를 가진 공격자라도 수정하지 못하도록 잠그는 수단으로 도입되었습니다.
읽기/쓰기 기능을 가진 공격자도 수정할 수 없습니다. 그러나 A11 SoC에서는 ARMv8 외부 디버그 레지스터와 독점적인
레지스터가 활성화되어 있었습니다. 이를 통해 이러한 디바이스에서 리셋 벡터의 실행을 무력화할 수 있습니다.
벡터의 실행을 무력화하여 MMU의 KTRR 초기화를 건너뛰고 커널을 재매핑하는 사용자 정의 페이지 테이블
커널을 쓰기 가능한 것으로 다시 매핑하는 사용자 정의 페이지 테이블을 설정할 수 있습니다. KTRR이 비활성화되면 다음과 같은 작업이 가능해집니다.
동적으로 로드된 커널 코드 실행, 즉 커널 확장을 로드할 수 있습니다.

커널이 쓰기 가능으로 리매핑되더라도, AMCC
RoRgn에 의해 스팬된 물리적 페이지는 메모리 컨트롤러에 의해 보호되므로 이러한 물리적 페이지에 대한 쓰기는
버려집니다. MMU에서 KTRR을 우회해도 AMCC에서 KTRR을 무효화할 수 없으므로, 유일한 방법은
커널을 쓰기 가능한 것으로 다시 매핑하는 유일한 방법은 AMCC RoRgn의 커널 데이터를 쓰기 가능한 새 물리적 페이지에 복사하는 것입니다.
물리적 페이지에 복사하는 것입니다. 그러나 AMCC는 여전히 원본 물리적 페이지를 보호하고 있고, 리셋 벡터가 물리적 페이지에서 실행되므로
리셋 벡터는 AMCC RoRgn 내부의 물리적 주소에서 실행되므로, 리셋 벡터를
더 강력한 기능 없이 재설정 시 자동으로 KTRR을 비활성화하도록 지속적으로 수정할 수 없습니다.
(예: 부트체인 취약점) 없이는 재설정 시 KTRR을 자동으로 비활성화하도록 수정할 수 없습니다. 따라서, 코어가 정상적으로 재설정되면(즉, 부트체인 취약점에 걸리지 않고)
정상적으로(즉, 다른 코어의 디버그 레지스터를 사용하여 하이재킹되지 않고) 사라집니다. 이는 곧
KTRR 바이패스는 영구적이지 않으며, 디바이스가 절전 모드로 전환되면 사라집니다.


Using KTRW
---------------------------------------------------------------------------------------------------

KTRW는 네 가지 구성 요소로 이루어져 있습니다: `pongo_kext_loader` 유틸리티, `kextload.pongo-module`
pongoOS 모듈, `ktrw_gdb_stub.ikext` 커널 확장, `ktrw_usb_proxy` USB-to-TCP
프록시 유틸리티. 이들은 각 하위 디렉터리에서 `make`를 실행하여 개별적으로 빌드하거나
최상위 디렉터리에서 `make`를 실행하여 일괄적으로 빌드할 수 있습니다.


KTRW를 사용하려면 세 가지 유틸리티를 실행합니다: [checkra1n], `pongo_kext_loader`, `ktrw_usb_proxy`입니다.

Checkra1n은 checkm8 SecureROM 익스플로잇을 사용하여 pongoOS라는 사전 부팅 환경을 설정합니다.
라는 사전 부팅 환경을 설정하여 임의의 코드 모듈을 로드하고 커널을 패치할 수 있습니다. 다음 명령을 실행하면
을 실행하면 checkra1n이 DFU 모드에서 연결된 iOS 디바이스를 수신 대기하고 pongoOS를 부팅합니다:

	$ /Applications/checkra1n.app/Contents/MacOS/checkra1n -c -p

pongoOS 자체로는 장치의 커널캐시에 XNU 커널 확장을 삽입하는 기능을 제공하지 않습니다.
에 삽입하는 기능을 제공하지 않습니다. 이를 위해 `pongo_kext_loader`와 `kextload.pongo-module`을 사용합니다. 이 경우
kextload` pongoOS 모듈은 pongoOS 셸에 두 개의 새로운 명령어인 `kernelcache-symbols`를 추가합니다.
커널 심볼을 확인하기 위해 심볼 테이블을 업로드할 수 있는 `커널 캐시 심볼`과 업로드된 커널 확장자를
업로드된 커널 확장을 커널캐시에 삽입합니다. pongo_kext_loader` 유틸리티는 모든 것을 하나로 묶어줍니다.
연결된 pongoOS 디바이스를 수신 대기하고, `kextload.pongo-module`을 로드하고, 커널 캐시에 업로드된
ktrw_gdb_stub.ikext` 커널 확장자를 iOS 커널캐시에 삽입하고 XNU를 부팅합니다.

다음 명령을 실행하여 `pongo_kext_loader`가 iOS 기기에서 `ktrw_gdb_stub.ikext`를 로드하도록 합니다.
를 로드합니다:


	$ pongo_kext_loader/pongo_kext_loader \
		pongo_kextload/kextload.pongo-module \
		ktrw_gdb_stub/kernel_symbols \
		ktrw_gdb_stub/ktrw_gdb_stub.ikext

마지막으로 실행해야 하는 유틸리티는 `ktrw_usb_proxy`입니다. ktrw_usb_proxy`는 USB를 통해 커널 확장 프로그램과 통신하기 위해 필요합니다.
커널 확장 프로그램과 통신하고 LLDB가 연결할 수 있도록 TCP를 통해 데이터를 중계하는 데 필요합니다. It
는 연결을 통해 교환되는 데이터를 stdout에 출력합니다. LLDB가 연결할 포트 번호와 함께 `ktrw_usb_proxy`를 실행합니다.
포트 번호로 `ktrw_usb_프록시`를 실행합니다:

	$ ktrw_usb_proxy/ktrw_usb_proxy 39399

Finally, connect an iOS device using a USB cable and enter DFU mode. First checkra1n will boot
pongoOS, then `pongo_kext_loader` will insert the KTRW GDB stub kernel extension, and finally XNU
will boot. The GDB stub will start running about 30 seconds after the kernel starts booting to give
the system time to initialize. (This delay can be configured during build using the
`ACTIVATION_DELAY` make variable.)

Once the GDB stub runs, it will claim one CPU core for itself and halt the remaining cores. It will
also hijack the Synopsys USB 2.0 OTG controller from the kernel so that it can communicate with the
host. As a result, the host will not see the iPhone as an iOS device and the phone (once it has
been resumed) will not be able to send data over USB as normal.

At this point, you are ready to debug the device.

A few common issues:

* If you're having trouble entering DFU mode using a USB C cable, try using a USB A cable instead.
* If the device immediately panics when the GDB stub starts to run, the issue may be that the USB
  hardware is not yet powered. This is likely to be the case if the device has a passcode and the
  "USB Accessories" setting is disabled to prevent USB accessories from connecting when the device
  is locked. Either disabling the passcode, allowing USB accessories, or unlocking the device
  before the GDB stub starts should solve the issue. Try changing `ACTIVATION_DELAY` if you need
  more time to unlock the device.
* Similarly, do not unplug the device while KTRW is running, as this powers down the USB hardware.
* If LLDB does not automatically detect that the target is an iOS kernelcache, it's possible that
  the system halted while a CPU core was in userspace. Try continuing and re-interrupting the
  target until all CPU cores are running in kernel mode, then reattach to the target.
* Sometimes the screen becomes unresponsive right after connecting LLDB. I have been unable to
  identify/fix the root cause of this issue, but it seems to help if you connect with LLDB as soon
  as the GDB stub first halts the device, then immediately continue to minimize the amount of time
  the system is stopped during this initial halt.
* KTRW is incompatible with on-device debugging, e.g. using Xcode or debugserver to debug a process
  on the device.
* The pongoOS kernel extension loading module currently has several limitations: it will only work
  on new-style kernelcaches and only one kernel extension can be inserted into the kernelcache.

The method described here has been tested as working on iOS 13.3. Prior versions of KTRW used a
KTRR bypass to load kernel extensions rather than relying on checkra1n; `ktrw_kext_loader`
implements this technique, and it should be used instead when debugging iOS 12.


Debugging with LLDB
---------------------------------------------------------------------------------------------------

Use LLDB to connect to `ktrw_usb_proxy` and communicate with `ktrw_gdb_stub.ikext`. Here I have
connected to an iPhone 8 running iOS 12.1.2:

	$ lldb kernelcache.iPhone10,1.16C101
	(lldb) target create "kernelcache.iPhone10,1.16C101"
	Current executable set to 'kernelcache.iPhone10,1.16C101' (arm64).
	(lldb) settings set plugin.dynamic-loader.darwin-kernel.load-kexts false
	(lldb) gdb-remote 39399
	Kernel UUID: 94463A80-7B38-3176-8872-0B8E344C7138
	Load Address: 0xfffffff027e04000
	Kernel slid 0x20e00000 in memory.
	Loaded kernel file kernelcache.iPhone10,1.16C101
	Process 2 stopped
	Target 0: (kernelcache.iPhone10,1.16C101) stopped.
	(lldb)

You can use `thread list` to list the code running on each physical CPU core. (Note that one core
is reserved for the debugger itself, so it will not show up in the list.)

	(lldb) th l
	Process 2 stopped
	* thread #1: tid = 0x0002, 0xfffffff027ffda18 kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol1734$$kernelcache.iPhone10,1.16C101 + 272
	  thread #2: tid = 0x0003, 0xfffffff027ffda18 kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol1734$$kernelcache.iPhone10,1.16C101 + 272
	  thread #3: tid = 0x0004, 0xfffffff027ffda18 kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol1734$$kernelcache.iPhone10,1.16C101 + 272
	  thread #4: tid = 0x0005, 0xfffffff027ffda18 kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol1734$$kernelcache.iPhone10,1.16C101 + 272
	  thread #5: tid = 0x0006, 0xfffffff027ffda18 kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol1734$$kernelcache.iPhone10,1.16C101 + 272
	(lldb)

Because KTRR has been disabled, it is possible to patch kernel memory:

	(lldb) x/12wx 0xfffffff027e04000
	0xfffffff027e04000: 0xfeedfacf 0x0100000c 0x00000000 0x00000002
	0xfffffff027e04010: 0x00000016 0x00001068 0x00200001 0x00000000
	0xfffffff027e04020: 0x00000019 0x00000188 0x45545f5f 0x00005458
	(lldb) mem wr -s 4 0xfffffff027e04000 0x11223344 0x55667788
	(lldb) x/12wx 0xfffffff027e04000
	0xfffffff027e04000: 0x11223344 0x55667788 0x00000000 0x00000002
	0xfffffff027e04010: 0x00000016 0x00001068 0x00200001 0x00000000
	0xfffffff027e04020: 0x00000019 0x00000188 0x45545f5f 0x00005458

Resume executing the kernel with `continue`. You can interrupt it at any time with `^C`:

	(lldb) c
	Process 2 resuming
	(lldb) ^C
	Process 2 stopped
	Target 0: (kernelcache.iPhone10,1.16C101) stopped.
	(lldb)

You can set breakpoints as usual. KTRW currently only supports hardware breakpoints, but LLDB will
automatically detect this and set the appropriate breakpoint type:

	(lldb) b 0xfffffff0282753b4
	Breakpoint 1: where = kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol4960$$kernelcache.iPhone10,1.16C101, address = 0xfffffff0282753b4
	(lldb) c
	Process 2 resuming
	Process 2 stopped
	* thread #4, stop reason = breakpoint 1.1
	    frame #0: 0xfffffff0282753b4 kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol4960$$kernelcache.iPhone10,1.16C101
	kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol4960$$kernelcache.iPhone10,1.16C101:
	->  0xfffffff0282753b4 <+0>:  sub    sp, sp, #0x80             ; =0x80
	    0xfffffff0282753b8 <+4>:  stp    x28, x27, [sp, #0x20]
	    0xfffffff0282753bc <+8>:  stp    x26, x25, [sp, #0x30]
	    0xfffffff0282753c0 <+12>: stp    x24, x23, [sp, #0x40]
	Target 0: (kernelcache.iPhone10,1.16C101) stopped.
	(lldb)

Single-stepping works as expected:

	(lldb) si
	Process 2 stopped
	* thread #4, stop reason = instruction step into
	    frame #0: 0xfffffff0282753b8 kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol4960$$kernelcache.iPhone10,1.16C101 + 4
	kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol4960$$kernelcache.iPhone10,1.16C101:
	->  0xfffffff0282753b8 <+4>:  stp    x28, x27, [sp, #0x20]
	    0xfffffff0282753bc <+8>:  stp    x26, x25, [sp, #0x30]
	    0xfffffff0282753c0 <+12>: stp    x24, x23, [sp, #0x40]
	    0xfffffff0282753c4 <+16>: stp    x22, x21, [sp, #0x50]
	Target 0: (kernelcache.iPhone10,1.16C101) stopped.
	(lldb) si
	Process 2 stopped
	* thread #4, stop reason = instruction step into
	    frame #0: 0xfffffff0282753bc kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol4960$$kernelcache.iPhone10,1.16C101 + 8
	kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol4960$$kernelcache.iPhone10,1.16C101:
	->  0xfffffff0282753bc <+8>:  stp    x26, x25, [sp, #0x30]
	    0xfffffff0282753c0 <+12>: stp    x24, x23, [sp, #0x40]
	    0xfffffff0282753c4 <+16>: stp    x22, x21, [sp, #0x50]
	    0xfffffff0282753c8 <+20>: stp    x20, x19, [sp, #0x60]
	Target 0: (kernelcache.iPhone10,1.16C101) stopped.
	(lldb)

Watchpoints are also supported:

	(lldb) wa s e -s 8 -w read_write -- 0xfffffff027e04000
	Watchpoint created: Watchpoint 1: addr = 0xfffffff027e04000 size = 8 state = enabled type = rw
	    new value: 6153737367135073092
	(lldb) reg w x1 0xfffffff027e04000
	(lldb) c
	Process 2 resuming
	
	Watchpoint 1 hit:
	old value: 6153737367135073092
	new value: 6153737367135073092
	Process 2 stopped
	* thread #5, stop reason = watchpoint 1
	    frame #0: 0xfffffff028275418 kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol4960$$kernelcache.iPhone10,1.16C101 + 100
	kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol4960$$kernelcache.iPhone10,1.16C101:
	->  0xfffffff028275418 <+100>: and    x21, x9, x10
	    0xfffffff02827541c <+104>: add    x10, x11, x10
	    0xfffffff028275420 <+108>: add    x8, x10, w8, sxtw
	    0xfffffff028275424 <+112>: and    x26, x9, x8
	Target 0: (kernelcache.iPhone10,1.16C101) stopped.
	(lldb) x/4i $pc-8
	    0xfffffff028275410: 0x9360fd29   asr    x9, x9, #32
	    0xfffffff028275414: 0xa9402eea   ldp    x10, x11, [x23]
	->  0xfffffff028275418: 0x8a0a0135   and    x21, x9, x10
	    0xfffffff02827541c: 0x8b0a016a   add    x10, x11, x10
	(lldb) reg r x23 x10 x11
	     x23 = 0xfffffff027e04000
	     x10 = 0x5566778811223344
	     x11 = 0x0000000200000000
	(lldb)

LLDB limits watchpoints to 1, 2, 4, or 8 bytes in size, even though the hardware supports even
larger watchpoints.


Debugging with IDA Pro
---------------------------------------------------------------------------------------------------

It is possible to use IDA Pro 7.3 or later for iOS kernel debugging, but there are some notable
limitations prior to IDA Pro 7.5.

On IDA Pro 7.3 and 7.4, you will need to modify `dbg_xnu.cfg` to add support for the KTRW GDB
stub's `target.xml` features. You can find the required changes in `misc/dbg_xnu.cfg.patch`. I also
recommend setting up `KDK_PATH` to point to a directory containing copies of any kernelcaches you
will be debugging to reduce the amount of memory IDA downloads off the device. You will also need
to be sure to always use hardware breakpoints rather than software breakpoints, which are currently
unsupported.

KTRW should work out of the box with IDA Pro 7.5.

Open the kernelcache corresponding to the device in IDA Pro, select the remote XNU debugger, and
connect to the port on which `ktrw_usb_proxy` is listening. You should see IDA rebase the
kernelcache and start downloading data off the device. This may take a long time to complete.


Adding support for new platforms
---------------------------------------------------------------------------------------------------

Kernel extensions are linked against the symbols supplied in the `ktrw_gdb_stub/kernel_symbols`
directory. The files in this directory are named `<hardware-version>_<build-version>.txt`, where
`<hardware-version>` is the hardware identifier (e.g. iPhone10,1) and `<build-version>` is the
iOS build version (e.g. 16C101). There must be a single file per kernelcache UUID, so more
hardware/build version pairs may be supported than what is implied by the file name. Each file
should declare all supported hardware/build pairs in the header.


Breakpoints and watchpoints
---------------------------------------------------------------------------------------------------

KTRW currently uses hardware breakpoints and watchpoints. The hardware supports 6 breakpoints and 4
watchpoints, which should be sufficient for most basic debugging tasks. It is possible to add
support for software breakpoints.


Dynamic code generation
---------------------------------------------------------------------------------------------------

Currently, LLDB automatically disables dynamic code generation when debugging Darwin kernels
without testing whether or not the feature is supported, which breaks the ability to call kernel
functions from within LLDB.

As a workaround, you can build LLDB from source and comment out the line
`process->SetCanRunCode(false)` from `DynamicLoaderDarwinKernel.cpp`. This will allow you to run
`call` and `expression` commands in LLDB that are evaluated in the iOS kernel.


Debugging the reset vector
---------------------------------------------------------------------------------------------------

While it is possible to use the CoreSight External Debug registers to debug execution of the reset
vector before the MMU has been enabled, this use case is not supported by KTRW. The main reason is
that KTRW disables core resets while the device is plugged in and active (i.e. not sleeping) anyway
in order to make the KTRR bypass partially persistent.


A note on security and safety
---------------------------------------------------------------------------------------------------

Do not run KTRW on your personal iPhone: only run it on a dedicated research device that you do not
mind permanently damaging. KTRW expects the kernel task port to be exposed to unprivileged
applications, which critically compromises the system's security. Additionally, KTRW operates by
running a debugger on a single core that never sleeps, which consumes a lot of power and generates
excessive heat that could permanently damage the device or cause physical burns.


---------------------------------------------------------------------------------------------------
Developed and maintained by Brandon Azad of Google Project Zero, <bazad@google.com>







---

KTRW
===================================================================================================

KTRW is an iOS kernel debugger for devices with an A11 SoC, such as the iPhone 8. It leverages
debug registers present on these devices to bypass KTRR, remap the kernel as writable, and load a
kernel extension that implements a GDB stub, allowing full-featured kernel debugging with LLDB or
IDA Pro over a standard Lightning to USB cable.


Bypassing KTRR
---------------------------------------------------------------------------------------------------

KTRR was introduced with the A10 as a means of locking down critical kernel data (including all
executable code) to prevent it from being modified, even by an attacker with a kernel memory
read/write capability. However, on A11 SoCs, the ARMv8 External Debug registers and a proprietary
register called DBGWRAP were left enabled. This makes it possible to subvert execution of the reset
vector on these devices, skipping the MMU's KTRR initialization and setting a custom page table
base that remaps the kernel as writable. Once KTRR has been disabled, it becomes possible to
execute dynamically loaded kernel code, i.e., load kernel extensions.

Note that even though the kernel is remapped as writable, the physical pages spanned by the AMCC
RoRgn remain protected by the memory controller, and thus writes to these physical pages will be
discarded. Bypassing KTRR on the MMU does not defeat KTRR on the AMCC, and thus the only way to
remap the kernel as writable is to copy the kernel data in the AMCC RoRgn onto new, writable
physical pages. But since the AMCC is still protecting the original physical pages, and since the
reset vector executes from a physical address inside the AMCC RoRgn, the reset vector cannot be
persistently modified to disable KTRR automatically on reset without a more powerful capability
(such as a bootchain vulnerability). Thus, the KTRR bypass will disappear once the core resets
normally (that is, without being hijacked using the debug registers from another core). This means
that the KTRR bypass is not persistent: it will be lost once the device sleeps.


Using KTRW
---------------------------------------------------------------------------------------------------

KTRW consists of three components: the `ktrw_gdb_stub.ikext` kernel extension, the `ktrw_usb_proxy`
USB-to-TCP proxy utility, and the `ktrw_kext_loader` tool to load kernel extensions. Depending on
how KTRW is being used, `ktrw_kext_loader` can be built as a command line tool or as an iOS app.

First, build the kernel extension:

	$ cd ktrw_gdb_stub
	$ make

The compiled iOS kext will automatically be copied to the `ktrw_kext_loader/kexts/` directory.

Next, build the USB-to-TCP proxy utility:

	$ cd ktrw_usb_proxy
	$ make

`ktrw_usb_proxy` is needed to communicate with the kernel extension over USB and relay the data
over TCP so that LLDB can connect. It will print the data being exchanged over the connection. Run
`ktrw_usb_proxy` with the port number LLDB will connect to:

	$ ./ktrw_usb_proxy 39399

Next, build `ktrw_kext_loader`. There are two modes of operation (depending on how the kernel task
port is being exposed), and the build process depends on which is being used.

If KTRW is being run with [checkra1n], then build `ktrw_kext_loader` as a command-line tool and scp
the necessary files to the device:

	$ cd ktrw_kext_loader
	$ make
	$ codesign -s '*' -f --entitlements ktrw_kext_loader.entitlements ktrw_kext_loader
	$ scp ktrw_kext_loader iphone:
	$ scp -r kernel_symbols iphone:
	$ scp kexts/ktrw_gdb_stub.ikext iphone:

Once all the necessary files are in place, ssh into the phone and run the kext loader:

	$ ssh iphone
	# ./ktrw_kext_loader ktrw_gdb_stub.ikext
	[+] Platform: iPhone10,1 17B102
	[+] task_for_pid(0) = 0x907
	[!] Could not find the kernel base address
	[!] Trying to find the kernel base address using an unsafe heap scan!
	[+] KASLR slide is 0x178e4000
	[+] Kext ktrw_gdb_stub.ikext loaded at address 0xffffffe0ca1a0000

A separate process should be used if the kernel task port is exposed via host special port 4, for
example after a kernel exploit. In this case, simply open the `ktw_kext_loader` project in Xcode
and run the app on a connected A11 iPhone to load `ktrw_gdb_stub.ikext` into the kernel and start
debugging.

Once the kext has loaded (using either method), it will claim one CPU core for itself and halt the
remaining cores. It will also hijack the Synopsys USB 2.0 OTG controller from the kernel so that it
can communicate with the host. As a result, the host will not see the iPhone as an iOS device and
the phone (once it has been resumed) will not be able to send data over USB as normal.

After this, you are ready to debug the device.

[checkra1n]: https://checkra.in


Debugging with LLDB
---------------------------------------------------------------------------------------------------

Use LLDB to connect to `ktrw_usb_proxy` and communicate with `ktrw_gdb_stub.ikext`. Here I have
connected to an iPhone 8 running iOS 12.1.2:

	$ lldb kernelcache.iPhone10,1.16C101
	(lldb) target create "kernelcache.iPhone10,1.16C101"
	Current executable set to 'kernelcache.iPhone10,1.16C101' (arm64).
	(lldb) settings set plugin.dynamic-loader.darwin-kernel.load-kexts false
	(lldb) gdb-remote 39399
	Kernel UUID: 94463A80-7B38-3176-8872-0B8E344C7138
	Load Address: 0xfffffff027e04000
	Kernel slid 0x20e00000 in memory.
	Loaded kernel file kernelcache.iPhone10,1.16C101
	Process 2 stopped
	Target 0: (kernelcache.iPhone10,1.16C101) stopped.
	(lldb)

You can use `thread list` to list the code running on each physical CPU core. (Note that one core
is reserved for the debugger itself, so it will not show up in the list.)

	(lldb) th l
	Process 2 stopped
	* thread #1: tid = 0x0002, 0xfffffff027ffda18 kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol1734$$kernelcache.iPhone10,1.16C101 + 272
	  thread #2: tid = 0x0003, 0xfffffff027ffda18 kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol1734$$kernelcache.iPhone10,1.16C101 + 272
	  thread #3: tid = 0x0004, 0xfffffff027ffda18 kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol1734$$kernelcache.iPhone10,1.16C101 + 272
	  thread #4: tid = 0x0005, 0xfffffff027ffda18 kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol1734$$kernelcache.iPhone10,1.16C101 + 272
	  thread #5: tid = 0x0006, 0xfffffff027ffda18 kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol1734$$kernelcache.iPhone10,1.16C101 + 272
	(lldb)

Because KTRR has been disabled in the MMU and the kernel has been remapped as read/write, it is
possible to patch kernel memory:

	(lldb) x/12wx 0xfffffff027e04000
	0xfffffff027e04000: 0xfeedfacf 0x0100000c 0x00000000 0x00000002
	0xfffffff027e04010: 0x00000016 0x00001068 0x00200001 0x00000000
	0xfffffff027e04020: 0x00000019 0x00000188 0x45545f5f 0x00005458
	(lldb) mem wr -s 4 0xfffffff027e04000 0x11223344 0x55667788
	(lldb) x/12wx 0xfffffff027e04000
	0xfffffff027e04000: 0x11223344 0x55667788 0x00000000 0x00000002
	0xfffffff027e04010: 0x00000016 0x00001068 0x00200001 0x00000000
	0xfffffff027e04020: 0x00000019 0x00000188 0x45545f5f 0x00005458

Resume executing the kernel with `continue`. You can interrupt it at any time with `^C`:

	(lldb) c
	Process 2 resuming
	(lldb) ^C
	Process 2 stopped
	Target 0: (kernelcache.iPhone10,1.16C101) stopped.
	(lldb)

You can set breakpoints as usual. KTRW currently only supports hardware breakpoints, but LLDB will
automatically detect this and set the appropriate breakpoint type:

	(lldb) b 0xfffffff0282753b4
	Breakpoint 1: where = kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol4960$$kernelcache.iPhone10,1.16C101, address = 0xfffffff0282753b4
	(lldb) c
	Process 2 resuming
	Process 2 stopped
	* thread #4, stop reason = breakpoint 1.1
	    frame #0: 0xfffffff0282753b4 kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol4960$$kernelcache.iPhone10,1.16C101
	kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol4960$$kernelcache.iPhone10,1.16C101:
	->  0xfffffff0282753b4 <+0>:  sub    sp, sp, #0x80             ; =0x80
	    0xfffffff0282753b8 <+4>:  stp    x28, x27, [sp, #0x20]
	    0xfffffff0282753bc <+8>:  stp    x26, x25, [sp, #0x30]
	    0xfffffff0282753c0 <+12>: stp    x24, x23, [sp, #0x40]
	Target 0: (kernelcache.iPhone10,1.16C101) stopped.
	(lldb)

Single-stepping works as expected:

	(lldb) si
	Process 2 stopped
	* thread #4, stop reason = instruction step into
	    frame #0: 0xfffffff0282753b8 kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol4960$$kernelcache.iPhone10,1.16C101 + 4
	kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol4960$$kernelcache.iPhone10,1.16C101:
	->  0xfffffff0282753b8 <+4>:  stp    x28, x27, [sp, #0x20]
	    0xfffffff0282753bc <+8>:  stp    x26, x25, [sp, #0x30]
	    0xfffffff0282753c0 <+12>: stp    x24, x23, [sp, #0x40]
	    0xfffffff0282753c4 <+16>: stp    x22, x21, [sp, #0x50]
	Target 0: (kernelcache.iPhone10,1.16C101) stopped.
	(lldb) si
	Process 2 stopped
	* thread #4, stop reason = instruction step into
	    frame #0: 0xfffffff0282753bc kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol4960$$kernelcache.iPhone10,1.16C101 + 8
	kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol4960$$kernelcache.iPhone10,1.16C101:
	->  0xfffffff0282753bc <+8>:  stp    x26, x25, [sp, #0x30]
	    0xfffffff0282753c0 <+12>: stp    x24, x23, [sp, #0x40]
	    0xfffffff0282753c4 <+16>: stp    x22, x21, [sp, #0x50]
	    0xfffffff0282753c8 <+20>: stp    x20, x19, [sp, #0x60]
	Target 0: (kernelcache.iPhone10,1.16C101) stopped.
	(lldb)

Watchpoints are also supported:

	(lldb) wa s e -s 8 -w read_write -- 0xfffffff027e04000
	Watchpoint created: Watchpoint 1: addr = 0xfffffff027e04000 size = 8 state = enabled type = rw
	    new value: 6153737367135073092
	(lldb) reg w x1 0xfffffff027e04000
	(lldb) c
	Process 2 resuming
	
	Watchpoint 1 hit:
	old value: 6153737367135073092
	new value: 6153737367135073092
	Process 2 stopped
	* thread #5, stop reason = watchpoint 1
	    frame #0: 0xfffffff028275418 kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol4960$$kernelcache.iPhone10,1.16C101 + 100
	kernelcache.iPhone10,1.16C101`___lldb_unnamed_symbol4960$$kernelcache.iPhone10,1.16C101:
	->  0xfffffff028275418 <+100>: and    x21, x9, x10
	    0xfffffff02827541c <+104>: add    x10, x11, x10
	    0xfffffff028275420 <+108>: add    x8, x10, w8, sxtw
	    0xfffffff028275424 <+112>: and    x26, x9, x8
	Target 0: (kernelcache.iPhone10,1.16C101) stopped.
	(lldb) x/4i $pc-8
	    0xfffffff028275410: 0x9360fd29   asr    x9, x9, #32
	    0xfffffff028275414: 0xa9402eea   ldp    x10, x11, [x23]
	->  0xfffffff028275418: 0x8a0a0135   and    x21, x9, x10
	    0xfffffff02827541c: 0x8b0a016a   add    x10, x11, x10
	(lldb) reg r x23 x10 x11
	     x23 = 0xfffffff027e04000
	     x10 = 0x5566778811223344
	     x11 = 0x0000000200000000
	(lldb)

LLDB limits watchpoints to 1, 2, 4, or 8 bytes in size, even though the hardware supports even
larger watchpoints.

Unfortunately, older versions of LLDB do not automatically detect kernelcaches on iOS 12.2 or later
because the kASLR slide has a finer granularity. However, kernelcache detection should work as
expected on HEAD when LLDB is built from source.
