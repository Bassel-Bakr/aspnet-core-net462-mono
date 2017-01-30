# aspnet-core-net462-mono
Example of getting an ASP.NET Core 1.1.0 app using the full .NET Framework 4.6.2 
with the Mono runtime on *nix systems.

Currently, the full .NET Framework is not supported on non-windows systems.
However, there are workarounds available to get the Mono runtime to work with ASP.NET Core apps.

Related issues:

* <https://github.com/aspnet/Home/issues/1610>
* <https://github.com/aspnet/KestrelHttpServer/issues/942>
* <https://github.com/aspnet/KestrelHttpServer/issues/945>
* <https://github.com/aspnet/KestrelHttpServer/issues/956>
* <https://github.com/aspnet/KestrelHttpServer/issues/963>
* <https://github.com/aspnet/Mvc/issues/4818>
* <https://github.com/aspnet/Mvc/issues/5088>
* <https://github.com/dotnet/cli/issues/3835>
* <https://github.com/dotnet/corefx/issues/9012>
* <https://github.com/dotnet/sdk/issues/335>

There appears to be two main issues:

1. The Reference Assemblies are not in a place that the .NET Core tooling and libraries expect.
1. Kestrel tries to call a native function that is not supported by mono. (`System.Native.GetUnixName()`)

For the 1st problem, there are actually two manifestations. One at build-time and the other at run-time.

This is an example of the build-time error:

```
$ dotnet restore -r osx
$ dotnet build -r osx
Microsoft (R) Build Engine version 15.1.539.38876
Copyright (C) Microsoft Corporation. All rights reserved.

/usr/local/share/dotnet/sdk/1.0.0-rc4-004689/Microsoft.Common.CurrentVersion.targets(1111,5): error MSB3644: The reference assemblies for framework ".NETFramework,Version=v4.6.2" were not found. To resolve this, install the SDK or Targeting Pack for this framework version or retarget your application to a version of the framework for which you have the SDK or Targeting Pack installed. Note that assemblies will be resolved from the Global Assembly Cache (GAC) and will be used in place of reference assemblies. Therefore your assembly may not be correctly targeted for the framework you intend. [/Users/xxx/src/dotnet/aspnet-core-net462-mono/Web/Web.csproj]
/usr/local/share/dotnet/sdk/1.0.0-rc4-004689/Sdks/Microsoft.NET.Sdk/build/Microsoft.NET.Sdk.targets(76,5): error : Cannot find project info for '/Users/xxx/src/dotnet/aspnet-core-net462-mono/Core/Core.csproj'. This can indicate a missing project reference. [/Users/xxx/src/dotnet/aspnet-core-net462-mono/Web/Web.csproj]
```

A workaround is described here: <https://github.com/dotnet/sdk/issues/335#issuecomment-271186591>.
This workaround depends on a private nuget package described here: <https://github.com/dotnet/sdk/issues/335#issuecomment-257457331>.

This `NuGet.config` file is required to register the private dotnet-core feed:

```
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="dotnet-core" value="https://dotnet.myget.org/F/dotnet-core/api/v3/index.json"/>
  </packageSources>
</configuration>
```

I added this to my `.csproj` files:

```
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    ...
    <RuntimeIdentifier>win7-x64</RuntimeIdentifier>
    <FrameworkPathOverride>$(NuGetPackageFolders)Microsoft.TargetingPack.NETFramework.v4.6.2\1.0.1\lib\net462\</FrameworkPathOverride>
  </PropertyGroup>

  <ItemGroup>
    ...
    <PackageReference Include="Microsoft.TargetingPack.NETFramework.v4.6.2" Version="1.0.1" ExcludeAssets="All" PrivateAssets="All" />
  </ItemGroup>
</Project>

```

With the build out of the way, we run into the 1st run-time issue:

```
$ mono --debug bin/Debug/net462/Web.exe

Unhandled Exception:
System.DllNotFoundException: System.Native
  at (wrapper managed-to-native) Interop+Sys:GetUnixNamePrivate ()
  at Interop+Sys.GetUnixName () [0x00000] in <2c0705c248b844f597694acdb70b3a23>:0
  at System.Runtime.InteropServices.RuntimeInformation.IsOSPlatform (System.Runtime.InteropServices.OSPlatform osPlatform) [0x00009] in <2c0705c248b844f597694acdb70b3a23>:0
...
--- End of stack trace from previous location where exception was thrown ---
  at Microsoft.AspNetCore.Hosting.Internal.WebHost.Initialize () [0x00008] in <24af31b64ae843689736582353a19b3a>:0
  at Microsoft.AspNetCore.Hosting.WebHostBuilder.Build () [0x0008a] in <24af31b64ae843689736582353a19b3a>:0
  at MvcApp.Program.Main (System.String[] args) [0x00001] in /Users/xxx/src/dotnet/aspnet-core-net462-mono/Web/Program.cs:14
```

One workaround is to build your own version of `System.Native` that has the required function.
[This](https://github.com/borgdylan/corefx) is a fork of `System.Native` from `dotnet/corefx` that will work with Mono.
[This](https://github.com/Tragetaschen/libSystem.Native) is a version of `System.Native` that is a very lightweight solution: 
it only has the required `GetUnixName` function.

My workaround was to use the `System.Native` library from the dotnet tooling. However, the provided library only works with 64-bits:

```
ln -s /usr/local/share/dotnet/shared/Microsoft.NETCore.App/1.1.0/System.Native.dylib bin/Debug/net462/libSystem.Native.dylib
mono --arch=64 --debug bin/Debug/net462/Web.exe
```

The last issue is related to the reference assemblies again. 
This time, Razor dynamic compilation requires them:

```
$ mono --arch=64 --debug bin/Debug/net462/Web.exe
Hosting environment: Production
Content root path: /Users/xxx/src/dotnet/aspnet-core-net462-mono/Web
Now listening on: http://localhost:5000
Application started. Press Ctrl+C to shut down.
fail: Microsoft.AspNetCore.Diagnostics.ExceptionHandlerMiddleware[0]
      An unhandled exception has occurred: Can not find reference assembly 'Microsoft.CSharp.dll' file for package microsoft.csharp
System.InvalidOperationException: Can not find reference assembly 'Microsoft.CSharp.dll' file for package microsoft.csharp
  at Microsoft.Extensions.DependencyModel.Resolution.ReferenceAssemblyPathResolver.TryResolveAssemblyPaths (Microsoft.Extensions.DependencyModel.CompilationLibrary library, System.Collections.Generic.List`1[T] assemblies) [0x00046] in <e03789ee78e447efafd895cff5f6d194>:0
  at Microsoft.Extensions.DependencyModel.Resolution.CompositeCompilationAssemblyResolver.TryResolveAssemblyPaths (Microsoft.Extensions.DependencyModel.CompilationLibrary library, System.Collections.Generic.List`1[T] assemblies) [0x0000b] in <e03789ee78e447efafd895cff5f6d194>:0
...
  at Microsoft.AspNetCore.Diagnostics.ExceptionHandlerMiddleware+<Invoke>d__6.MoveNext () [0x00080] in <54abb6e5c34d41e5b18eb3ce35092c5b>:0
```

The `ReferenceAssemblyPathResolver` will actually use the `DOTNET_REFERENCE_ASSEMBLIES_PATH` environment variable, so we just need to do this:

```
export DOTNET_REFERENCE_ASSEMBLIES_PATH=/Library/Frameworks/Mono.framework/Versions/Current/lib/mono/4.5
```


