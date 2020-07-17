# Prevent_Process_Creation
Record & prevent process creation in kernel mode

# Study Notes
1. Use `PsSetCreateProcessNotifyRoutineEx` to register a routine. `PsSetCreateProcessNotifyRoutine` has limited functionality.
2. Using `PsSetCreateProcessNotifyRoutineEx` requires the image that contains the callback pointer to have `IMAGE_DLLCHARACTERISTICS_FORCE_INTEGRITY` set in its image header. Otherwise, the function call will return `STATUS_ACCESS_DENIED`. (https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/ntddk/nf-ntddk-pssetcreateprocessnotifyroutineex#return-value) To do this, add `/integritycheck` in linker parameters. (https://social.technet.microsoft.com/wiki/contents/articles/255.forced-integrity-signing-of-portable-executable-pe-files.aspx)
3. To print `PCUNICODE_STRING` using `DbgPrint`, use `%wZ` format specifier.
4. To turn off "Spectre Mitigation", go to project properties - C/C++ - Code Generation - set "Spectre Mitigation" to "Disabled".
5. Loading a x86 driver on a x64 system will fail. `StartService` will give an error code 1275, which is "This driver has been blocked from loading". When you meet this error, think about architecture first, then think about if the system is really blocking it from loading.
6. Remember to set `DriverObject->DriverUnload` in DriverEntry, or the driver won't be unloaded correctly. Some drivers I previously wrote didn't do this, so those drivers can't unload correctly. If the driver is not unloaded correctly, you won't be able to load it for a second time and `CreateService` will give an error code 1073, which is "The specified service already exists".
7. If `DriverEntry` doesn't return `STATUS_SUCCESS`, user mode process that called `StartService` will receive an error even the code in the driver is executed.
8. To prevent a process creation, modify `CreateInfo` parameter in `PcreateProcessNotifyRoutineEx` routine. (https://webcache.googleusercontent.com/search?q=cache:4vxTVzmlrd4J:https://bitnuts.de/articles/blocking_process_creation_using_a_windows_kernel_driver.html+&cd=11&hl=en&ct=clnk&gl=ca) e.g.
```c
CreateInfo->CreationStatus = STATUS_ACCESS_DENIED;
```
9. The above `CreationStatus` will cause a error popup when you trying to create a new process. To avoid a error message popup, we can set `CreateInfo0>CreationStatus` to `STATUS_ACCESS_DISABLED_NO_SAFER_UI_BY_POLICY`. (https://blog.csdn.net/a907763895/article/details/52863952)
10. This method is only available for Windows Vista and above. I havn't test it on Windows XP. (https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/ntddk/nf-ntddk-pssetcreateprocessnotifyroutineex#requirements)

# Notes for Windbg
Finally I figured it out how to use Windbg... (I am too stupid XD)
1. You need to install the correct version of WDK. Using the newest version is strongly suggested... Or the debugger will likely fail to connect.
2. How to set host computer IP: `bcdedit /dbgsettings net hostip:w.x.y.z port:n` (https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/setting-up-a-network-debugging-connection#setting-up-the-target-computer)
3. `DbgPrint` prints nothing: Enter `ed nt!KD_DEFAULT_MASK 8` in Windbg to enable verbose output. (https://reverseengineering.stackexchange.com/questions/16685/how-can-i-receive-dbgprint-messages-in-windbg-on-windows-10)
4. Using `x /D xx!yy` to show symbols.
5. Load symbols: `.sympath+ folder`, then `.reload`
6. Good reference: https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debug-universal-drivers--kernel-mode-
7. We can use `!process` to list processes. `!process PID verbose_level`

# What I want to do
Now I finally figured it out how to prevent process creation in Windows 10 x64. I am going to figure it out how to prevent file creation / deletion using kernel mode driver. I will need to learn more about file system minifilter. After doing that, I may make a simple driver that communicates with my user-mode process to show warnings when it detects process creation and file creation.
