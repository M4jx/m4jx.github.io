---
title: Cheating on Exams with Simulated User Input
date: 2026-01-17 02:12:00 +0100
image: /assets/img/fc31032b-95d2-425d-a6cb-4b53b06d5ce0.png
categories: [Exploit]
tags: [exploit, cheating]     # TAG names should always be lowercase
---

About six years ago, back when I was in high school and my biggest daily responsibility was not missing the bus, I ran into a very familiar academic enemy: subjects that reward memorization instead of understanding.

History was the worst.

Unlike math or physics, where you actually think and apply logic, history required you to become a human S3 bucket. Dates, names, events, and thousands of words you were expected to remember. Our teacher didn’t really teach; he outsourced that job to a website and said, *“Read the docs,”* as if we’d just asked a question on Stack Overflow.

To pass the exams, you must reproduce the text almost word for word. Fail to do that, and you’d be doing that exam again.

I hated it.

Not because history is boring (although, at the time, it absolutely was), but because the learning process felt hollow. No reasoning. No problem-solving. Just memorization. And since I had zero interest in turning my brain into a glorified char buffer, I started thinking... creatively.

## The Locked-Down Exam Environment

Our school used a special digital exam application installed on school laptops. Once an exam started, it entered full lockdown mode: fullscreen only, no alt-tabbing, no browsers, and no other programs. The application displayed the questions and input fields for the answers. After submitting the exam, the computer unlocked and returned to normal operation. The intent was obvious: prevent cheating.

Naturally, the intended lesson was *“study harder.”*\
Naturally, I chose a different lesson.

It was time to bypass the system entirely.

## First Attempt: Virtualizing the Problem

If the software locked down the computer, then maybe we could run it inside a virtual machine and use the host system to look up the answers.\
It would work, right?\
Right?

It didn’t. The software refused to run and displayed an error stating that it does not support virtualized environments.

![Desktop View](/assets/img/cant-run-under-vm.png){: .center }

Clearly, the software had VM detection mechanisms.

At this point, I had two theoretical options:
1. Spend significant time reverse-engineering the exam software to locate and patch the VM checks
2. Find a different solution

I had some basic reverse-engineering knowledge, enough to know that this would not be quick or easy task for me. So I went looking for another solution.

## Second Attempt: Let the Hardware Type

At the time, I happened to own a USB Rubber Ducky, a device I enjoyed experimenting with. It had been sitting around collecting dust, because apparently hacking into other people's computers without their consent is illegal.

For the unfamiliar, it’s a USB device that pretends to be a regular keyboard. Whatever you program into it, it just types out on the computer by itself. And it types fast. Really fast.

That’s when a new idea popped into my head 💡:

> What if I preload the entire exam material into the Rubber Ducky and let it type everything for me during the exam?

To test this, I installed the exam software at home, launched a demo exam, plugged in the Rubber Ducky, and watched it type the entire text straight into the input field.

![Desktop View](/assets/img/usb-rubber-ducky.png){: .center w="400" }

Now, the plan was simple:

1. Predict the exam scope
2. Copy all relevant text from the website
3. Load it into the USB Rubber Ducky
4. Plug it in during the exam

## The Exam Day

The plan worked beautifully. Once the USB Rubber Ducky was connected, it typed thousands of words directly into the exam’s input field. From there, I just reshaped paragraphs to match the questions. It was like bringing the textbook into the exam, except the textbook typed itself at superhuman speed.

Effective? Absolutely.\
Stealthy? Not at all.

Plugging in a USB device during an exam isn’t subtle. I had to connect it, wait two to three minutes while it typed nonstop, unplug it, hide it in my backpack, and hope the teacher wouldn’t walk by and notice my screen writing an essay by itself, while I sat there pretending my heart wasn’t trying to escape my chest.

It was stressful. And far too visible. I needed something less likely to end my academic career in a meeting with the principal.

## Third Attempt: Beating Hardware with Software

The exam software cleared the clipboard as soon as an exam starts. The goal is obvious: prevent students from copying stuff beforehand and pasting them during the exam. But what if I could copy text after the exam had already started? Then the problem would be solved, and I could simply paste everything into the input field.

This was pre-ChatGPT. Apparently, we were expected to think for ourselves and write code like cavemen.

After some research, I discovered the Win32 API function `SetClipboardData`, which allows a program to place data into the clipboard.

I wrote a small program in C that would:

1. Wait for a specific key press (F11) using a global keyboard hook
2. Read the text file from disk containing the answers
3. Copy its contents to the clipboard

I don’t have the original code, but the snippet below shows the general idea.

```c
#define _CRT_SECURE_NO_WARNINGS
#include <windows.h>
#include <stdio.h>

#define FILE_PATH "C:\\Users\\Majx\\Desktop\\exam.txt"

HHOOK g_hook = NULL;

void CopyFileToClipboard(void) {
    printf("[INFO] Attempting to read file: %s\n", FILE_PATH);

    FILE* f = fopen(FILE_PATH, "rb");
    if (!f) {
        printf("[ERROR] Failed to open file\n");
        return;
    }

    fseek(f, 0, SEEK_END);
    long size = ftell(f);
    rewind(f);

    if (size <= 0) {
        printf("[ERROR] File is empty or size invalid\n");
        fclose(f);
        return;
    }

    HGLOBAL hMem = GlobalAlloc(GMEM_MOVEABLE, size + 1);
    if (!hMem) {
        printf("[ERROR] GlobalAlloc failed\n");
        fclose(f);
        return;
    }

    char* data = (char*)GlobalLock(hMem);
    if (!data) {
        printf("[ERROR] GlobalLock failed\n");
        GlobalFree(hMem);
        fclose(f);
        return;
    }

    size_t bytesRead = fread(data, 1, size, f);
    if (bytesRead != (size_t)size) {
        printf("[ERROR] fread failed\n");
        GlobalUnlock(hMem);
        GlobalFree(hMem);
        fclose(f);
        return;
    }

    data[size] = '\0';

    GlobalUnlock(hMem);
    fclose(f);

    printf("[INFO] File read successfully (%ld bytes)\n", size);

    if (OpenClipboard(NULL)) {
        EmptyClipboard();
        SetClipboardData(CF_TEXT, hMem);
        CloseClipboard();
        printf("[INFO] Clipboard updated\n");
    }
    else {
        printf("[ERROR] Failed to open clipboard\n");
        GlobalFree(hMem);
    }
}

LRESULT CALLBACK KeyboardProc(int nCode, WPARAM wParam, LPARAM lParam) {
    if (nCode == HC_ACTION) {
        KBDLLHOOKSTRUCT* kb = (KBDLLHOOKSTRUCT*)lParam;
        if (wParam == WM_KEYDOWN && kb->vkCode == VK_F11) {
            printf("[EVENT] F11 pressed\n");
            CopyFileToClipboard();
        }
    }
    return CallNextHookEx(g_hook, nCode, wParam, lParam);
}

int main(void) {
    printf("[INFO] Starting F11 clipboard hook\n");

    g_hook = SetWindowsHookEx(
        WH_KEYBOARD_LL,
        KeyboardProc,
        GetModuleHandle(NULL),
        0
    );

    if (!g_hook) {
        printf("[ERROR] Failed to install keyboard hook\n");
        return 1;
    }

    printf("[INFO] Hook installed. Press F11 to copy file to clipboard.\n");

    MSG msg;
    while (GetMessage(&msg, NULL, 0, 0)) {
        TranslateMessage(&msg);
        DispatchMessage(&msg);
    }

    UnhookWindowsHookEx(g_hook);
    return 0;
}
```

Testing it was trivial. Press F11, paste into Notepad, and it worked perfectly. It was simple and incredibly stealthy.

![Desktop View](/assets/img/setclipboarddata-with-keyboard-hook.gif){: .center }

Then I tested it inside the exam software.

It failed.

My program would be terminated almost instantly after an exam is started, which made it clear that the exam software actively scans and kills untrusted processes.

I even tried marking the process as critical using the undocumented Windows API `RtlSetProcessIsCritical`, which tells the operating system to immediately crash if that process is terminated. I hoped this would stop the exam software from killing it. Instead, the process still got terminated and took the entire system down with it in a glorious blue-screen crash.

![Desktop View](/assets/img/outstanding-move.png){: .center w="500" }

At this point, I was back to first square with same options:

* Spend significant time reverse-engineering the exam software to locate and patch its process checks
* Find a different solution

Reverse-engineering was off the table, so I kept searching for another easier solution.

## The Breakthrough: Borrowing a Trusted Process

If running new processes wasn’t allowed, maybe I could *borrow* one that already had the system’s trust?

I already knew a little bit about the internals of malware and game hacking, and one technique in particular came to mind: DLL injection 💉.

For the unfamiliar, DLL injection is a technique that allows code to run inside the address space of another process by forcing it to load a dynamic-link library.

After closing the exam, I reviewed the list of running processes and noticed a survivor: the Synaptics touchpad driver.

![Desktop View](/assets/img/synaptics-task-manager.png){: .center }

The plan was straightforward. Convert my EXE into a DLL and inject it into that process.

Initial tests behaved exactly as expected. The DLL was injected and loaded successfully, my code executed inside the Synaptics process, and clipboard copy worked exactly as they did from a normal EXE. Pasting into Notepad worked.

Then I started the demo exam, and things got weird.

Pressing F11 then pasting did nothing. Nothing was copied to the clipboard.

Closing the exam immediately restored functionality. I could paste into Notepad again as if nothing had happened.

The pattern was clear. The moment the exam is started, all clipboard write operations stopped working. Once it closed, everything resumed.

That’s when it clicked.

Since the code was running inside a trusted process and couldn’t write to the clipboard when the exam was running, the only remaining explanation was that the exam software was intercepting Win32 clipboard APIs and refusing write access from any process except itself.

Once again, I was left with the same choices:

* Reverse-engineer the software and disable the clipboard hooks
* Find a different solution

You already know I wasn't going to reverse engineer it!

## The Final Trick: Becoming the Keyboard (Again)

I went back to what had already worked once: keyboard input.

If the system trusted keyboard events, maybe I could become the keyboard, in software.

After some research, I found the `SendInput` function in the Win32 API. It allows software to inject keyboard events directly into the system’s input stream.

From there, everything fell into place:

* Store the exam material in a file on disk
* Read it character by character
* Simulate each keystroke using `SendInput`
* Trigger the entire sequence with an unused key (F11, because tradition)

I don’t have the original code, but the snippet below shows the general idea.

```c
#define _CRT_SECURE_NO_WARNINGS
#include <windows.h>
#include <stdio.h>

#define FILE_PATH "C:\\Users\\Majx\\Desktop\\exam.txt"

HHOOK g_hook = NULL;

void SendTextAsKeyboard(const wchar_t* text) {
    size_t len = wcslen(text);
    printf("[INFO] Sending %zu characters via SendInput\n", len);

    for (size_t i = 0; i < len; i++) {
        INPUT inputs[2] = { 0 };

        // Key down
        inputs[0].type = INPUT_KEYBOARD;
        inputs[0].ki.wScan = text[i];
        inputs[0].ki.dwFlags = KEYEVENTF_UNICODE;

        // Key up
        inputs[1].type = INPUT_KEYBOARD;
        inputs[1].ki.wScan = text[i];
        inputs[1].ki.dwFlags = KEYEVENTF_UNICODE | KEYEVENTF_KEYUP;

        SendInput(2, inputs, sizeof(INPUT));
    }
}

void ReadFileAndType(void) {
    printf("[INFO] Reading file: %s\n", FILE_PATH);

    FILE* f = fopen(FILE_PATH, "rb");
    if (!f) {
        printf("[ERROR] Failed to open file\n");
        return;
    }

    fseek(f, 0, SEEK_END);
    long size = ftell(f);
    rewind(f);

    if (size <= 0) {
        printf("[ERROR] File empty or invalid\n");
        fclose(f);
        return;
    }

    char* buffer = (char*)malloc(size + 1);
    if (!buffer) {
        printf("[ERROR] malloc failed\n");
        fclose(f);
        return;
    }

    size_t read = fread(buffer, 1, size, f);
    fclose(f);

    if (read != (size_t)size) {
        printf("[ERROR] fread failed\n");
        free(buffer);
        return;
    }

    buffer[size] = '\0';
    printf("[INFO] File read successfully (%ld bytes)\n", size);

    // Convert to Unicode
    int wlen = MultiByteToWideChar(CP_UTF8, 0, buffer, -1, NULL, 0);
    if (wlen <= 0) {
        printf("[ERROR] UTF-8 conversion failed\n");
        free(buffer);
        return;
    }

    wchar_t* wtext = (wchar_t*)malloc(wlen * sizeof(wchar_t));
    if (!wtext) {
        printf("[ERROR] malloc failed (unicode)\n");
        free(buffer);
        return;
    }

    MultiByteToWideChar(CP_UTF8, 0, buffer, -1, wtext, wlen);

    printf("[INFO] Typing text now...\n");
    SendTextAsKeyboard(wtext);

    free(wtext);
    free(buffer);
}

LRESULT CALLBACK KeyboardProc(int nCode, WPARAM wParam, LPARAM lParam) {
    if (nCode == HC_ACTION) {
        KBDLLHOOKSTRUCT* kb = (KBDLLHOOKSTRUCT*)lParam;
        if (wParam == WM_KEYDOWN && kb->vkCode == VK_F11) {
            printf("[EVENT] F11 pressed\n");
            ReadFileAndType();
        }
    }
    return CallNextHookEx(g_hook, nCode, wParam, lParam);
}

int main(void) {
    printf("[INFO] Starting global keyboard hook (F11 trigger)\n");

    g_hook = SetWindowsHookEx(
        WH_KEYBOARD_LL,
        KeyboardProc,
        GetModuleHandle(NULL),
        0
    );

    if (!g_hook) {
        printf("[ERROR] Failed to install keyboard hook\n");
        return 1;
    }

    printf("[INFO] Hook installed. Focus a text field and press F11.\n");

    MSG msg;
    while (GetMessage(&msg, NULL, 0, 0)) {
        TranslateMessage(&msg);
        DispatchMessage(&msg);
    }

    UnhookWindowsHookEx(g_hook);
    return 0;
}
```

Now it was time to test in in the exam software.

Press F11, and the system typed the text as if I were calmly entering it myself.

> I don’t have the exam software anymore, so this GIF shows the same idea using Notepad.

![Desktop View](/assets/img/autotype-with-keyboard-hook.gif){: .center }

Now I had achieved the same thing as the USB Rubber Ducky, but purely in software, making it far more stealthy.

In the end, I passed the exam without reading a single page from the website, which was kind of the point.

![Desktop View](/assets/img/passed-exam.jpg){: .center }


## Lessons Learned

What started as frustration with memorization turned into a another lesson in offensive thinking: understanding constraints, analyzing defenses, and finding creative solutions.

I’m not proud of cheating, but I am grateful for what the experience taught me. It pushed me to learn about operating systems, process trust, input handling, and flawed security assumptions.

In hindsight, it wasn’t really about passing a history exam (okay, it was, but still).

That lesson stuck with me far longer than any memorized paragraph from that history subject ever did.
