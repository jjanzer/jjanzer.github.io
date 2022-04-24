---
title: "Managed and Unmanaged Library Interfaces with IL2CPP in Unity"
date: 2021-10-10T17:04:28-06:00
draft: false
---

## The Problem with Precompiled Managed Libraries in Unity

I work on an SDK that has a non-platform specific compiled managed C# library along with unmanaged C++ libaries. There are peculirarirties with iOS/tvOS/watchOS where you will want to use a static library as using a dynamic library is not *officially supported by Apple* like other platforms (Windows, Linux, macOS, Android). So far we've avoided having loose C# helper scripts alongside our managed C# dll and also we've avoided having different managed dlls for each platform. 

So the scenrio is the following you have C/C++ dynamic libraries for some platforms such as Windows `.dll`, macOS `.bundle` or `.dylib`, Linux `.so`, and Android `.so` and you have static libraries for iOS `.a`, tvOS `.a`, watchOS `.a`. How do you compile your managed C# Unity plugin into a single dynamic library that works on all platforms.

This page will describe a method for allowing you to continue to ship a single non-platform specific precompiled C# DLL without also shipping loose C#, though if you plan to support iOS and other platforms using IL2CPP you will need a loose C file as well.

Having only one managed DLL makes some things tricky in unity as we can't use defines in our to compile out different platform and architecture code and instead must rely on runtime checks instead. In other words

~~~ C#
#if UNITY_IOS || UNITY_TVOS
	//handle iOS specific code here
#else
	//handle other platforms here
#endif
~~~

must instead become 

~~~ C#
if(Application.platform == RuntimePlatform.IPhonePlayer || RuntimePlatform.tvOS)
{
	//handle iOS specific functions here
}
else
{
	//handle other platforms here
}
~~~

When using PInvoke via `DLLImport` in C# it requires a constant string, that means you can't change the string at runtime on demand.

~~~ C#
//Valid
[DllImport("MyLib")]
public static extern my_func();

//Also valid, used for statically linked libraries such as on iOS
[DllImport("__Internal")]
public static extern my_func();

//Also valid, string is const
public const string lib_name = "MyLib";
[DllImport(lib_name)]
public static extern my_func();

//Invalid! The name is no longer const meaning it could change during runtime
public string lib_name = "MyLib";
[DllImport(lib_name)]
public static extern my_func();
~~~

You could load your plugin manually for each platform such as using `LoadLibrary`  in `kernel32` on Windows. This leads to maintaining dynamic injection of the libraries per-platform and then on platforms where you can't dynamically, such as iOS, rely on loading a module/plugin with the `__Internal` trick. Hot plugin reloading does make developers lives easier when working on the native libraries, but increases your complexity and maintenance.

## A Solution

Since we're making a single managed we can't rely on the compile time options, and we can't rely on swapping strings out of the `DllImport` attributes. We could ship loose C# files to try to work around this but we want to avoid this for various reasons. This leaves us with an option; which is maintaining two classes where one class handles the dynamic import and the other class handles the static import. Then you'll have a wrapper class that determines which class to use at runtime.

The way we'll accomplish this is having an inteface that defines our C interface but doesn't implement them. Then we'll create two classes that implement the static methods from the interface and expose a wrapper prefix function that is not static. We'll also create an overall wrapper with the same method names as our C API so we don't externally call the wrapper functions. Finally we'll create some helpers to tell us if we should load the static or dynamic version of our interface class.

There's just one more thing we need to do now, which is we need to create a stub C interface/implementation file that is used for the static IL2CPP process if you plan to support IL2CPP targets outside of iOS. Without this you will get linker failures during builds with IL2CPP for platforms that are expecting dynamic linking instead of static linking (ie: non iOS targets such as on Android). This is kind of cheating as we now have to ship a *loose* source file, but I find this better as there is pretty much no logic in the C file and it's only used when using IL2CPP.


### Steps To Support One Managed Library with IL2CPP and native Libraries

To summarize you need to complete the following steps:

1. Create an interface that holds all your C APIs but add a prefix such as `call_` to them
1. Create a class that implements the interface using the dynamic library and exposes the true C function (you'll have 2 functions per C function here)
1. Duplicate this class and replace the `DllImport` library name for the static version `__Internal`
1. Create a wrapper class that exposes the public interface you want to use/expose
	1. Add a way to determine which class to instantiate based on platform
	1. House your constant strings here statically for ease of swapping
1. Add your function to a stub C file for IL2CPP

Now whenever you add a new C function you will:

1. Add the function signature to your interface
1. Implement the function dynamically and create a wrapper function for calling into it non-statically
1. Repeat for the static class
1. Expose the method to your wrapper class
1. Finally add your C function stub implemention to your stub file

Below is a blueprint for how one can implement a system like this.


#### Unity Library Interface Example

##### MyLib.cs

~~~ C#
public class MyLib
{
	//static linking
	public const string LibNameStatic = "__Internal";
	//dynamic linking
	public const string LibNameDynamic = "MyLib"

	//So why aren't we using bool? instead of 2 vars
	// it's b/c there's a cost to casting the IsStatic function to a regular bool and it's faster this way
	private static bool _isStatic = false;
	private static bool _isStaticCalc = false; //have we checked?
	/// <summary>
	/// Are we in an environment where we would be loading library methods statically/embedded or dynamically?
	/// </summary>
	/// <returns>False in all editor, true if ios/tvos/webgl</returns>
	public static bool IsStatic
	{
		get 
		{
			if(!_isStaticCalc)
			{
				//Regardless of what are target platform is if we're in editor we are always using the methods loaded via dynamic libraries
				if(Application.isEditor)
				{
					_isStatic = false;
				}
				else {
					//Other platforms may use static libs as follows below
					_isStatic = 
						Application.platform == RuntimePlatform.IPhonePlayer 
						|| Application.platform == RuntimePlatform.tvOS 
						|| Application.platform == RuntimePlatform.WebGLPlayer;
				}

				//We determined which linkage we need
				_isStaticCalc = true;
			}
			return _isStatic;
		}
		set
		{
			//You really shouldn't override this tbh
			UnityEngine.Debug.LogWarning("It's not advised to set isStatic manually, let the system handle it for you");
			_isStatic = value;
		}
	}

	private static IMyLib _myLib = null;
	/// <summary>
	/// This object is the native interface object for either static or dynamic linking
	/// </summary>
	public static IMyLib MyLib 
	{
		get 
		{
			if(_myLib == null){
				//TODO: handle dynamic reloading for win/macos/lin
				if(IsStatic)
				{
					_myLib = new MyLibStatic();
				} else 
				{
					_myLib = new MyLibDynamic();
				}
			}
			return _myLib;
		}
	}


	public static int mylib_add(int a, int b)
	{
		return MyLib.call_mylib_add(a, b);
	}
}
~~~

##### IMyLib.cs

~~~ C#
public interface IMyLib
{
	int call_mylib_add(int a, int b);
}
~~~

##### NativeDynamic.cs

~~~ C#
public class MyLibDynamic : IMyLib
{
	[DllImport(MyLib.LibNameDynamic)]
	public static extern int mylib_add(int a, int b);

	public int call_mylib_add(int a, int b)
	{
		return mylib_add(a, b);
	}
}
~~~

##### NativeStatic.cs
~~~ C#
public class MyLibStatic : IMyLib
{
	[DllImport(MyLib.LibNameStatic)]
	public static extern int mylib_add(int a, int b);

	public int call_mylib_add(int a, int b)
	{
		return mylib_add(a, b);
	}
}
~~~

##### NativeStub.c
~~~ C
void mylib_add(){}
~~~

Notice how the static and dynamic classes are identical outside of the library name in the `DllImport` attribute? This makes it fairly easy to maintain as you simply need to copy/paste each method. You could also write a script that generates the actual calls from your interface if desired as well.

Also notice that your stub C file doesn't need to match the interface at all, you can simply use void return types and no arguments for the declaration and empty definitions.

So now you can compile your C# files together as a single managed DLL and leave a copy of your stub C file next to it. Since Unity treats loose C and C++ files as plugins ensure you mark to exclude the C file from `iOS`, `tvOS`, and `watchOS` as those platforms are the ones that require static linking of your library in Unity. When distributing your plugins make sure you include the .meta (or use a .unitypackage) which will maintain the appropriate target inclusion/exclusions.

Now you're all set, no need to maintain multiple managed DLL files or loose C# scripts and you can also hide most of your interface inside your library if you don't plan on exposing it publicly as well.


## Dynamic Libraries on iOS

If you're curious about dyanic libraries on iOS this *does work* but it's not allowed for apps shipping via the App Store. Unity also will assume you are using a static library, so you'll need to manually copy your .dylib file and manually include it into your XCode project. 

Initially this is how we first started distributing Astra, but it requires additional steps after your iOS build in XCode so it's not a "one button" build without adding additional build scripts, and ultimately it won't work for the App Store.

[Additional Info](https://stackoverflow.com/questions/4733847/can-you-build-dynamic-libraries-for-ios-and-load-them-at-runtime)
