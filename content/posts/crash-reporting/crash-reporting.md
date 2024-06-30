---
title: "Crash Reporting"
date: 2024-06-30T10:21:04-07:00
draft: false
toc: false
images:
tags: 
  - tools
---

* [Supporting Minidumps](#supporting-minidumps)
* [Crash Report](#crash-report)
* [Win32 UI](#win32-ui)
* [Trigger Crash Reporter](#trigger-crash-reporter)
* [Where Reports Live](#where-reports-live)

If you've ever had to debug a fatal exception then you likely know how hard it can be to fix those issues when you only have the steps to reproduce the problem and sometimes you don't even have that. These days a lot of things can contribute to a crash and it's important to get as much information as you can. Hardware information is always very useful to have. Things like cpu and gpu models, how much RAM the system has, etc. Gathering hardware information can sometimes shed a light on problems specific to a piece of hardware or driver version. It would be great if there was an automated way to collect this information, among other things, and store it somewhere developers can look at the data to do some debugging on their end.

This is pretty much what all popular engines do. Unity and Unreal for example, have a separate application to handle crash reporting on their editors as well as any games made with those engines. The engine will make sure to launch their crash reporter whenever a fatal exception has occured. This application is very straightforward to use and it's responsibilties boil down to automatically bundling a minidump, most recent log file, and other data so that it can send it off to a server. The crash reporter also lets the user type some additional information to attach to the report. Things like what user was doing when the crash occured and anything else they may want to add. This is the approach the Tempest crash reporter takes. A screenshot below shows what the Tempest crash reporter looks like in all its Win32 glory. Looks sort of ugly but gets the job done.

![Crash Reporter](/images/crash-reporting/CrashReporter.png)

## Supporting Minidumps
If you aren't aware what application minidumps are then I will briefly mention what they are. In short, the minidump is a file that contains various useful information for debugging at a later point in time. Information such as call stack at the time of the fatal exception, local variables, loaded modules, and will also point to the source code line that triggered the exception. Needless to say, this is going to be the most important file to have from a crash report. However, these files do not get created automatically and the c++ project needs to be configured properly for them to be useful. Using Visual Studio 2022 as the IDE of choice, what you need to configure is in your C/C++ general section, set **Debug Information Format** to **Program Database** and then make sure that in your linker debugging section's **Generate Debug Info** is set to **Generate Debug Information**. This should make your release or shipping builds generate debug symbols in a PDB file. It is also good practice to save these PDB files when you make a new build so you can easily use the correct PDB file when loading up the minidump in VS or your debugger of choice.

The next thing required is to create the minidump file when the fatal exception is being handled. There are many articles out there that describe how to do this. It isn't a lot of code so I'll post below what Tempest does so you can see for yourself.

```cpp
// After this runs a minidump file is created at the location
// specified by the filepath variable.
static void CreateMinidump(struct _EXCEPTION_POINTERS* apExceptionInfo)
{
  HMODULE mhLib = ::LoadLibrary(_T("dbghelp.dll"));
  MINIDUMPWRITEDUMP pDump = (MINIDUMPWRITEDUMP)::GetProcAddress(mhLib, "MiniDumpWriteDump");
  
  char filepath[MAX_PATH + 1];
  snprintf(filepath, sizeof(filepath), "%s/crashdump.dmp", Application::GetAppDataDir()); 
  HANDLE hFile = ::CreateFileA(filepath, GENERIC_WRITE, FILE_SHARE_WRITE, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);
  
  _MINIDUMP_EXCEPTION_INFORMATION ExInfo;
  ExInfo.ThreadId = ::GetCurrentThreadId();
  ExInfo.ExceptionPointers = apExceptionInfo;
  ExInfo.ClientPointers = FALSE;
  
  pDump(GetCurrentProcess(), GetCurrentProcessId(), hFile, MiniDumpNormal, &ExInfo, NULL, NULL);
  ::CloseHandle(hFile);
}
```

It is important to mention that the minidump file will be created by the application that just had a fatal exception. This fatal exception could be anything, including memory corruption. Due to the unknown nature of the fatal exception it is very important to be careful with the code you run when handling the exception. Particularly take care to not do any or very few dynamic memory allocations as there is no guarantee that will work. Of course, this does not only apply to creating the minidump file but anything done while handling the exception. 

## Crash Report
In addition to creating a minidump file, the Tempest crash reporter will also create an additional text file that contains some useful information. It logs the exception code by value and also a string so it is easy to identify. A callstack is created but it's only useful in development as otherwise, in shipping builds the callstack won't be able to symbolicate the function addresses. It also adds some system info like what Windows OS, processor, etc. Once again, it gathers all this extra information with little dynamic memory allocations and where there are allocations, that's usually because the Windows API is doing some behind the scene. You can see a bit of the crash report information in the crash reporter screenshot above. 

The last couple things the crash report generates that are also very important have to do with a unique ID for the report itself that is tied to the machine that created the crash and the other is the game's build number. The crash report ID is super useful as you can use external tools to bundle crash reports based on that ID so you can get a sense of what problems might be plaguing a certain user. You can also use it to track progress on improving stability for that user and set of hardware. The game build/version number is also important to track in my opinion. You can again use external software to bundle crash reports based on what version of the game and get a sense of what the major problems are for that build. I'll mention soon enough what software I'm using to be able to perform these queries.

## Win32 UI
Using the Win32 API to make the UI for the crash reporter was for the most part easy but there was one unexpected thing that was way more annoying to handle than I thought it would be. You'll notice in the crash reporter screenshot that there is a text box where the user can type any additional information into it. I wanted this box to handle all the main keyboard shortcut that people expect these days to work. Copy and paste keyboard shortcuts for instance, are supported by default in the Win32 API so no issues there but if you wanted to highlight all the text using Ctrl+a then you would find out that does nothing. Adding support for that was not as trivial as you might think and searching online yielded several workarounds that were more complex than I thought it needed to be. Eventually I stumbled on a post that ran into the same issue and it presented the solution they used. It was a simple solution and was easy to integrate into my crash reporter. All you have to do is create a new window procedure that specifically handles the Ctrl+a keyboard shortcut and then make the text box window use that function. 
```cpp
// This function will handle the Ctrl+a keyboard shortcut to highlight all the text
LRESULT CALLBACK Edit_Prc(HWND hwnd, UINT msg, WPARAM wParam, LPARAM lParam, UINT_PTR uIdSubclass, DWORD_PTR dwRefData)
{
  if (msg == WM_CHAR && wParam == 1)
  {
    SendMessage(hwnd, EM_SETSEL, 0, -1);
    return 1;
  }
  return DefSubclassProc(hwnd, msg, wParam, lParam);
}

// Later when creating the input text box window
HWND hwnd = CreateWindowW(...); // handle to the input text box window
SetWindowSubclass(hwnd, Edit_Prc, 0, 0); // Sets the correct window proc to handle the select all text keyboard shortcut
```

## Trigger Crash Reporter
Now that we've talked about how the crash report data is created and what it contains, let's talk about how the executable prepares itself to launch the crash reporter if a fatal exception happens. In c++ this done in a very simple way. It boils down to setting up a callback that will run when a fatal exception happens. To my knowledge, on Windows there are at least two different functions that allows this setup. The Tempest engine uses [SetUnhandledExceptionFilter](https://learn.microsoft.com/en-us/windows/win32/api/errhandlingapi/nf-errhandlingapi-setunhandledexceptionfilter) and the other option is [_set_se_translator](https://learn.microsoft.com/en-us/cpp/c-runtime-library/reference/set-se-translator?view=msvc-170). Whichever function the application uses the setup is the same and that's to call the respective function on application initialization. Deciding which function to use will highly depend on your application as well as any requirements imposed by third party libraries used so be sure to read the documentation and choose accordingly. You can easily test your setup by dereferencing an uninitialized pointer and run the executable without the debugger attached. If it is setup correctly then you should see your callback get called when the application crashes.

## Where Reports Live
We've yet to mention what happens when the user clicks "Send" in the crash reporter. Obviously we need a place to store these crash reports. Some platforms may already offer a place for that and you simply need to configure your application to use it but in other cases you need to host that data yourself. Being a small developer your storage needs are going to be pretty basic and if you can save some money then all the better. This was the main thing I wanted to figure out before deciding to make the crash reporter since if there is no good and cheap solution that meets a small developer's needs then it might not be worth spending the time to develop it. Looking for solutions I stumbled on to a Twitter post where an indie developer mentioned using Slack as a place to send to the crash reports to. I don't tend to use Slack personally but I do use Discord and decided to look for a Discord specific solution. To my excitement I saw that a Discord Webhook could be used to send data to a server. 

The steps to set this up were pretty simple. Just make a new server for the game and in that server you can make a private channel where it will host all the crash reports. Through the Discord app you can create a special webhook id for the channel that you can then use with a HTTP post request in order to send the crash report data to the channel. So the crash reporter takes care of making a zip file containing the minidump, crash report, and log file. This is set as an attachment to the HTTP post request. It also sends a string with the post request which contains the message the user typed in the crash reporter and a few other key strings that are useful for searching through the channel in Discord. These key strings are things like the user's unique ID and game build version number. You can use either one to filter the crash report channel using Discord's own search feature. So if you wanted to find all the reports attached to a specific user you can search the channel using the user's unique ID and see all the reports generated by that user. All super useful and for a small developer probably everything you need. 