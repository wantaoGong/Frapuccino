# UEFI lab

### 一、edk2 环境搭建

一、到[这里](https://github.com/tianocore/edk2/releases)下载UEFI SDK 2018（Release版本）并解压（我是解压到Downloads目录），将解压好的目录改名为edk2。

二、安装必备的工具（iasl已经改名为acpica-tools，不过继续用iasl这个旧名字也可以装上）：

```bash
sudo apt-get install build-essential uuid-dev iasl git gcc-5 nasm qemu-system-x86
```

三、编译OVMF。执行以下命令：

```bash
//切换到edk2目录下执行
make -C BaseTools
. edksetup.sh

cd ..
make -C edk2/BaseTools

cd edk2
export EDK_TOOLS_PATH=$HOME/Downloads/edk2/BaseTools
. edksetup.sh BaseTools

build -a X64 -p OvmfPkg/OvmfPkgX64.dsc -t GCC5 -b RELEASE
```

四、测试OVMF。执行以下命令：

```bash
cd ~/Downloads
cp edk2/Build/OvmfX64/RELEASE_GCC5/FV/OVMF.fd ./
qemu-system-x86_64 -bios ./OVMF.fd
```



五 若关闭了终端重新打开,需要重新配置变量,命令如下:

```bash
. edksetup.sh 或者 source edksetup.sh
```



此时，我们应该看到TianoCore图标在QEMU虚拟机中显示，然后系统会进入UEFI Shell(可能会比较慢)。这代表我们成功地编译了OVMF。





#### 模拟器使用(我未试成功,下面这是我找到的,你可以试试)

EmulatorPkg提供了一个环境，可以在不可能完全兼容UEFI的环境下模拟UEFI环境。（例如，在OS进程承载UEFI仿真环境的OS下运行.(这个我没有试成功)

- 构建并运行
  - 具有X窗口的类似posix的环境
    - Linux的
    - OS X.
  - Windows环境
    - Win10（已验证）
    - Win8（未经验证）

#### 如何建立和运行

**您可以使用以下命令进行构建。**

- Windows中的32位仿真器：

  `build -p EmulatorPkg\EmulatorPkg.dsc -t VS2017 -D WIN_SEC_BUILD -a IA32`

- Windows中的64位模拟器：

  `build -p EmulatorPkg\EmulatorPkg.dsc -t VS2017 -D WIN_SEC_BUILD -a X64`

- Linux中的32位仿真器：

  `build -p EmulatorPkg/EmulatorPkg.dsc -t GCC5 -D UNIX_SEC_BUILD -a IA32`

- Linux中的64位仿真器：

  `build -p EmulatorPkg/EmulatorPkg.dsc -t GCC5 -D UNIX_SEC_BUILD -a X64`

**您可以使用以下命令启动/运行模拟器：**

- Windows中的32位仿真器：

  `cd Build\EmulatorIA32\DEBUG_VS2017\IA32\ && WinHost.exe`

- Windows中的64位模拟器：

  `cd Build\EmulatorX64\DEBUG_VS2017\X64\ && WinHost.exe`

- Linux中的32位仿真器：

  `cd Build/EmulatorIA32/DEBUG_GCC5/IA32/ && ./Host`

- Linux中的64位仿真器：

  `cd Build/EmulatorX64/DEBUG_GCC5/X64/ && ./Host`

**在具有bash shell的类似posix的环境中，您可以使用EmulatorPkg / build.sh来简化构建和运行模拟器。**

例如，要构建+运行：

```
$ EmulatorPkg/build.sh`
`$ EmulatorPkg/build.sh run
```

构建体系结构将与主机的体系结构相匹配。

在X64主机上，您也可以构建+运行IA32模式：

```
$ EmulatorPkg/build.sh -a IA32`
`$ EmulatorPkg/build.sh -a IA32 run
```



### Hello world!

1 编译

```bash
ai@local:~/Documents/edk2$ source edksetup.sh //配置环境变量
ai@local:~/Documents/edk2$ build -p AppPkg/AppPkg.dsc -a X64 -m AppPkg/Applications/Hello/Hello.inf -t GCC5 -b RELEASE
/*****************/
build 是一个程序,用来自动化编译模块
-p 参数表示*Pkg 要编译的模块位于哪个包下面
-a 表示系统结构是多少位的,32bit就用IA32,64bit就用X64
-m 表示要编译哪个模块,这里编译的是Hello这个模块,参数对应的值为Hello目录下的Hello.inf文件路径,*.inf文件相当于Makefile文件
-t 表示使用哪个编译器,这里使用GCC5这个编译器
-b 表示想编译生成的*.efi文件是什么版本,有两种RELEASE(release)/DEBUG(debug)
/*****************/
```

如图:

![1540292372988](/home/ai/.config/Typora/typora-user-images/1540292372988.png)

编译后有提示: 

​	成功------done

​	失败------failed

成功则生成*.efi文件,所有生成的*.efi文件都会在edk2/build下的相应目录

如:Hello.efi文件位于Build/AppPkg/RELEASE_GCC5/X64/AppPkg/Applications/Hello/Hello/OUTPUT目录下

​	![1540740380160](/home/ai/.config/Typora/typora-user-images/1540740380160.png)

2\在固件中运行





### 二、UEFI中的Protocol



#### 如何使用Protocol

一般步骤:

①通过gBS->OpenProtocol(或者HandleProtocol/LocateProtocol)找出Protocol对象

②使用Protocol提供的服务

③通过gBS->CloseProtocol关闭打开的Protocol



例子: 读取设备路径

```C
#include <Base.h>
#include <Library/BaseLib.h>
#include <Library/BaseMemoryLib.h>
#include <Library/DevicePathLib.h>
#include <Library/PrintLib.h>
#include <Library/UefiBootServicesTableLib.h>
#include <Library/UefiLib.h>
#include <Protocol/BlockIo.h>
#include <Protocol/DevicePath.h>
#include <Protocol/DevicePathToText.h>
#include <Protocol/DiskIo.h>
#include <Uefi.h>
#include <Uefi/UefiGpt.h>

EFI_STATUS
EFIAPI
ReadDevicePath(IN EFI_HANDLE ImageHandle, IN EFI_SYSTEM_TABLE *SystemTable) {
  EFI_STATUS Status;
  UINTN HandleIndex, HandleCount;
  EFI_HANDLE *DiskControllerHandles = NULL;
  EFI_DISK_IO_PROTOCOL *DiskIo;

  /*
  gBS->LocateHandleBuffer()
  gBS是启动服务的一个指针,几乎提供的所有启动服务都需要使用这个指针
  LocateHandleBuffer()这个函数的目的是为了找到提供协议提供某个协议的所有设备,在这里就是找提供磁盘IO的所有设备.这个函数是用来找到支持某协议的所有设备,如果想要使用某设备所提供的该协议需要在后续的操作中需要遍历一遍看看.
  */
  Status = gBS->LocateHandleBuffer(ByProtocol, &gEfiDiskIoProtocolGuid, NULL,
                                   &HandleCount, &DiskControllerHandles);

  if (!EFI_ERROR(Status)) {
    CHAR8 gptHeaderBuf[512];

    EFI_PARTITION_TABLE_HEADER *gptHeader =
        (EFI_PARTITION_TABLE_HEADER *)gptHeaderBuf;

    for (HandleIndex = 0; HandleIndex < HandleCount; HandleIndex++) {
      /*open EFI_DISK_IO_PROTOCOL  */
      /*
      上面LocateHandleBuffer()已经找到了支持磁盘IO的所有设备,现在来遍历一遍
      HandleProtocol()函数是用来打开指定设备的某个协议,在这里用来打开磁盘的磁盘IO协议
      */
      Status = gBS->HandleProtocol(DiskControllerHandles[HandleIndex],
                                   &gEfiDiskIoProtocolGuid, (VOID **)&DiskIo);

      if (!EFI_ERROR(Status)) {
        {
          EFI_DEVICE_PATH_PROTOCOL *DiskDevicePath;
          EFI_DEVICE_PATH_TO_TEXT_PROTOCOL *Device2TextProtocol = 0;
          CHAR16 *TextDevicePath = 0;
          /*1.open EFI_DEVICE_PATH_PROTOCOL  */
          // Status = gBS->OpenProtocol(DiskControllerHandles[HandleIndex],
          //                            &gEfiDevicePathProtocolGuid,
          //                            (VOID **)&DiskDevicePath, ImageHandle,
          //                            NULL, EFI_OPEN_PROTOCOL_GET_PROTOCOL);
          /*
          这里的HandleProtocol()用来打开设备的设备路径协议
          */
          Status = gBS->HandleProtocol(DiskControllerHandles[HandleIndex],
                                       &gEfiDevicePathProtocolGuid,
                                       (VOID **)&DiskDevicePath);
          if (!EFI_ERROR(Status)) {
            if (Device2TextProtocol == 0)
              /*
               *这里的LocateProtocol()用来找到提供支持DevicePathToText协议的第一个实例
               */
              Status = gBS->LocateProtocol(&gEfiDevicePathToTextProtocolGuid,
                                           NULL, (VOID **)&Device2TextProtocol);
            /*2.use EFI_DEVICE_PATH_PROTOCOL 得到文本格式的 Device Path  */
            TextDevicePath = Device2TextProtocol->ConvertDevicePathToText(
                DiskDevicePath, TRUE, TRUE);
            Print(L"%s\n", TextDevicePath);
            if (TextDevicePath) gBS->FreePool(TextDevicePath);
            /*3. close EFI_DEVICE_PATH_PROTOCO */
            Status = gBS->CloseProtocol(DiskControllerHandles[HandleIndex],
                                        &gEfiDevicePathProtocolGuid,
                                        ImageHandle, NULL);
          }
        }
        {
          /*	*/
          EFI_BLOCK_IO_PROTOCOL *BlockIo =
              *(EFI_BLOCK_IO_PROTOCOL **)(DiskIo + 1);

          EFI_BLOCK_IO_MEDIA *Media = BlockIo->Media;

          /*read 1 part*/
          Status =
              DiskIo->ReadDisk(DiskIo, Media->MediaId, 512, 512, gptHeader);

          /*charck GPT mark*/
          if ((!EFI_ERROR(Status)) &&
              (gptHeader->Header.Signature == 0x5452415020494645)) {
            UINT32 CRCsum;
            UINT32 GPTHeaderCRCsum = (gptHeader->Header.CRC32);
            gptHeader->Header.CRC32 = 0;
            gBS->CalculateCrc32(gptHeader, (gptHeader->Header.HeaderSize),
                                &CRCsum);
            if (GPTHeaderCRCsum == CRCsum) {
              // find out GPT Header
            }
          }
        }
      }
    }
    gBS->FreePool(DiskControllerHandles);
  }
  return 0;
}
```



| OpenProtocol            | 打开Protocol                                                 |
| ----------------------- | :----------------------------------------------------------- |
| HandleProtocol          | 打开Protocol, OpenProtocol的简化版                           |
| LocateProtocol          | 找出系统中指定Protocol的第一个实例                           |
| LocateHandleBuffer      | 找出支持指定Protocol的所有Handle,系统负责分配内存,调用者负责释放内存 |
| LocateHandle            | 找出支持指定Protocol的所有Handle,调用者负责分配和释放内存    |
| OpenProtocolInformation | 返回指定Protocol的打开信息                                   |
| ProtocolsPerHandle      | 找出指定Handle上安装的所有Protocol                           |
| CloseProtocol           | 关闭Protocol                                                 |

以上读取设备分区表的代码已经写好,还得给这个源码写一个类似Makefile的文件,以便好编译和使用.文件要.inf结尾.

```makefile
[Defines]
  INF_VERSION                    = 0x00010006
  BASE_NAME                      = ReadDevicePath 
  FILE_GUID                      = 4ea97c46-7491-4dfd-b442-747010f3ce5f
  MODULE_TYPE                    = UEFI_APPLICATION
  VERSION_STRING                 = 0.1
  ENTRY_POINT                    = ReadDevicePath 
[Sources]#标明源文件
  ReadDevicePath.c

[Packages]
  MdePkg/MdePkg.dec
[Protocols]#用协议的GUID标明源文件用到哪些协议
  gEfiDiskIoProtocolGuid
  gEfiBlockIoProtocolGuid
  gEfiDevicePathProtocolGuid
  gEfiDevicePathToTextProtocolGuid
  gEfiSimpleFileSystemProtocolGuid
  gEfiFirmwareVolume2ProtocolGuid

[LibraryClasses]   
  UefiApplicationEntryPoint
  UefiLib

```

以上两个文件为一个模块,放在同一个目录下.如:

![1540743273490](/home/ai/.config/Typora/typora-user-images/1540743273490.png)





然后编译这个模块.最后生成.efi可执行文件,这个文件只能在UEFI环境下执行,在操作系统上是执行不了的.切记.

在编译前需要把该模块的配置文件,也就是.inf文件的文件路径放入你想放在某个包下面,也就是目录名是*Pkg特征的(一般是这样).dsc文件里面的components标记后.如:我放在AppPkg/AppPkg.dsc里面

![1540743639258](/home/ai/.config/Typora/typora-user-images/1540743639258.png)

在我这儿编译命令为:

```bash
ai@local:~/Documents/edk2$ build -p AppPkg/AppPkg.dsc -m lab/ReadDevicePath/ReadDevicePath.inf -a X64 -t GCC5 -b RELEASE
#-m 表示模块
#-a 表示arch
#-t 表示编译器
#-b 表示生成的版本
```

编译成功会会在build目录下的相应目录下能够找到生成的ReadDevicePath.efi文件



执行:











附录:

<<qemu使用>>

```
`-hda file' `-hdb file' `-hdc file' `-hdd file'
使用 file 作为硬盘0、1、2、3镜像。
`-fda file' `-fdb file'
使用 file 作为软盘镜像，可以使用 /dev/fd0 作为 file 来使用主机软盘。
`-cdrom file'
使用 file 作为光盘镜像，可以使用 /dev/cdrom 作为 file 来使用主机 cd-rom。
`-boot [a|c|d]'
从软盘(a)、光盘(c)、硬盘启动(d)，默认硬盘启动。
`-snapshot'
写入临时文件而不写回磁盘镜像，可以使用 C-a s 来强制写回。
`-m megs'
设置虚拟内存为 msg M字节，默认为 128M 字节。
`-smp n'
设置为有 n 个 CPU 的 SMP 系统。以 PC 为目标机，最多支持 255 个 CPU。
`-nographic'
禁止使用图形输出。
其他：
可用的主机设备 dev 例如：
vc
虚拟终端。
null
空设备
/dev/XXX
使用主机的 tty。
file: filename
将输出写入到文件 filename 中。
stdio
标准输入/输出。
pipe：pipename
命令管道 pipename。
等。
使用 dev 设备的命令如：
`-serial dev'
重定向虚拟串口到主机设备 dev 中。
`-parallel dev'
重定向虚拟并口到主机设备 dev 中。
`-monitor dev'
重定向 monitor 到主机设备 dev 中。
其他参数：
`-s'
等待 gdb 连接到端口 1234。
`-p port'
改变 gdb 连接端口到 port。
`-S'
在启动时不启动 CPU， 需要在 monitor 中输入 'c'，才能让qemu继续模拟工作。
`-d'
输出日志到 qemu.log 文件。
```

load VlanConfigDxe.efi ArpDxe.efi Ip4Dxe.efi Tcp4Dxe.efi Udp4Dxe.efi Dhcp4Dxe.rfi MnpDxe.efi SnpDxe.efi

ifconfig -?  //获取帮助,很详细,教你如何配置网络参数



qemu-system-x86_64 -L . --bios ovmf/OVMF.fd -hda fat:rw:hda-contents -net nic -net user