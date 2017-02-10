Windows MSI installer build toolkit
================

This project creates a Salt Minion msi installer using [WiX][WiXId].

##Requirements##
- [Windows Installer][WindowsInstaller] 2.0, included since Windows XP. (Version probably changes)
-  VC++ 2008 Redistributable, included since Windows Server 2008 SP2/Windows Vista.
  - Due to the old age, I intentionally do not include it, see [Issue #18][issue18], comments welcome.
 
##Differences vs. NSIS (Nullsoft) installer##
The msi differs from the NSIS exe installer in:
- It allows installation to any directory. (TODO!)
- It supports unattended installation.
- By default, it leaves configuration, remove configuration with `KEEP_CONFIG=0`.

Additional benefits:
- A problem during the install causes the installation to be rolled back, as in a database transaction.
- Built-in logging (/l option to msiexec).
- The 32/64bit msi installer only run on the respective 32/64bit Windows.
- Both 32/64bit msi installers can be build on the same client. 

##Features##
- The msi detects and uninstalls an existing NSIS installation.


###On unattended install ("silent install")###

An msi allows you to install unattended ("silently"), meaning without opening any window, while still providing
customized values for e.g. master hostname, minion id, installation path, using the following command line:

> msiexec /i *.msi /qb! PROPERTY=VALUE PROPERTY=VALUE 


Available properties:

- `MASTER_HOSTNAME`: The master hostname. The default is `salt.
- `MINION_HOSTNAME`: The minion id. The default is `%COMPUTERNAME%`.
- `START_MINION_SERVICE`: Whether to start the salt-minion service after installation. The default is false.
- `KEEP_CONFIG`: keep c:\salt\conf. Default is `1` (true). Only from command line.
- `INSTALLFOLDER`: Where to install the files. Default is `c:\salt`. DO NOT CHANGE


##Build Requirement##

- Python 2.7 in `c:\python27`
- This project git clone in `c:\git\salt-windows-msi`
- Salt git clone in `c:\git\salt`
- The NSIS build in `c:\git\salt\pkg\windows`
- [WiX][WiXId] v3.10.
- [MSBuild 2015][MSBuild2015Id]  https://www.microsoft.com/en-in/download/confirmation.aspx?id=48159
- .Net 4.5
- (Probably not required: Visual Studio 2013 or 2015)


###Building###

yclean.cmd and ybuild.cmd are shortcuts for msbuild.
You CANNOT build the msi in Visual Studio.

The build will produce:
 - $(StagingDir)/wix/Salt-Minion-$(DisplayVersion)-$(TargetPlatform).msi

###<a id="msbuild"></a>MSBuild###

General command line:

> msbuild msbuild.proj \[/t:target[,target2,..]] \[/p:property=value [ .. /p:... ] ]

A 'help' target is available which prints out all the targets, customizable
properties, and the current value of those properties:

> msbuild msbuild.proj /t:help


###Components###

msbuild.proj imports common\targets\Minion.Common.targets


- common/targets/: MSBuild targets files used by the WiX projects.
  - BuildDistFragment.targets: contains msbuild targets to generate a WiX fragment from the extracted distribution.
  - DownloadVCRedist.targets: contains msbuild targets to download the appropriate Visual C++ redistributable for the WiX Bundle build.
  - Minion.Common.targets: contains targets to discover the correct distribution zip file, extract it and determine version.
- wix.sln: Visual Studio solution file. Requires VS2010 or above. See note above about dependencies.
- wix/MinionConfigurationExtension/: A WiX Extension implementing custom actions for configuration manipulation.
- wix/MinionMSI/: This is the WiX .msi project
  - dist-$(TargetPlatform).wxs: WiX fragment describing the contents of the distribution zip file. 
    This is autogenerated and added to the compile at build time; it does not show up in the Visual Studio solution.
  - SettingsCustomizationDlg.wxs: A custom MSI dialog for the master/minion id properties.
  - MinionMSI.wixproj: the main project file.
  - MinionConfigurationExtensionCA.wxs: A WiX fragment setting up the configuration manipulator custom actions.
  - Product.wxs: contains the main MSI description and event sequence
  - service.wxs: contains a WiX component for nssm.exe and the associated Windows Service description/control settings.
    - wix\MinionMSI\dist-amd64.wxs lists all the discovered sources.
    - Because nssm.exe must be a (handwritten) WiX component in service.wxs, it also must be excluded from dist-amd64.wxs. 
    - BuildDistFragment.xsl excludes nssm.exe from dist-amd64.wxs.
  - WixUI\_Minion.wxs: WiX fragment describing the UI for the setup.
  - Banner.jpg: Used as the top bar banner in most of the UI dialogs.
  - Dialog.jpg: Used as the dialog background for Welcome and Exit dialogs.
- wix/MinionEXE/: This is the WiX bundle .exe project (TO BE REMOVED)
  - Bundle.wxs: contains the bundle description and package chain


###Extending###

Additional configuration manipulations may be able to use the existing
MinionConfigurationExtension project. Current manipulations read the
value of a particular property (e.g. MASTER\_HOSTNAME) and apply to the
existing configuration file using a regular expression replace. Each new
manipulation will require changes to the following files:

- MinionConfiguration.cs: a new method (i.e. new custom action).
- MinionConfigurationExtensionCA.wxs: a &lt;CustomAction /&gt; entry to
  make the new method available.
- Product.wxs: a &lt;Custom /&gt; entry in the &lt;InstallSequence /&gt;
  to make the configuration change.
- Product.wxs: a &lt;Property /&gt; entry containing a default for the
  manipulated configuration setting.
- README.md: alter this file to explain the new configuration option and
  log the property name.

If the new custom action should be exposed to the UI, additional changes
are required:

- SettingsCustomizatonDlg.wxs: There is room to add 1-2 more properties
  to this dialog.
- WixUI\_Minion.wxs: A &lt;ProgressText /&gt; entry providing a brief
  description of what the new action is doing.

If the new custom action requires its own dialog, these additional
changes are required:

- The new dialog file.
- WixUI\_Minion.wxs: &lt;Publish /&gt; entries hooking up the dialog
  buttons to other dialogs. Other dialogs will also have to be adjusted
  to maintain correct sequencing.
- MinionMSI.wixproj: The new dialog must be added as a &lt;Compile /&gt;
  item to be included in the build.

##On versioning##
The user sees a [3-tuple][version_html] version, e.g. `2016.11.3`.

msi rules demand that numbers must be smaller than 256, therefore only the "short year" is used.
e.g. `16.11.3.77`

The msi properties `DisplayVersion` and `InternalVersion` store these values.

[Internally][version_py], version is a 8-tuple.


[WiXId]: http://wixtoolset.org "WiX Homepage"
[MSBuildId]: http://msdn.microsoft.com/en-us/library/0k6kkbsd(v=vs.120).aspx "MSBuild Reference"
[MSBuild2015Id]: https://www.microsoft.com/en-in/download/details.aspx?id=48159
[version_html]: https://docs.saltstack.com/en/latest/topics/releases/version_numbers.html
[version_py]: https://github.com/saltstack/salt/blob/develop/salt/version.py
[WindowsInstaller]:https://en.wikipedia.org/wiki/Windows_Installer#Versions
[issue18]:https://github.com/markuskramerIgitt/salt-windows-msi/issues/18
