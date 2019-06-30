+++
title = "Common IDE Keyboard Shortcuts"
date = 2019-06-29T22:56:30+10:00
draft = false
tags = []
categories = []
+++

I’m working on (yet and tiresomely) another IDE for Vim and other editors, and thought that it would be useful to survey the ‘lay of the land’ in the space of keyboard shorcuts in existing IDEs. Below are the results of a 20 minute Googling on the subject.

# The Representative Sample

|Editor|Start Debugging|Step Into|Step Over|Step Out / Step Return|Continue / Pause / Resume|Stop|
|---|---|---|---|---|---|---|
Eclipse|?|F5|F6|F7|F8|F12|?
Visual Studio|?|F11|F10|Shift-F11|F5|Shift-F5|
Android Studio|Shift-F9|F7|F8|Shift-F8|Alt-F9|F9
Netbeans 8.0|Ctrl-F5|F7|F8|Ctrl-F7|F5|Shift-F5
IntelliJ|?|F7|F8|Shift-F8|F9|?

# Conclusions:

Looking at the table above, there are only a few things that can be derived by the table above:

- F7 is commonly used to ‘step into’ a code base
- F8 is commonly used to ‘step over’ code
- Anything to 'step out' of the current line of code should be a key combination of a Ctrl or Shift and a function key above F6
- F5 or F9 are common for continuing the debugging process once started

Anyway, this was a fun little survey in the area of popular IDEs and is by no means exhaustive; but more a way of getting an understanding for what is being used out there. Happy keyboarding!
