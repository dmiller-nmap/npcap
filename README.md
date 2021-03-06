Npcap
==========
http://www.npcap.org

[![Build status](https://ci.appveyor.com/api/projects/status/01yoks5rn14wgny2?svg=true)](https://ci.appveyor.com/project/hsluoyz/npcap)
![Environment](https://img.shields.io/badge/Windows-Vista, 7, 8, 10-yellow.svg)
![Release](https://img.shields.io/github/release/nmap/npcap.svg)
![License](https://img.shields.io/github/license/nmap/npcap.svg)
![Downloads](https://img.shields.io/github/downloads/nmap/npcap/latest/total.svg)
![TotalDownloads](https://img.shields.io/github/downloads/nmap/npcap/total.svg)

Npcap is an update of [**WinPcap**](http://www.winpcap.org/) to [**NDIS 6 Light-Weight Filter (LWF)**](https://msdn.microsoft.com/en-us/library/windows/hardware/ff565492(v=vs.85).aspx) technique. It supports **Windows Vista, 7, 8 and 10**. It is sponsored by the [**Nmap Project**](http://nmap.org/) and developed by [**Yang Luo**](http://www.veotax.com/) under [**Google Summer of Code 2013**](https://www.google-melange.com/gsoc/project/details/google/gsoc2013/hsluoyz/5727390428823552) and [**2015**](https://www.google-melange.com/gsoc/project/details/google/gsoc2015/hsluoyz/5723971634855936). It also received many helpful tests from [**Wireshark**](https://www.wireshark.org/) and [**NetScanTools**](http://www.netscantools.com/).

## Features

1. **NDIS 6 Support**: Npcap makes use of new LWF driver in Windows Vista and later (the legacy driver is used on XP). It's faster than the legacy [**NDIS 5 Intermediate**](https://msdn.microsoft.com/en-us/library/windows/hardware/ff557012(v=vs.85).aspx) technique. One reason is that packet data stucture has changed (from ``NDIS_PACKET`` to ``NET_BUFFER_LIST``) since Vista and NDIS 5 needs to handle extra packet structure conversion.
2. **"Admin-only Mode" Support**: Npcap supports to restrict its use to Administrators for safety purpose. If Npcap is installed with the option **Restrict Npcap driver's access to Administrators only** checked, when a non-Admin user tries to start a user software (Nmap, Wireshark, etc), the [**User Account Control (UAC)**](http://windows.microsoft.com/en-us/windows/what-is-user-account-control#1TC=windows-7) dialog will prompt asking for Administrator privilege. Only when the end user chooses ``Yes``, the driver can be accessed. This is similar to UNIX where you need root access to capture packets.
3. **"WinPcap Compatible Mode" Support**: "WinPcap Compatible Mode" is used to decide whether Npcap should coexist With WinPcap or be compatible with WinPcap. With "WinPcap Compatible Mode" ``OFF``, Npcap can coexist with WinPcap and share the DLL binary interface with WinPcap. So the applications unaware of Npcap **SHOULD** be able to use Npcap automatically if WinPcap is unavailable. The applications who knows Npcap's existence can choose to use Npcap or WinPcap first. The key about which is loaded first is [**DLL Search Path**](https://msdn.microsoft.com/en-us/library/windows/desktop/ms682586(v=vs.85).aspx). With "WinPcap Compatible Mode" ``OFF``, Npcap installs its DLLs into ``C:\Windows\System32\Npcap\`` instead of WinPcap's ``C:\Windows\System32\``. So applications who want to load Npcap first must make ``C:\Windows\System32\Npcap\`` precedent to other paths in ways such as calling [**SetDllDirectory**](https://msdn.microsoft.com/en-us/library/ms686203.aspx), etc. Another point is Npcap uses service name ``npcap`` instead of WinPcap's ``npf`` with "WinPcap Compatible Mode" ``OFF``. So applications using ``net start npf`` for starting service must use ``net start npcap`` instead. If you want 100% compatibility with WinPcap, you should install Npcap choosing "WinPcap Compatible Mode" (Install Npcap in WinPcap API-compatible Mode). In this mode, Npcap will install its Dlls in WinPcap's ``C:\Windows\System32\`` and use the ``npf`` service name. It's notable that before installing in this mode, you must uninstall WinPcap first (the installer wizard will prompt you that).
4. **Loopback Packets Capture Support**: Now Npcap is able to see Windows loopback packets using [**Windows Filtering Platform (WFP)**](https://msdn.microsoft.com/en-us/library/windows/desktop/aa366510(v=vs.85).aspx) technique. After installation, Npcap will create an adapter named ``Npcap Loopback Adapter`` for you. If you are a Wireshark user, choose this adapter to capture, you will see all loopback traffic the same way as other non-loopback adapters. Try it by typing in commands like ``ping 127.0.0.1`` (IPv4) or ``ping ::1`` (IPv6).
5. **Loopback Packets Send Support**: Besides loopback packets capturing, Npcap can also send out loopback packets based on [**Winsock Kernel (WSK)**](https://msdn.microsoft.com/en-us/library/windows/hardware/ff556958(v=vs.85).aspx) technique. A user software (e.g. Nmap) can just send packets out using ``Npcap Loopback Adapter`` like other adapters. ``Npcap Loopback Adapter`` will automatically remove the packet's Ethernet header and inject the payload into Windows TCP/IP stack, so this kind of loopback packet never go out of the machine.
6. **Raw 802.11 Packets Capture Support**: Npcap is able to see **802.11** packets instead of **fake Ethernet** packets on ordinary wireless adapters. You need to select the ``Support raw 802.11 traffic (and monitor mode) for wireless adapters`` option in the installation wizard to enable this feature. When your adapter is in ``Monitor Mode``, Npcap will supply all ``802.11 data + control + management`` packets with ``radiotap`` headers. When your adapter is in ``Managed Mode``, Npcap will only supply ``802.11 data`` packets with ``radiotap`` headers. Moreover, Npcap provides the ``WlanHelper.exe`` tool to help you switch to ``Monitor Mode`` on Windows. See more details about this feature in section ``For softwares that use Npcap raw 802.11 feature``. See more details about ``radiotap`` here: http://www.radiotap.org/

## Architecture

Npcap tries to **keep the original WinPcap architecture as much as possible**. As the table shows, you will find it very similar with WinPcap.
```
File                     Src Directory            Description
wpcap.dll                wpcap                    the libpcap API, added "loopback support" to original WinPcap
Packet.dll               packetWin7\Dll           the Packet API for Windows, added "Admin-only Mode" to original WinPcap
npf.sys (or npcap.sys)   packetWin7\npf           the driver, ported from NDIS 5 to NDIS 6, we support two names: npf or npcap, based on whether Npcap is installed in "WinPcap Compatible Mode"
NPFInstall.exe           packetWin7\NPFInstall    a LWF & WFP driver installation tool we added to Npcap
NPcapHelper.exe          packetWin7\Helper        the helper program for "Admin-only Mode", will run under Administrator rights
WlanHelper.exe           packetWin7\WlanHelper    a tool is used to set/get the operation mode (like monitor mode) for a wireless adapter
```

## For softwares that use Npcap loopback feature

Npcap's loopback adapter device is based on ``Microsoft KM-TEST Loopback Adapter`` (Win8 and Win10) or ``Microsoft Loopback Adapter`` (Vista, Win7). It is an Ethernet adapter, and Npcap has changed its behavior and rename it to ``Npcap Loopback Adapter``, to make it see the real loopback traffic only. The traffic captured by original WinPcap will not appear here. 

The IP address of ``Npcap Loopback Adapter`` is usually like ``169.254.x.x``. However, this IP is totally meaningless. Softwares using Npcap should regard this interface's IP address as ``127.0.0.1`` (IPv4) and ``::1`` (IPv6). This work can't be done by Npcap because Windows forbids any IP address to be configured as ``127.0.0.1`` or ``::1`` as they're reserved.

The MAC address of ``Npcap Loopback Adapter`` is usually like ``02:00:4C:4F:4F:50``. However, this address is meaningless too. Softwares using Npcap should think this interface doesn't own a MAC address, as the loopback traffic never goes to link layer. For softwares using Npcap to capture loopback traffic, the MAC addresses in captured data will be all zeros (aka ``00:00:00:00:00:00``). For softwares using Npcap to send loopback traffic, any MAC addresses can be specified as they will be ignored. But notice that ``ether_type`` in Ethernet header should be set correctly. Only ``IPv4`` and ``IPv6`` are accepted. Other values like ``ARP`` will be ignored. (You don't need an ARP request for loopback interface)

The MTU of ``Npcap Loopback Adapte`` is hard-coded to ``65536`` by Npcap. Softwares using Npcap should get this value automatically and no special handling is needed. This value is determined personally by me and doesn't mean Windows loopback stack can only support packet size as large as ``65536``. So don't feel weird if you have captured packets whose size are larger than it.

Don't try to make OID requests to ``Npcap Loopback Adapter`` except ``OID_GEN_MAXIMUM_TOTAL_SIZE`` (MTU). Those requests will still succeed like other adapters do, but they only make sense for NDIS adapters and Npcap doesn't even use the NDIS way to handle the loopback traffic. The only handled OID request by Npcap is ``OID_GEN_MAXIMUM_TOTAL_SIZE``. If you query its value, you will always get ``65550`` (65536 + 14). If you try to set its value, the operation will always fail.

To conclude, a software that wants to support Npcap loopback feature should do these steps:

* Detect ``Npcap Loopback Adapter``'s presence, by reading registry value ``Loopback`` at key ``HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\npf`` (or ``npcap`` if you installed Npcap With "WinPcap Compatible Mode" ``OFF``). If ``Loopback`` value exsits, it means ``Npcap Loopback Adapter`` is OK. Then perform the following steps.
* Treat the IP address of ``Npcap Loopback Adapter`` as ``127.0.0.1`` (IPv4) and ``::1`` (IPv6).
* Treat the MAC address of ``Npcap Loopback Adapter`` as ``00:00:00:00:00:00``.
* If you use [**IP Helper API**](https://msdn.microsoft.com/en-us/library/aa366073.aspx) to get adapter list, you will get an interface named like ``Loopback Pseudo-Interface 1``. This interface is a **DUMMY** interface by Microsoft and can't be seen in NDIS layer. And tt also takes the ``127.0.0.1``/``::1`` IP address. A good practise for softwares is merging the entry of ``Npcap Loopback Adapter`` and the entry of ``Loopback Pseudo-Interface 1`` into one entry, like what I have implemented for Nmap (see the ``Other code (for developers)`` part).
* Don't make use of OID requests for ``Npcap Loopback Adapter`` except ``OID_GEN_MAXIMUM_TOTAL_SIZE`` requests.

## For softwares that use Npcap raw 802.11 feature

### Steps

1. Install the latest version Npcap with the ``Support raw 802.11 traffic (and monitor mode) for wireless adapters`` option checked in the installation wizard. With this option checked, Npcap will see packets with Radiotap + ``802.11`` headers for wireless adapters. Otherwise, Npcap will see packets with ``fake Ethernet`` headers for wireless adapters.

2. Run ``WlanHelper.exe`` with **Administrator** privilege. Type in the index of your wireless adapter (usually ``0``) and press ``Enter``. Then type in ``1`` and press ``Enter`` to  to switch on the **Monitor Mode**. ``WlanHelper.exe`` also supports parameters to be used in an API manner, run ``WlanHelper.exe -h`` for details.

3. An example: launch ``Wireshark`` and capture on the wireless adapter, you will see **all 802.11 packets (data + control + management)**. Here you should make your software interact with Npcap using the WinPcap API (open the adapter, read packets, send packets, etc).

4. If you need to return to **Managed Mode**, run ``WlanHelper.exe`` again and input the index of the adapter, then type in ``0`` and press ``Enter`` to  to switch off the **Monitor Mode**.

### Tips

* You need to use ``WlanHelper.exe`` tool to switch on the **Monitor Mode** in order to see ``802.11 control and management packets`` in Wireshark (also ``encrypted 802.11 data packets``, you need to specify the ``decipher key`` in Wireshark in order to decrypt those packets), otherwise you will only see ``802.11 data packets``.

* Switching on the **Monitor Mode** will disconnect your wireless network from the AP, you can switch back to **Managed Mode** using the same ``WlanHelper.exe`` tool.

* The ``WlanHelper.exe`` tool automatically installed to your system path after installing Npcap.

### Terminology

**Managed Mode** (for ``Linux``) = **Extensible Station Mode** (aka **ExtSTA**, for ``Windows``)

**Monitor Mode** (for ``Linux``) = **Network Monitor Mode** (aka **NetMon**, for ``Windows``)

### WlanHelper

WlanHelper is used to set/get the operation mode (like **Monitor Mode**) for a wireless adapter on Windows. WlanHelper tries to follow the grammar of [iwconfig](http://linux.die.net/man/8/iwconfig), a wireless management tool for Linux. So if you rename ``WlanHelper.exe`` to ``iwconfig.exe``, your command lines for WlanHelper will be exactly the same with the ``iwconfig`` tool.

#### WlanHelper's Usage

##### Interactive way:

Run ``WlanHelper`` without parameters.

##### Command-line API way:

1. Run ``netsh wlan show interfaces``, get the ``GUID`` for the interface.

2. Run ``WlanHelper -h`` to see the usage. ``{Interface Name}`` refers to the ``GUID`` in the above step.

```
Command Usage:
WlanHelper {Interface Name} mode [*null*|managed|monitor]
*null* - get interface mode
managed - set interface mode to managed mode (aka ExtSTA)
monitor - set interface mode to monitor mode (aka NetMon)
```

An example:

```
C:\>netsh wlan show interfaces

There is 1 interface on the system:

    Name                   : Wi-Fi
    Description            : Qualcomm Atheros AR9485WB-EG Wireless Network Adapter
    GUID                   : 42dfd47a-2764-43ac-b58e-3df569c447da
    Physical address       : a4:db:30:d9:3a:9a
    State                  : connected
    SSID                   : LUO-PC_Network
    BSSID                  : d8:15:0d:72:8c:18
    Network type           : Infrastructure
    Radio type             : 802.11n
    Authentication         : WPA2-Personal
    Cipher                 : CCMP
    Connection mode        : Auto Connect
    Channel                : 1
    Receive rate (Mbps)    : 150
    Transmit rate (Mbps)   : 150
    Signal                 : 100%
    Profile                : LUO-PC_Network

    Hosted network status  : Not available

C:\>WlanHelper.exe 42dfd47a-2764-43ac-b58e-3df569c447da mode
managed
C:\>WlanHelper.exe 42dfd47a-2764-43ac-b58e-3df569c447da mode monitor
Success
C:\>WlanHelper.exe 42dfd47a-2764-43ac-b58e-3df569c447da mode 
monitor
C:\>WlanHelper.exe 42dfd47a-2764-43ac-b58e-3df569c447da mode managed
Success
C:\>WlanHelper.exe 42dfd47a-2764-43ac-b58e-3df569c447da mode
managed
C:\>WlanHelper.exe 42dfd47a-2764-43ac-b58e-3df569c447da mode master
Error: invalid parameter, type in "WlanHelper -h" for help.
```

## Build

Run ``installer\Build.bat``: build all DLLs and the driver. The DLLs need to be built using **Visual Studio 2013**. And the driver needs to be built using **Visual Studio 2015** with **Windows SDK 10 10586** & **Windows Driver Kit 10 10586**.

## Packaging

Run ``installer\Deploy.bat``: copy the files from build directories to deployment directories and sign the files. Generate an installer named ``npcap-nmap-%VERSION%.exe`` using [NSIS large strings build](http://nsis.sourceforge.net/Special_Builds) with the [SysRestore plug-in (special build for Npcap)](https://github.com/hsluoyz/SysRestore) and sign the installer.

## Generating debug symbols (optional)

Run ``installer\Deploy_Symbols.bat``: copy the debug symbol files (.PDB) from build directories to deployment directories and package them into a zip file named ``npcap-nmap-<VERSION>-DebugSymbols.zip`` using [7-Zip](http://www.7-zip.org/).

## Redistribution

(You need to first notice our [LICENSE](https://github.com/nmap/npcap/blob/master/LICENSE) before distributing Npcap)

The Npcap installer is friendly for redistribution by supporting two installation ways: ``GUI Mode`` (direct run) and ``Silent Mode`` (run with ``/s`` paramter).

### Installation options

The current Npcap installation options by default are (for both ``GUI Mode`` and ``Silent Mode``):

``/admin_only=no /loopback_support=yes /dlt_null=no /dot11_support=no /vlan_support=no /winpcap_mode=yes``

1. ``/admin_only=yes``: Restrict Npcap driver's access to Administrators only
2. ``/loopback_support=yes``: Support **loopback** traffic (``Npcap Loopback Adapter`` will be created)
3. ``/dlt_null=yes``: Use ``DLT_NULL`` as the loopback interface' link layer protocol instead of ``DLT_EN10MB``
4. ``/dot11_support=yes``: Support **raw 802.11** traffic (and **Monitor Mode**) for wireless adapters
5. ``/vlan_support=yes``: Support **802.1Q VLAN** tag when capturing and sending data
6. ``/winpcap_mode=yes``: Install Npcap in ``WinPcap API-compatible Mode``

### How to change default options for ``GUI Mode`` installation

Default options for Npcap installer GUI can be changed. An example is:

``npcap-nmap-0.07.exe /admin_only=no /loopback_support=yes /dlt_null=no /dot11_support=no /vlan_support=no /winpcap_mode=no``

or even simpler:

``npcap-nmap-0.07.exe /winpcap_mode=no``

As the default option of ``/winpcap_mode`` is ``yes``. Running the installer directly without options will see ``Install Npcap in WinPcap API-compatible Mode`` **CHECKED** by default in the ``Installation Options`` page. However, the above two commands will launch the installer GUI, and in the ``Installation Options`` page, the ``Install Npcap in WinPcap API-compatible Mode`` option will be **UNCHECKED** by default.

### How to change options for ``Silent Mode`` installation

An example of changing option feature for silent installation is:

``npcap-nmap-0.07.exe /S /admin_only=no /loopback_support=yes /dlt_null=no /dot11_support=no /vlan_support=no /winpcap_mode=yes``

1. The above example shows the default value. e.g., if you doesn't specify the key ``/admin_only``, it will take the default value ``no``. This is the same with the GUI.

2. The keys are **case-insensitive**.

3. The values are **case-sensitive**, only two values are permitted: ``yes`` or ``no``.

## Downloads & Run (for users)

1. Download and install the latest Npcap installer: https://github.com/nmap/npcap/releases
2. Use ``Nmap`` or ``Wireshark`` to test Npcap.

## Other code (for developers)

#### Previous installers before Npcap 0.05:
https://svn.nmap.org/nmap-exp/yang/NPcap-LWF/npcap_history_versions/

#### The changes of Nmap to use Npcap (**NOT** in ``WinPcap Compatible Mode``):
https://github.com/hsluoyz/nmap/

#### The changes of Nmap to use Npcap's loopback feature:
https://svn.nmap.org/nmap-exp/yang/nmap-npcap/

#### The compiled Nmap binaries after above changes:
https://svn.nmap.org/nmap-exp/yang/nmap-npcap_compiled_binaries/

## The list of incompatible softwares
The LWF technique Npcap used is usually a cause of conflicts between the programs using network drivers. The incompatible result can be **bluescreen**, **not see traffic**, **not see adapter**, etc. The reason can **either be Npcap's bug or the incompatible software's bug**. If the latter is the case, there's nothing much can be done by Npcap but for youself to remove the incompatible software.

The commonly seen sources of the incompatble softwares are ``anti-virus``, ``network firewall``, ``VPN``, ``traffic capture``, etc. To help you identify those softwares, We have collected the programs that have trouble with Npcap according to user's report, so what you need to do is to remove them if your Npcap doesn't work normally.

- Avaya Collaboration Services
- Avaya Collaboration Services x64
- Avaya UCA Type Library

## License

See: [LICENSE](https://github.com/nmap/npcap/blob/master/LICENSE)

## Contact

* ``dev@nmap.org`` (Nmap development list, this is **preferred**)
* ``hsluoyz@gmail.com`` (Yang Luo's email, if your issue needs to be kept private, please contact me via this mail)
