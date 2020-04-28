---
layout: post
title: Modern Unity
categories:
- Unity
feature_image: "https://picsum.photos/2560/600?image=1060"
---

# Using modern .Net tools with Unity

I have been using Unity for two years now, and during that time I have also developed a few small applications as side projects using various .Net technologies(WPF, ASP.NET, Blazor) and all the time some things were bugging me with Unity. Something did not feel right, and it's hard to grasp at first. On one hand Unity is a great tool for making games. It has numerous tools for artists, an easy to learn UI, a comprehensive API and just generally a lot off good things. On the other hand lies the dirty parts of the .Net tooling Unity uses:

- Mono (not the latest official version)
- Project files (csproj, sln) are auto generated
- No nuget support
- Everything needs a .meta file
- Different scripting backends
- Platform specific problems (WebGL, Android)
- No main method
- Poor async support

These are not necessarily deal breaking problems (after all a lot of devs are using Unity and the company itself has been around for quite a few years now) but just generally makes life a lot harder than it should. What actual limiting problems does Unity have?

- No global startup might lead to hacky static or singleton classes
- A lot of the Unity API does not support being called from other threads (even basic things like random number generation) which leads to most people not even bothering to try using asynchronous programming.
- You can download any dependency you want from nuget and just put the dll into your assets folder, but then you have to version control dlls. Which might get out of hand, which you might try to solve with git lfs but then again that can also turn into a nightmare if you don't handle them well
- No no built in support for dependency injection
- No familiar tools for other .Net developers

The last point is which is the real problem in my opinion. Despite being run on .Net the Unity ecosystem (ecosystem in another ecosystem) has moved a bit too much away from how we develop usual .Net applications. You want to add a new project? No dotnet new, you MUST go through UI or create some in-house tool to generate project templates. YOU CAN'T JUST ADD A NUGET PACKAGE. A logically separate part of your application logic needs to be placed in a project where do you put it? No not under a csproj file, under an asmdef file. You want to use ILogger<>, Serilog, IHostedService, GRPC, SignalR, MediatR? Yea you can just figure out the dozens of files you get when you download a nuget package and all of its dependencies. You want to reference your other projects? Go through asmdef.

You basically can do everything you want with Unity you just have to go through a lots of hoops to get there. You can use first/third party tools which work with a varying degree of success:

- [Nuget for Unity](https://github.com/GlitchEnzo/NuGetForUnity) does not properly supports nuget versions, still can't properly resolve some dependencies)
- [Zenject](https://github.com/modesttree/Zenject) or [Extenject](https://github.com/svermeulen/Extenject) (what the hell is going on with theese libraries?) works rather good for dependency injection framework but it has it's own quirks
- [Bovine Labs Analyzers](https://github.com/tertle/com.bovinelabs.analyzers) works surprisingly good
- UPM is nice, but at the end it does nothing but downloads a bunch of raw assets and places it in your project folder
- [OpenUPM](https://openupm.com/) is a godsent

And so you will manage to live with these things in mind, find every tool you can, just to make your life as a developer easier, but the actual problems remain. How can we pull Unity closer to the regular .Net ecosystem? At the barest minimum I want:

- Nuget
- .Net Standard libraries without Unity dependencies
- Microsoft.Extensions.Hosting with all of it's goodies (DI, ILogger, IHostedService, IConfiguration)
- Main entry point for the application
- Less pain in my life managing these things

So how can we achieve all this? By abusing the Unity Package manager. Is this solution foolproof? No. Does this work with any version of Unity? Absolutely not. Does this work with all target platforms/scripting backends/compatibility versions? Probably not. Can we just make this work? Yes if you put in the effort.

## Making this happen

### Getting the tools

For this project you will need:

- Unity 2019.3.11 (or any recent version maybe?)
- .Net Core SDK (3.1.201 at the moment)
- Visual Studio 2019 / Visual Studio Code / Rider
- Git, Git LFS

This is also probably a good time to start using [chocolatey](https://chocolatey.org/) if you are running on windows.

### Setting up the repo

This is 2020 we will be using git. The repo created with this tutorial can be found [here](https://github.com/KuraiAndras/UnityNuget).

- Add [any](https://github.com/github/gitignore/blob/master/VisualStudio.gitignore) C# gitignore to the root of the project
- Add a license because you never know
- Add a readme because the tools you use will always generate one
- Use Git Flow and you will love your life a lot more
- Think ahead and set up [LFS](https://thoughtbot.com/blog/how-to-git-with-unity), then regret it when problems come up

### Adding the projects

For this example we will set up three projects:

- UnityNuget.Shared: for a project with no Unity dependencies
- UnityNuget.Unity: for the Unity project
- UnityNuget.Unity.Dependencies: for the magic to happen

Your shared project csproj should look like this:
```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <LangVersion>7.3</LangVersion>
    <IsPackable>true</IsPackable>
    <PackageId>UnityNuget.Shared</PackageId>
    <PackageLicenseFile>LICENSE.md</PackageLicenseFile>
  </PropertyGroup>

  <ItemGroup>
    <None Include="..\LICENSE.md" Pack="true" PackagePath="$(PackageLicenseFile)" />
  </ItemGroup>

</Project>
```

Your dependencies csproj should look like this:
```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net471</TargetFramework>
    <LangVersion>7.3</LangVersion>
    <PackageId>UnityNuget.Unity.Dependencies</PackageId>
    <PackageLicenseFile>LICENSE.md</PackageLicenseFile>
  </PropertyGroup>

  <ItemGroup>
    <None Include="..\LICENSE.md" Pack="true" PackagePath="$(PackageLicenseFile)" />
  </ItemGroup>

</Project>
```

Also you need to add at least one class to your dependencies project.

Add unity project inside a folder with the same name in your root. Don't forget to add a unity [specific gitignore](https://github.com/github/gitignore/blob/master/Unity.gitignore) to that folder. Add one plus line to that gitignore to keep the manifest file which is ignored by the gitignore in the root:
```
!/**/manifest.json
```

Your project structure should look like this:

![image](/assets/images/FileStructure1.png)

### Reference standard projects from Unity

Here happens the magic with the Unity Package Manager. As I mentioned before at the most basic UPM does nothing special, but reads your manifest.json, finds your referenced packages (from the added registries, git or local file) and then copies the files around and bellow your package.json. That's it. Unity's latest package manager solution is nothing but a fancy copy paste. Which is a bit better then the old unitypackage which is a fancy zip file, and you are the copypasta.

How can we use this to our advantage?

First create an asmdef for the unity project to keep things clean. It should look something like this:
```json
{
    "name": "UnityNuget.Unity",
    "references": [],
    "includePlatforms": [],
    "excludePlatforms": [],
    "allowUnsafeCode": false,
    "overrideReferences": false,
    "precompiledReferences": [],
    "autoReferenced": true,
    "defineConstraints": [],
    "versionDefines": [],
    "noEngineReferences": false
}
```

The file structure should look like this:

![image](/assets/images/FileStructure2.png)

To make a usable unity package you need 3 things:

- [package.json](https://docs.unity3d.com/Manual/upm-manifestPkg.html) which describes your package
- [asmdef](https://docs.unity3d.com/Manual/ScriptCompilationAssemblyDefinitionFiles.html) file is not needed but makes working with things a lot better (think of it as csproj == asmdef)
- meta files which unity uses to identify files

We can add the first two, and then let unity add the meta files, as when it does not find them, they get generated. Add these to your shared project:
```json
// package.json
{
    "name": "com.unitynuget.shared",
    "version": "1.0.0",
    "displayName": "UnityNuget.Shared"
}
```

```json
// UnityNuget.Shared.asmdef
{
    "name": "UnityNuget.Shared",
    "references": [],
    "includePlatforms": [],
    "excludePlatforms": [],
    "allowUnsafeCode": false,
    "overrideReferences": false,
    "precompiledReferences": [],
    "autoReferenced": true,
    "defineConstraints": [],
    "versionDefines": [],
    "noEngineReferences": false
}
```

One problem here: if we just add a reference to this project in the manifest.json things will go to hell rather fast, because unity will try to load the files in the bin and obj folders. Fortunately there are some [patterns](https://docs.unity3d.com/Manual/SpecialFolders.html) by which unity ignores folders, so we can use this our advantage. To override the bin and out folders your have to modify the shared project's csproj:
```xml
<Project>
<!-- set output paths -->
  <PropertyGroup>
    <BaseIntermediateOutputPath>.obj\</BaseIntermediateOutputPath>
    <BaseOutputPath>.bin\</BaseOutputPath>
  </PropertyGroup>

<!-- this is needed -->
  <Import Sdk="Microsoft.NET.Sdk" Project="Sdk.props" />

  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <LangVersion>7.3</LangVersion>
    <IsPackable>true</IsPackable>
    <PackageId>UnityNuget.Shared</PackageId>
    <PackageLicenseFile>LICENSE.md</PackageLicenseFile>
  </PropertyGroup>

<!-- ignore meta files in the project -->
  <ItemGroup>
    <None Remove="**/**/*.meta" />
  </ItemGroup>

  <ItemGroup>
    <None Include="../LICENSE.md" Pack="true" PackagePath="$(PackageLicenseFile)" />
  </ItemGroup>

<!-- this is needed -->
  <Import Sdk="Microsoft.NET.Sdk" Project="Sdk.targets" />

</Project>
```

After this you must delete the old obj and bin folders. Also you should add a gitignore file to your shared project which ignores the generated meta files if they are not ignored by your root gitignore settings.

Now you can reference this package from unity in the manifest.json:
```json
{
  "dependencies": {
    // don't forget to use relative path
    "com.unitynuget.shared": "file:../../UnityNuget.Shared",
    "com.unity.collab-proxy": "1.2.16",
    // other references
  }
}
```
To use this package in your unity project you also need to add reference in unity to the new asmdef file:
```json
{
    "name": "UnityNuget.Unity",
    "references": [
        "UnityNuget.Shared"
    ],
    "includePlatforms": [],
    "excludePlatforms": [],
    "allowUnsafeCode": false,
    "overrideReferences": false,
    "precompiledReferences": [],
    "autoReferenced": true,
    "defineConstraints": [],
    "versionDefines": [],
    "noEngineReferences": false
}
```

Cool. Now every class you write in your shared project is accessible from unity, separated in a project, and ready to use by any other .net projects, publishable to nuget, because at the end of the day it is just a simple .net standard project.

#### Note
You can use this technique to reference other unity projects as intended. If you target multiple platforms with platform specific implementations, you could have a shared unity project, with another unity project targeting the needed platforms, thus you can separate platform specific code to different projects.

### Adding nuget support

Here comes the tricky part. What happens if the shared or the unity project needs some dependencies from nuget? We can use the same trick with the package.json as before, but with a little modification: now we don't want raw cs files around the package.json but dll-s. Remember, as long as Unity sees dlls which it can use, it will use it. The package.json can be thought of as a pointer to raw resources. The idea is that now we want to put the package.json next to the actual dlls we want to use, and let dotnet restore handle the resolution of the dlls. For this to work we need to modify the csproj file of the Unity.Dependencies project.

First add some dependency to the shared project:

```xml
<!-- Refernce some package -->
<ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Hosting" Version="3.1.3" />
</ItemGroup>
```

Reference the shared project from the dependencies project:
```xml
<ItemGroup>
  <ProjectReference Include="..\UnityNuget.Shared\UnityNuget.Shared.csproj" />
</ItemGroup>
```

This will cause that whenever we build the dependencies project it contains all the dependencies needed by the shared project. Now add package.json and asmdef files to the dependencies project the same way as before:

```json
// package.json
{
    "name": "com.unitynuget.unity.dependencies",
    "version": "1.0.0",
    "displayName": "UnityNuget.Unity.Dependencies"
}
```
```json
// UnityNuget.Unity.Dependencies.asmdef
{
    "name": "UnityNuget.Unity.Dependencies",
    "references": [],
    "includePlatforms": [],
    "excludePlatforms": [],
    "allowUnsafeCode": false,
    "overrideReferences": false,
    "precompiledReferences": [],
    "autoReferenced": true,
    "defineConstraints": [],
    "versionDefines": [],
    "noEngineReferences": false
}
```

Now we have to modify the Unity.Dependencies.csproj to place the files to some other folder the the default (Here it will be placed in bin/Dependencies):

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net471</TargetFramework>
    <LangVersion>7.3</LangVersion>
    <PackageId>UnityNuget.Unity.Dependencies</PackageId>
    <PackageLicenseFile>LICENSE.md</PackageLicenseFile>
  </PropertyGroup>

  <ItemGroup>
    <None Include="..\LICENSE.md" Pack="true" PackagePath="$(PackageLicenseFile)" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\UnityNuget.Shared\UnityNuget.Shared.csproj" />
  </ItemGroup>

<!-- Copy needed files to output -->
  <ItemGroup>
    <None Remove="*.meta" />
    <None Update="package.json" CopyToOutputDirectory="PreserveNewest" />
    <None Update="UnityNuget.Unity.Dependencies.asmdef" CopyToOutputDirectory="PreserveNewest" />
  </ItemGroup>

  <ItemGroup>
    <NugetDlls Include="./NugetDlls/**/*.dll" />
    <AllOutDirFiles Include="$(OutDir)/**/*" />
    <DependenciesFolder Include="$(OutDir)/../../Dependencies" />
  </ItemGroup>

  <Target Name="Remove Dll" AfterTargets="AfterBuild">
<!-- Delete the dlls not needed by the unity project -->
    <Delete Files="$(OutDir)/UnityNuget.Unity.Dependencies.dll" />
    <Delete Files="$(OutDir)/UnityNuget.Unity.Dependencies.pdb" />
    <Delete Files="$(OutDir)/UnityNuget.Shared.dll" />
    <Delete Files="$(OutDir)/UnityNuget.Shared.pdb" />

<!-- If you need specific dlls, or the .net framework targeted dll does not work for you place them unity the NugetDllsFolder. -->
<!-- Don't forget to delete the bad ones before copying -->
    <Copy SourceFiles="@(NugetDlls)" DestinationFolder="$(OutDir)/%(RecursiveDir)" />

<!-- Copy the dlls to a different directory to not depend on the out dir -->
    <RemoveDir Directories="@(DependenciesFolder)" />
    <Copy SourceFiles="@(AllOutDirFiles)" DestinationFolder="@(DependenciesFolder)" />
  </Target>

</Project>
```
To make this work you need to rebuild your project twice for the first time, and then whenever the dependencies change you need to rebuild this project.

Now add the dependencies package to unity in the manifest.json:
```json
{
  "dependencies": {
    "com.unitynuget.shared": "file:../../UnityNuget.Shared",
    "com.unitynuget.unity.dependencies": "file:../../UnityNuget.Unity.Dependencies/bin/Dependencies",
    "com.unity.collab-proxy": "1.2.16",
    // other dependencies
}
```
Reference the asmdef:
```json
{
    "name": "UnityNuget.Unity",
    "references": [
        "UnityNuget.Shared",
        "UnityNuget.Unity.Dependencies"
    ],
    "includePlatforms": [],
    "excludePlatforms": [],
    "allowUnsafeCode": false,
    "overrideReferences": false,
    "precompiledReferences": [],
    "autoReferenced": true,
    "defineConstraints": [],
    "versionDefines": [],
    "noEngineReferences": false
}
```

Great! Now we have nuget, can seperate non unity code to non unity projects, without relying on any third party components, manageable, and CI-CD ready.

### Other quality of life things

So how to go further? Still no Main method, async is still bad, how to use DI? Try some of my packages which work well with this kind of setup and solve these problems:

- [Injecter.Unity](https://github.com/KuraiAndras/Injecter) use Microsoft.Extensions.DependencyInjection inside unity, also helps create an application start point
- [MainThreadDispatcher.Unity](https://github.com/KuraiAndras/MainThreadDispatcher) Help deal with async code, by letting you run code on the main thread
- [Serilog.Sinks.Unity3D](https://github.com/KuraiAndras/Serilog.Sinks.Unity3D) Serilog sink which writes to unity Debug.Log

Also start using OpenUPM as it provides a really easy way to share unity packages.

### Notes

- [MSBuildForUnity](https://github.com/microsoft/MSBuildForUnity) makes this setup unnecessary, but right now it does not work with Unity 2019.1 and newer
- If you place code in the Unity.Dependencies project, and don't delete the dll produced, you can write c# 8.0 code there, with the [tradeoffs](https://stackoverflow.com/questions/56651472/does-c-sharp-8-support-the-net-framework) that come with using c# 8.0 under .net framework 4.7.1
- Not tested on mobile, web, console, IL2CPP
- On IL2CPP you might want to [customize stripping](https://docs.unity3d.com/Manual/ManagedCodeStripping.html)
- This was tested with .Net Standard 2.0 compatibility and Mono scripting backend, on Windows