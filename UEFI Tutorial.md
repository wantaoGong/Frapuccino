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

如:

​	![1540294441449](/home/ai/.config/Typora/typora-user-images/1540294441449.png)





### UEFI中的Protocol



#### 如何使用Protocol

一般步骤:

①通过gBS->OpenProtocol(或者HandleProtocol/LocateProtocol)找出Protocol对象

②使用Protocol提供的服务

③通过gBS->CloseProtocol关闭打开的Protocol



例子:

​	

















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