[TOC]



## TA0004 - 提权 Privilege Escalation

在这个环节攻击者试图获得系统或网络上更高级别的权限（提权）。

攻击者通常可以使用无特权账户进入和探索网络，但是需要更高的权限才能实现目标。常见的方法是利用系统弱点、错误配置和漏洞进行权限提升，以获取系统/根级别帐户、本地管理员帐户、具有与管理员类似访问权限的帐户、具有对特定系统的访问权限或执行特定功能的帐户。这些技术通常与持久性技术（[TA0003](https://attack.mitre.org/tactics/TA0003/)）有重叠之处，以允许攻击者在系统的上下文中持久地执行操作。



### [T1134 - Access Token Manipulation  访问令牌操作](https://attack.mitre.org/techniques/T1134/)

> Tactic: Defense Evasion, Privilege Escalation
> Platform: Windows
> Permissions Required: User, Administrator
> Effective Permissions: SYSTEM
> Data Sources: API monitoring, Access tokens, Process monitoring, Process command-line parameters
> CAPEC ID: [CAPEC-633](https://capec.mitre.org/data/definitions/633.html)
> Contributors: Tom Ueltschi @c_APT_ure; Travis Smith, Tripwire; Robby Winchester, @robwinchester3; Jared Atkinson, @jaredcatkinson
> Version: 1.0

#### 概述

Windows 使用访问令牌来确定正在运行的进程的所有权，用户可以对访问令牌进行操作，使正在运行的进程看起来像是属于其他用户，而不是启动该进程的实际用户。当这种情况发生时，该进程还接受与新令牌相关联的安全上下文（security context）。Microsoft 提倡使用访问令牌作为安全最佳实践，管理员应该以普通用户身份登录，但使用内置的访问令牌操作命令 [runas](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-xp/bb490994(v=technet.10)?redirectedfrom=MSDN) ，以管理员权限运行他们的工具。

攻击者可以使用访问令牌在不同的用户或系统的安全上下文中进行操作，以执行命令和逃避检测。他们会使用内置的 Windows API 函数从现有的进程中复制访问令牌，这也被称之为令牌窃取。然而，令牌窃取的前提是攻击者必须已经拥有了管理员权限，之后攻击者通过令牌窃取将其身份的安全上下文从管理员级别提升到系统级别（SYSTEM）。

攻击者可以通过以下三种方法利用访问令牌：

- 令牌模拟/窃取（Token Impersonation/Theft）：攻击者使用 `DuplicateToken(Ex)` 创建一个新的访问令牌，并用该令牌复制现有的令牌。这个令牌可以与 `ImpersonateLoggedOnUser` 一起使用，以允许调用线程在用户的安全上下文中模拟一次登录，或者与 `SetThreadToken` 一起将模拟的令牌分配给线程。当目标用户在系统上有一个非网络的登录会话时，这非常有用。
- 使用令牌创建进程（Create Process with a Token）：攻击者使用 `DuplicateToken(Ex)` 复制并创建一个新的访问令牌，并使用 `CreateProcessWithTokenW` 创建一个新进程，并在被模拟用户的安全上下文中运行。当在不同用户的安全上下文中创建新进程时，这非常有用。
- 制作和模拟令牌（Make and Impersonate Token）：当攻击者拥有用户名和密码，但是该用户没有登录过系统。这时，攻击者就可以使用 `LogonUser` 函数为用户创建登录会话，该函数将返回新会话的访问令牌的一份拷贝，之后攻击者可以使用 `SetThreadToken` 将该令牌分配给线程。

需要注意的是，任何普通用户都可以使用 `runas` 命令和 `Windows API` 函数来创建模拟令牌，它不需要管理员帐户权限。

Meterpreter 中的 payload 允许任意的令牌操作，并使用模拟令牌来提升权限；Cobalt Strike beacon 中的 payload 也允许任意的令牌模拟，并创建新的令牌。

#### 检测

- 审计命令行活动来检测令牌操作，比如对 runas 常见命令做匹配。



### [T1015 - Accessibility Features 辅助功能](https://attack.mitre.org/techniques/T1015/)

> Tactic: Persistence, Privilege Escalation
> Platform: Windows
> Permissions Required: Administrator
> Effective Permissions: SYSTEM
> Data Sources: Windows Registry, File monitoring, Process monitoring
> CAPEC ID: [CAPEC-558](https://capec.mitre.org/data/definitions/558.html)
> Contributors: Paul Speulstra, AECOM Global Security Operations Center
> Version: 1.0

Windows 自带的辅助功能（轻松使用）特性可以在用户登录之前通过键组合启动，攻击者可以修改这些程序的启动方式，从而在不登录系统的情况下调用命令提示符或后门程序。

两个常见的辅助功能是 Windows 粘滞键（C:\Windows\System32\sethc.exe，5 次 shift）和 Windows 辅助工具管理器（C:\Windows\System32\utilman，Windows + U）。根据不同的 Windows 版本，攻击者可能会以不同的方式利用这些特性。在较新的 Windows 版本中，被替换的二进制文件需要进行数字签名校验，二进制文件必须驻留在 `%systemdir%\` 目录下，并且受到 WFP/WRP（Windows File / Resource Protection） 的保护。“调试器（debugger）”方法很可能成为一种利用方案，因为它不需要替换相应的二进制文件。

两种方法举例如下：

在 Windows XP 和 Windows Server 2003 R2 及其以后的版本，可以直接通过对可执行程序的二进制文件进行替换来实现攻击。比如，将 C:\Windows\System32 目录下的 utilman.exe 替换为 cmd.exe 或另一个后门程序。之后，在通过远程程桌面协议（[RDP](https://attack.mitre.org/techniques/T1076)）连接时，在登录界面按下适当的组合键就可以导致所替换的文件以 SYSTEM 权限执行。

对于 Windows Vista 和 Windows Server 2008 及其以后的版本，需要使用“调试器”方法。比如，可以修改注册表项以配置 cmd.exe 或另一个后门程序作为某个辅助功能（例如 utilman.exe）的“调试器”。修改注册表后，在键盘上或使用 RDP 连接时在登录界面按下适当的组合键将导致“调试器”程序以 SYSTEM 权限执行。

其他可能以类似方式利用的辅助功能：

- 屏幕键盘: C:\Windows\System32\osk.exe
- 放大镜: C:\Windows\System32\Magnify.exe
- 讲述人: C:\Windows\System32\Narrator.exe
- 显示切换器: C:\Windows\System32\ DisplaySwitch.exe
- 应用程序切换器: C:\Windows\System32\ AtBroker.exe



### [T1182 - AppCert DLLs 注入](https://attack.mitre.org/techniques/T1182/)

> Tactic: Persistence, Privilege Escalation
> Platform: Windows
> Permissions Required: Administrator, SYSTEM
> Effective Permissions: Administrator, SYSTEM
> Data Sources: Loaded DLLs, Process monitoring, Windows Registry
> Version: 1.0

利用 AppCertDlls 注册表项来实现 DLL 注入。Windows 中的大量 API 函数（CreateProcess, CreateProcessAsUser, CreateProcessWithLoginW, CreateProcessWithTokenW, WinExec）都会加载处于注册表 `HKLM\System\CurrentControlSet\Control\Session Manager\AppCertDlls` 下的 DLL 文件。所以只需要在 AppCertDlls 中写入 DLL 的绝对路径，就可以将此注册表项下的 DLL 加载到调用 Windows API 函数每个的进程中。

与进程注入（[Process Injection](https://attack.mitre.org/techniques/T1055)）类似，攻击者可以利用这个值在独立进程的上下文中加载和运行恶意 DLL，以获得持久性和权限提升。



### [T1103 - AppInit DLLs 注入](https://attack.mitre.org/techniques/T1103/)

> Tactic: Persistence, Privilege Escalation
> Platform: Windows
> Permissions Required: Administrator
> Data Sources: Loaded DLLs, System calls, Windows Registry, Process monitoring, Process command-line parameters
> Version: 1.0

利用 AppInit DLLs 注册表项来实现 DLL 注入。Windows 允许加载了 user32.dll 的进程加载处于注册表 `HKLM\Software\Microsoft\Windows NT\CurrentVersion\Windows\AppInit_DLLs` 或 `HKLM\Software\Wow6432Node\Microsoft\Windows NT\CurrentVersion\Windows\AppInit_DLLs` 下的 DLL 文件。所以只需要在 AppInit_DLLs 下写入 DLL 的绝对路径，并把数值改为 1，就可以使得所有加载 user32.dll 的进程全部加载目标路径的 DLL 文件。由于 user32.dll 是一个非常常用的库，在实际情况下，这个 DLL 几乎会被加载到每一个进程。

与进程注入（[Process Injection](https://attack.mitre.org/techniques/T1055)）类似，攻击者可以利用这个值在独立进程的上下文中加载和运行恶意 DLL，以获得持久性和权限提升。


在 Windows 8 及以后的版本中，当启用安全启动（secure boot）时，将禁用 AppInit DLL 的功能。



### [T1138 - Application Shimming 应用兼容性](https://attack.mitre.org/techniques/T1138/)

> Tactic: Persistence,Privilege Escalation
> Platform: Windows
> System Requirements: Secure boot disabled on systems running Windows 8 and later
> Permissions Required: Administrator
> Effective Permissions: Administrator, SYSTEM
> Data Sources: Loaded DLLs, Process monitoring, Windows Registry
> Version: 1.0

#### 概述

Application Shim（Microsoft Windows Application Compatibility Infrastructure/Framework，应用程序兼容性基础架构/框架）的作用是为了在操作系统中保持向后兼容性，shim 被用来充当程序和 Windows 操作系统之间的缓冲区，当程序执行时将引用 shim 缓存来确定程序是否需要使用自定义数据库（.sdb）。如果需要使用，则自定义数据库将根据需要使用 [Hooking](https://attack.mitre.org/techniques/T1179) 技术重定向代码，以便应用程序可以在不同的 Windows 版本下执行。

> 使用 sdbinst.exe 命令行工具来部署自定义数据库文件

当前已安装的兼容性修补程序（shims）的列表保存在以下目录：

- `%WINDIR%\AppPatch\sysmain.sdb`
- `hklm\software\microsoft\windows nt\currentversion\appcompatflags\installedsdb`

自定义数据库存储在：

- `%WINDIR%\AppPatch\custom & %WINDIR%\AppPatch\AppPatch64\Custom`
- `hklm\software\microsoft\windows nt\currentversion\appcompatflags\custom`

为了保证兼容性修补程序的安全，Windows 将它们设计为在用户模式下运行，不能修改内核，并且必须具有管理员权限才能进行安装。但是，某些兼容性修补程序可以绕过 UAC（[Bypass User Account Control](https://attack.mitre.org/techniques/T1088/)，RedirectEXE）、将 DLL 注入进程（InjectDLL）、禁用数据执行保护（Data Execution Prevention，DisableNX）和结构化异常处理（Structure Exception Handling，DisableSEH），以及截获内存地址（GetProcAddress）。类似于 Hooking 技术，攻击者可以利用这些兼容性修补程序执行一些恶意行为，如提高特权，安装后门，禁用安全防御软件，如 Windows Defender 等。

#### 检测

- 相关工具
  - Shim-Process-Scanner - 检查每个运行进程的内存中是否有任何 shim 标志
  - Shim-Detector-Lite - 检查自定义 shim 数据库的安装
  - Shim-Guard - 监视任何安装 shim 的注册表行为
  - ShimScanner - 检查在内存中活跃的兼容性修补程序
  - ShimCacheMem - 从内存中提取 shim 缓存

- 监视 sdbinst.exe 的进程执行和命令行参数



### [T1088 - Bypass User Account Control 绕过用户帐户控制](https://attack.mitre.org/techniques/T1088/)

> Tactic: Defense Evasion, Privilege Escalation
> Platform: Windows
> Permissions Required: User, Administrator
> Effective Permissions: Administrator
> Data Sources: System calls, Process monitoring, Authentication logs, Process command-line parameters
> Defense Bypassed: Windows User Account Control
> Contributors: Stefan Kanthak; Casey Smith
> Version: 1.0

Windows 用户帐户控制（User Account Control，UAC）允许程序通过提示用户是否对应用程序授权，来提升权限，以执行管理员权限下的任务，影响范围十分广泛。

如果 UAC 保护等级设置为最高等级以外的任何等级，则某些 Windows 应用程序可以在不通过 UAC 提示框的情况下进行权限提升，或执行某些可自动提权的 COM 对象。

目前已经公开过很多种绕过 UAC 的方式，Github 上的 [UACME](https://github.com/hfiref0x/UACME) 项目包含了一个完整了 UAC 可利用方式列表。此外，还有一些其他的绕过方式，比如使用 `eventvwr.exe` 可以自动提升权限并执行指定的二进制文件或脚本；由于 UAC 是单系统环境下的安全机制，所以在某一个系统上运行的进程的权限和完整性级别（Integrity Level），对于其他系统是未知的（默认具有高完整性级别），如果知道管理员帐户的凭据，则可以通过一些横向移动技术进行绕过。




### [T1038 - DLL Search Order Hijacking DLL 搜索顺序劫持 ](https://attack.mitre.org/techniques/T1038/)

> Tactic: Persistence, Privilege Escalation, Defense Evasion
> Platform: Windows
> System Requirements: Ability to add a DLL, manifest file, or .local file, directory, or junction.
> Permissions Required: User, Administrator, SYSTEM
> Effective Permissions: User, Administrator, SYSTEM
> Data Sources: File monitoring, DLL monitoring, Process monitoring, Process command-line parameters
> Defense Bypassed: Process whitelisting
> CAPEC ID: [CAPEC-471](https://capec.mitre.org/data/definitions/471.html)
> Contributors: Stefan Kanthak; Travis Smith, Tripwire
> Version: 1.0

Windows 系统使用一种常见的方法来查找需要加载到程序中的 DLL 动态链接库，攻击者可能会利用 Windows DLL 的搜索顺序和模糊指定 DLL 的程序，来获得持久性和权限提升。

攻击者可以利用 DLL 预加载（也被称为二进制植入攻击），使应用程序加载恶意 DLL。本地 DLL 预加载攻击的方法是将一个与模糊指定 DLL 同名的恶意 DLL 放在 Windows 程序对合法 DLL 进行搜索之前的位置，这个位置通常是程序当前的工作目录；当程序在加载 DLL 之前将其当前目录设置为远程位置（如 Web 共享）时，就会发生远程 DLL 预加载攻击。

攻击者还可以通过替换现有 DLL 或修改 `.manifest` 或 `.local` 重定向文件、目录或文件夹映射来直接修改程序加载 DLL 的方式，使程序加载恶意 DLL 文件。

最终获得到的权限由加载恶意 DLL 的程序的权限决定，根据程序权限的不同，攻击者可以使用这种技术将权限从用户升级到 Admin 或 SYSTEM 权限。



### [T1157 - Dylib Hijacking Dylib 劫持](https://attack.mitre.org/techniques/T1157/)

> Tactic: Persistence, Privilege Escalation
> Platform: macOS
> Permissions Required: User
> Effective Permissions: Administrator, root
> Data Sources: File monitoring
> CAPEC ID: [CAPEC-471](https://capec.mitre.org/data/definitions/471.html)
> Version: 1.0

macOS 和 OS X 系统使用一种常见的方法来查找需要加载到程序中的 Dylib 动态链接库，攻击者可以利用模糊路径来植入 Dylib，以获得特权升级或持久性。


攻击者可以通过查看应用程序使用了什么 Dylib，然后在合法 Dylib 的搜索路径之前植入同名的恶意 Dylib，这个位置通常是程序当前的工作目录。


如果将程序在高于当前用户的权限级别上运行，那么当将 Dylib 加载到应用程序中时，Dylib 也将在更高的权限上运行，使攻击者实现权限提升。



### [T1514 - Elevated Execution with Prompt 命令行权限提升](https://attack.mitre.org/techniques/T1514/)

> Tactic: Privilege Escalation
> Platform: macOS
> Permissions Required: Administrator, User
> Effective Permissions: root
> Data Sources: File monitoring, Process monitoring, API monitoring
> Contributors: Erika Noerenberg, @gutterchurl, Carbon Black; Jimmy Astle, @AstleJimmy, Carbon Black
> Version: 1.0

攻击者可以利用 `AuthorizationExecuteWithPrivileges` API 提示用户输入凭据，从而提升权限。这个 API 的最初目的是为应用程序开发人员提供一种使用 root 权限执行操作的简便方法，例如用于应用程序的安装或更新。当调用此 API 时，将提示用户输入他们的凭据，但不会验证请求 root 权限的程序来源是否可靠或者是否已经被恶意修改，攻击者可能会利用这个 API 结合钓鱼的方式来欺骗用户授权，以使恶意程序能够在 root 权限下执行。



### [T1519 - Emond 守护进程](https://attack.mitre.org/techniques/T1519/)

> Tactic: Persistence, Privilege Escalation
> Platform: macOS
> Permissions Required: Administrator
> Effective Permissions: root
> Data Sources: File monitoring, API monitoring
> Contributors: Ivan Sinyakov
> Version: 1.0

#### 概述

攻击者可以通过在事件触发器（predictable event triggers）上执行恶意指令来使用事件监视器守护进程（Event Monitor Daemon，emond）建立持久性。Emond 是一个守护进程（[Launch Daemon](https://attack.mitre.org/techniques/T1160)），它接受来自各种服务的事件，通过规则采取行动。在 `/sbin/emond` 处的 emond 程序将从 `/etc/emond.d/rules/` 中读取规则（plist格式），一旦有显式事件发生，就立即采取行动。如果路径 `/private/var/db/emondClients` 下没有文件存在 emond 服务将不会启动。


攻击者可以利用这个服务，编写一条规则，在定义的事件发生时执行恶意指定。当 emond 服务使用 root 权限执行时，攻击者就可以将特权从管理员升级到 root 权限。

#### 检测

- 通过检测 `/etc/emond.d/rules/` 和 `/private/var/db/emondClients` 目录下的文件创建或修改操作，来监视 emond 规则的创建。



### [T1068 - Exploitation for Privilege Escalation 漏洞利用](https://attack.mitre.org/techniques/T1068/)

> Tactic: Privilege Escalation
> Platform: Linux, macOS, Windows
> System Requirements: In the case of privilege escalation, the adversary likely already has user permissions on the target system.
> Permissions Required: User
> Effective Permissions: User, Administrator, SYSTEM/root
> Data Sources: Windows Error Reporting, Process monitoring, Application logs
> Version: 1.1

程序、服务、操作系统软件或内核本身可能存在漏洞，存在漏洞的程序可能以较高权限运行在操作系统中，攻击者可以利用这些漏洞来获得对系统更高级别的访问权限。



### [T1181 - Extra Window Memory Injection EWM 注入](https://attack.mitre.org/techniques/T1181/)

> Tactic: Defense Evasion, Privilege Escalation
> Platform: Windows
> Permissions Required: Administrator, SYSTEM
> Data Sources: API monitoring, Process monitoring
> Defense Bypassed: Anti-virus, Host intrusion prevention systems, Data Execution Prevention
> Version: 1.0

在创建窗口之前，基于图形化窗口的进程必须要注册一个 Windows 类，这时应用程序可以申请一小部分的额外内存空间（EWM），EWMI 的原理就是将恶意代码注入到资源管理器（Explorer）窗口的额外窗口内存中。



### [T1044 - File System Permissions Weakness 文件系统权限缺陷](https://attack.mitre.org/techniques/T1044/)

> Tactic: Persistence, Privilege Escalation
> Platform: Windows
> Permissions Required: Administrator, User
> Effective Permissions: SYSTEM, User, Administrator
> Data Sources: File monitoring, Services, Process command-line parameters
> CAPEC ID: [CAPEC-17](https://capec.mitre.org/data/definitions/17.html)
> Contributors: Stefan Kanthak; Travis Smith, Tripwire
> Version: 1.0

进程可以自动执行指定的二进制文件，如果不正确地设置了包含目标二进制文件的文件系统目录的权限或二进制文件本身的权限，则可能使用用户级权限用另一个二进制文件覆盖目标二进制文件，并由原始进程执行。如果原始进程和线程在更高的权限级别下运行，那么被替换的二进制文件也将在更高的权限级别下执行。

攻击者可以利用这个缺陷将正常的二进制文件替换为恶意的二进制文件，作为在更高权限级别执行代码的一种方式。如果将执行进程设置为在特定时间或特定事件期间运行（如：系统启动），则此技术也可用于持久性攻击。

主要利用方式有两种：

- Windows 服务：攻击者用恶意程序替换 windows 服务中正常的可执行文件。
- 可执行的安装程序：在安装过程中，安装程序通常使用 `%TEMP%` 目录中的子目录来解压二进制文件，当安装程序创建子目录和文件时，它们通常不会设置适当的权限来限制写访问，这就允许攻击者可以利用这种缺陷，在子目录中执行不受信任的代码，或者覆盖安装过程中使用的二进制文件。这种行为同时可能与 [DLL Search Order Hijacking](https://attack.mitre.org/techniques/T1038) 和 [Bypass User Account Control](https://attack.mitre.org/techniques/T1088) 相关。由于一些安装程序可能需要提升权限，这将导致攻击者植入的恶意代码也以更高权限执行。



### [T1179 - Hooking 钩子](https://attack.mitre.org/techniques/T1179/)

> Tactic: Persistence, Privilege Escalation, Credential Access
> Platform: Windows
> Permissions Required: Administrator, SYSTEM
> Data Sources: API monitoring, Binary file metadata, DLL monitoring, Loaded DLLs, Process monitoring, Windows event logs
> Version: 1.0

Windows 进程通常利用 API 函数来执行需要重用系统资源的任务，Hooking 的目的就是重定向调用这些功能，实现方式有以下几种：

- 过程钩子（hooks procedures），对于消息、击键和鼠标输入等事件进行拦截并执行指定的代码。
- IAT 钩子（import address table hooking），对进程的 IAT 进行修改，而 IAT 中存储了指向导入 API 函数的指针。
- 内联钩子（inline hooking）, 覆盖 API 函数中的第一个字节来重定向代码流。

与进程注入（[Process Injection](https://attack.mitre.org/techniques/T1055)）类似，攻击者可以 Hooking 在另一个进程的上下文中加载和隐蔽执行恶意代码，同时允许访问进程的内存，进行权限提升和持久性。

Hooking 通常被 [Rootkits](https://attack.mitre.org/techniques/T1014) 用来隐藏文件、进程、注册表项和其他对象，以隐藏恶意软件和其相关行为。



### [T1183 - Image File Execution Options Injection IFEO 注入](https://attack.mitre.org/techniques/T1183/)

> Tactic: Privilege Escalation, Persistence, Defense Evasion
> Platform: Windows
> Permissions Required: Administrator, SYSTEM
> Data Sources: Process monitoring, Windows Registry, Windows event logs
> Defense Bypassed: Autoruns Analysis
> Contributors: Oddvar Moe, @oddvarmoe
> Version: 1.0

映像文件执行选项（IFEO）使得开发人员能够将调试器 attach 到要调试的应用程序上，开发人员通过注册表设置 IFEOs 值，就可以将软件 attach 到一个要调试的程序上，之后只要一启动软件，被 attach 的程序也会一起启动。利用这种方式，攻击者可以修改此注册键值将恶意代码注入到目标软件中，当目标软件启动时，被注入的恶意代码就会一起启动，同时获得持久性和权限提升。



### [T1160 - Launch Daemon 守护进程](https://attack.mitre.org/techniques/T1160/)

> Tactic: Persistence, Privilege Escalation
> Platform: macOS
> Permissions Required: Administrator
> Effective Permissions: root
> Data Sources: Process monitoring, File monitoring
> Version: 1.0

根据苹果的官方文档，当 macOS 和 OS X 启动时，将运行 launchd 来完成系统初始化，这个过程会从 `/System/Library/LaunchDaemons` 和 `/Library/LaunchDaemons` 中的属性列表（plist）文件中为每个计划启动的系统级守护进程加载参数。

攻击者可以创建一个新的守护进程，使用 launchd 或 launchctl 将 plist 加载到特定目录，以在启动时执行这个守护进程，守护进程的名称可以伪装成系统进程或正常软件。由于守护进程可以使用管理员权限创建，在 root 权限下执行，因此攻击者可以将权限从管理员升级到 root。



### [T1050 - New Service 新服务](https://attack.mitre.org/techniques/T1050/)

> Tactic: Persistence, Privilege Escalation
> Platform: Windows
> Permissions Required: Administrator, SYSTEM
> Effective Permissions: SYSTEM
> Data Sources: Windows Registry, Process monitoring, Process command-line parameters, Windows event logs
> CAPEC ID: [CAPEC-550](https://capec.mitre.org/data/definitions/550.html)
> Contributors: Pedro Harrison
> Version: 1.0

当操作系统启动时，服务会在后台启动以执行系统功能。服务的配置信息，包括服务的可执行文件路径，都存储在 Windows 注册表中。

攻击者可以创建一个新的服务，通过使用程序与服务交互程序或直接修改注册表将其配置为在启动时执行，服务名称可以伪装成系统进程或正常软件来进行伪装（[Masquerading](https://attack.mitre.org/techniques/T1036)）。服务可以使用管理员权限创建，但在 SYSTEM 权限下执行，因此攻击者可以使用 Windows 服务将权限从管理员升级到 SYSTEM。攻击者也可以通过服务执行（[Service Execution](https://attack.mitre.org/techniques/T1035)）直接启动服务。



### [T1502 - Parent PID Spoofing 父进程标识符欺骗](https://attack.mitre.org/techniques/T1502/)

> Tactic: Defense Evasion, Privilege Escalation
> Platform: Windows
> Permissions Required: User, Administrator
> Data Sources: Windows event logs, Process monitoring, API monitoring
> Defense Bypassed: Host forensic analysis, Heuristic Detection
> Contributors: Wayne Silva, Countercept

新进程通常直接从父进程或调用进程派生，除非显式指定。显式分配新进程 PPID 的一种方法是通过调用 `CreateProcess API`，它支持定义要使用的 PPID 的参数。在系统（通常是通过`svchost.exe` 或 `consent.exe`）生成请求权限提升的进程后，由用户帐户控制（UAC）等 Windows 特性使用此功能来正确设置PPID。攻击者可以利用这个特性，伪造新进程的父进程标识符（PPID）来逃避监视和提高权限，或者利用管理员权限生成一个新进程，并将父进程分配为以 SYSTEM 权限运行的进程（如 lsass.exe），从而通过继承访问令牌的方式提升新进程的权限。




显式地分配 PPID 还可以启用特权升级(给予父进程适当的访问权限)。例如，特权用户上下文中的敌手(即管理员)可能生成一个新进程，并将父进程分配为作为系统运行的进程(如lsass.exe)，从而通过继承的访问令牌提升新进程



### [T1034 - Path Interception 路径拦截](https://attack.mitre.org/techniques/T1034/)

> Tactic: Persistence, Privilege Escalation
> Platform: Windows
> Permissions Required: User, Administrator, SYSTEM
> Effective Permissions: User, Administrator, SYSTEM
> Data Sources: File monitoring, Process monitoring
> CAPEC ID: [CAPEC-159](https://capec.mitre.org/data/definitions/159.html)
> Contributors: Stefan Kanthak

当可执行文件被放在特定的路径中，由其他应用程序而不是预期的程序执行时，就会发生路径拦截。比如，在一个有漏洞的应用程序的当前工作目录中放置 cmd 的副本，使该应用程序调用 `CreateProcess` 函数加载 cmd 程序或 BAT 文件。

在执行路径拦截时，攻击者可能会利用多个明显的漏洞或错误配置，比如：未引用的路径（Unquoted Paths）、路径环境变量错误配置（PATH Environment Variable Misconfiguration）和搜索顺序劫持（Search Order Hijacking）。



### [T1013 - Port Monitors 端口监控](https://attack.mitre.org/techniques/T1013/)

> Tactic: Persistence, Privilege Escalation
>
> Platform: Windows
>
> Permissions Required: Administrator, SYSTEM
>
> Effective Permissions: SYSTEM
>
> Data Sources: File monitoring, API monitoring, DLL monitoring, Windows Registry, Process monitoring
>
> Contributors: Stefan Kanthak; Travis Smith, Tripwire
>
> Version: 1.0

可以通过调用端口监视器的 API 来在启动时加载特定的DLL。这个 DLL 可以位于 `C:\Windows\System32` 下并将在系统引导时被 `spoolsv.exe`加载，此进程会在系统权限下运行。另外，如果权限允许将 DLL 的绝对路径写入`HKLM\SYSTEM\CurrentControlSet\Control\Print\`中，则可以加载任意 DLL。


该注册表项包含以下条目：

- 本地端口
- 标准的 TCP / IP 端口
- USB 监控
- WSD 端口

攻击者可以使用此技术在启动时加载恶意代码，这些恶意代码将在系统重新启动时持续存在，并以 SYSTEM 权限执行。



### [T1504 - PowerShell Profile 配置文件](https://attack.mitre.org/techniques/T1504/)

> Tactic: Persistence, Privilege Escalation
>
> Platform: Windows
>
> Permissions Required: User, Administrator
>
> Data Sources: Process monitoring, File monitoring, PowerShell logs
>
> Contributors: Allen DeRyke, ICE
>
> Version: 1.0

在某些情况下，攻击者可以通过利用 [PowerShell](https://attack.mitre.org/techniques/T1086) 的配置文件来获得持久性和提升权限。PowerShell 配置文件（`profile.ps1`）是 PowerShell 启动时运行的脚本，可以用作自定义用户环境的登录脚本。PowerShell 本身支持多个配置文件，具体取决于用户或主机程序。例如，PowerShell 控制台、PowerShell ISE 或 Visual Studio Code 都有不同的配置文件。

攻击者可以修改这些配置文件，使其包含任意命令、函数、模块和 PowerShell 驱动，以获得持久性。每次用户打开 PowerShell 会话时，除非在启动时使用 `-NoProfile` 参数，否则将执行修改后的脚本。

如果 PowerShell 配置文件中的脚本是由具有更高权限的帐户（例如域管理员）加载和执行的，则攻击者也可以获得权限提升。



### [T1055 - Process Injection 进程注入](https://attack.mitre.org/techniques/T1055/)

> Tactic: Defense Evasion, Privilege Escalation
>
> Platform: Linux, macOS, Windows
>
> Permissions Required: User, Administrator, SYSTEM, root
>
> Effective Permissions: User, Administrator, SYSTEM, root
>
> Data Sources: API monitoring, Windows Registry, File monitoring, DLL monitoring, Process monitoring, Named Pipes
>
> Defense Bypassed: Process whitelisting, Anti-virus
>
> CAPEC ID: [CAPEC-640](https://capec.mitre.org/data/definitions/640.html)
>
> Contributors: Anastasios Pingios; Christiaan Beek, @ChristiaanBeek; Ryan Becwar
>
> Version: 1.0

进程注入是在独立活动进程的地址空间中执行任意代码的方法，在另一个进程的上下文中运行恶意代码可能允许访问进程的内存、系统、网络资源，并提高权限。



### [T1053 - Scheduled Task 计划任务](https://attack.mitre.org/techniques/T1053/)

> Tactic: Execution, Persistence, Privilege Escalation
>
> Platform: Windows
>
> Permissions Required: Administrator, SYSTEM, User
>
> Effective Permissions: SYSTEM, Administrator, User
>
> Data Sources: File monitoring, Process monitoring, Process command-line parameters, Windows event logs
>
> Supports Remote:  Yes
>
> CAPEC ID: [CAPEC-557](https://capec.mitre.org/data/definitions/557.html)
>
> Contributors: Prashant Verma, Paladion; Leo Loobeek, @leoloobeek; Travis Smith, Tripwire; Alain Homewood, Insomnia Security
>
> Version: 1.1

诸如 [at](https://attack.mitre.org/software/S0110) 和 [schtasks](https://attack.mitre.org/software/S0111) 之类的实用工具，以及 Windows 任务调度程序（Windows Task Scheduler），可以用来自定义在某个日期和时间执行程序或脚本，也可以在远程系统上调度任务。条件是满足使用 RPC 的身份验证，并打开文件和打印机共享。在远程系统上调度任务通常需要成为远程系统上 Administrators 组的成员。


攻击者可以使用计划任务在系统启动时执行程序，以实现持久性和获得系统权限，或在指定帐户的上下文中运行进程。



### [T1058 - Service Registry Permissions Weakness 服务注册权限缺陷](https://attack.mitre.org/techniques/T1058/)

> Tactic: Persistence, Privilege Escalation
>
> Platform: Windows
>
> System Requirements: Ability to modify service values in the Registry
>
> Permissions Required: Administrator, SYSTEM
>
> Effective Permissions: SYSTEM
>
> Data Sources: Process command-line parameters, Services, Windows Registry
>
> CAPEC ID: [CAPEC-478](https://capec.mitre.org/data/definitions/478.html)
>
> Contributors: Matthew Demaske, Adaptforward; Travis Smith, Tripwire
>
> Version: 1.1

Windows 在 `HKLM\SYSTEM\CurrentControlSet\Services` 的注册表项中存储本地服务配置信息。可以通过服务控制器、sc.exe、PowerShell 或 Reg 等工具操作存储在服务注册表项下的信息，以修改服务的执行参数。


如果没有正确设置用户和组的权限并允许访问服务的注册表项，则攻击者可以更改服务的相关路径，使其指向攻击者控制下的其他可执行程序。当服务启动或重新启动时，由攻击者控制的程序就将执行，从而允许攻击者获得持久性和权限提升。



### [T1166 - Setuid and Setgid 标志位](https://attack.mitre.org/techniques/T1166/)

> Tactic: Privilege Escalation, Persistence
>
> Platform: Linux, macOS
>
> Permissions Required: User
>
> Effective Permissions: Administrator, root
>
> Data Sources: File monitoring, Process monitoring, Process command-line parameters
>
> Version: 1.0

当在 Linux 或 macOS 上为应用程序设置了 setuid（SUID） 或 setgid（SGID） 位时，应用程序将分别以拥有用户或组的特权运行。通常应用程序是在当前用户的上下文中运行的，而与拥有该应用程序的用户或组无关。任何用户都可以为自己的应用程序指定设置 setuid 或 setgid 标志，而无需在 sudoers 文件中创建一个必须由 root 用户完成的条目。

攻击者可以利用这一点来进行 shell 转义，或者利用应用程序中的 setsuid 或 setgid 位的漏洞来让代码在不同的用户上下文中运行。此外，攻击者可以在他们自己的恶意软件上使用这种机制，以确保他们能够在更高权限的上下文中执行。



### [T1178 - SID-History Injection](https://attack.mitre.org/techniques/T1178/)

>Tactic: Privilege Escalation                                        
>
>Platform: Windows
>
>Permissions Required: Administrator, SYSTEM
>
>Data Sources: API monitoring, Authentication logs, Windows event logs
>
>Contributors: Vincent Le Toux; Alain Homewood, Insomnia Security
>
>Version: 1.0

Windows 安全标识符（SID）是唯一标识用户或组帐户身份的值，在安全描述符和访问令牌中都使用了 SID。一个帐户可以在 SID-History Active Directory 属性中持有额外的 SID，从而允许域之间进行帐户迁移。


攻击者可以利用此机制进行权限提升，即使用域管理员（或等效的）权限，将获得的或已知的 SID 值插入到 SID-History 记录中，以启用对任意用户/组的身份的模拟。这种操作可以结合横向移动技术，以获得对本地资源和其他不可访问域的访问权限。



### [T1169 - Sudo 命令](https://attack.mitre.org/techniques/T1169/)

> Tactic: Privilege Escalation
>
> Platform: Linux, macOS
>
> Permissions Required: User
>
> Effective Permissions: root
>
> Data Sources: File monitoring
>
> Version: 1.0

sudoers 文件（`/etc/sudoers`）描述了哪些用户可以执行哪些命令以及从哪些终端执行这些命令，这还描述了哪些命令用户可以作为其他用户或组运行。这实现了最小权限的思想，可以保证用户在大多数时间都以他们可能的最低权限运行，并且在需要将权限提升到其他用户权限时，通常需要根据提示输入密码。但是 sudoers 文件也可以指定何时不需要提示用户输入密码，比如 `user1 ALL=(ALL) NOPASSWD: ALL`。

攻击者可以修改此文件使其作为其他用户执行命令，或者以更高的权限运行进程。



### [T1206 - Sudo Caching](https://attack.mitre.org/techniques/T1206/)

> Tactic: Privilege Escalation
>
> Platform: Linux, macOS
>
> Permissions Required: User
>
> Effective Permissions: root
>
> Data Sources: File monitoring, Process command-line parameters
>
> Version: 1.0

sudo命令允许系统管理员授权给某些用户（或用户组）以 root 用户或其他用户的身份运行某些命令，而 sudo 能够缓存一段时间的访问凭证，通过 timestamp_timeout 字段（时间记录在`/var/db/sudo`文件下）判定用户登录是否超时而重新提示输入密码。此外，还有一个tty_tickets变量，它独立地处理每个新的 tty（终端会话）。

攻击者可以利用这种糟糕的配置来提升权限，而不需要管理员的密码。可以监视 `/var/db/sudo` 的时间戳，看它是否在 timestamp_timeout 范围内，如果在范围内，则恶意软件可以直接执行 sudo 命令，而不需要提供用户的密码。当 tty_tickets 被禁用时，攻击者可以从任何 tty 执行此操作。



### [T1078 - Valid Accounts 有效账户](https://attack.mitre.org/techniques/T1078/)

> Tactic: Defense Evasion, Persistence, Privilege Escalation, Initial Access
>
> Platform: Linux, macOS, Windows, AWS, GCP, Azure, SaaS, Office 365
>
> Permissions Required: User, Administrator
>
> Effective Permissions: User, Administrator
>
> Data Sources: AWS CloudTrail logs, Stackdriver logs, Authentication logs, Process monitoring
>
> Defense Bypassed: Firewall,  Host intrusion prevention systems, Network intrusion detection system,  Process whitelisting, System access controls, Anti-virus
>
> CAPEC ID: [CAPEC-560](https://capec.mitre.org/data/definitions/560.html)
>
> Contributors: Netskope; Mark Wee; Praetorian
>
> Version: 2.0

攻击者可以使用凭据访问技术窃取特定用户或服务帐户的凭据，或者通过社会工程在侦察过程中更早地捕获凭据，以获得初始访问权限。

攻击者可能使用的帐户可以分为三类：默认帐户、本地帐户和域帐户。默认帐户是那些内置在操作系统中的帐户，如 Windows 系统上的 Guest 或管理员帐户。本地帐户是组织为用户、远程支持、服务或单个系统上的管理员而配置的帐户。域帐户是由 Active Directory 域服务管理的，可以包含用户、管理员和服务。

被利用的登录凭证可以用来绕过系统内各种资源的访问控制，甚至可以用于持久访问远程系统和外部服务，如 VPNs、Outlook Web access 和远程桌面服务。也可能授予更高的权限以访问特定的受限区域，同攻击者可能会选择不使用恶意软件或工具，只使用这些凭证进行合法的访问，使自己更难以被发现。



### [T1100 - Web Shell](https://attack.mitre.org/techniques/T1100/)

> Tactic: Persistence, Privilege Escalation
>
> Platform: Linux, Windows, macOS
>
> System Requirements: Adversary access to Web server with vulnerability or account to upload and serve the Web shell file.
>
> Effective Permissions: SYSTEM, User
>
> Data Sources: Anti-virus, Authentication logs, File monitoring, Netflow/Enclave netflow, Process monitoring
>
> CAPEC ID: [CAPEC-650](https://capec.mitre.org/data/definitions/650.html)
>
> Version: 1.0

Web shell 是放置在可公开访问的 Web 服务器上的 Web 脚本，允许攻击者将 Web 服务器用作进入网络的网关。Web shell 可以提供一组要执行的函数，或者在运行 Web 服务器的系统上提供一个命令行访问接口。除了服务器端脚本外，Web shell 可能还有一个用于连接 Web 服务器的客户端程序。

攻击者可以利用 Web shell 作为冗余访问（[Redundant Access](https://attack.mitre.org/techniques/T1108)）和持久性机制，以防万一攻击者的主要访问手段被发现和清除。

