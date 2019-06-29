+++
title = "Common Ide Keyboard Shortcuts"
date = 2019-06-29T22:56:30+10:00
draft = false
tags = []
categories = []
+++

# Initial Thoughts

I’m working on (yet and tiresomely) another IDE for Vim and other editors, and thought that it would be useful to survey the ‘lay of the land’ in the space of keyboard shorcuts in existing IDEs. Below are the results of a 20 minute Googling on the subject.

# The Sensis

|Editor|Start Debugging|Step Into|Step Over|Step Out/Step Return|Continue/Pause/Resume|Stop|
|---|---|---|---|---|---|---|
Eclipse|?|F5|F6|F7|F8|F12|?
Visual Studio|?|F11|F10|Shift-F11|F5|Shift-F5|
Android Studio|Shift-F9|F7|F8|Shift-F8|Alt-F9/F9|?
Netbeans 8.0|Ctrl-F5|F7|F8|Ctrl-F7|F5|Shift-F5
IntelliJ|?|F7|F8|Shift-F8|F9|?

# Conclusions:

It would appear that start and stop keys are ill-defined - at least in terms of keyboard shortcuts. The main reason these are missing is that there was slim-to-no information on these online. I just assume everyone is using the start/stop icons that most IDEs provide instead of doing this in an input-consistent way. It speaks volumes to the psychology behind clicking a button to start debugging as opposed to pressing a key combination. It’s still seen as that ‘entertaining’ part where the developer ‘clicks the button’ to take a ride on luck before discovering the turmoil that lay within their own paradigm of code logic.

Looking at the table above, there are only a few things that can be derived by the current state of play in the IDE space:

- F7 looks like a good key to ‘step into’ a code base
- F8 is a likely contender as a ‘step over’ key binding
- Anything to step out of the current line of code should be a key combination of a Ctrl or Shift and a function key above F6
- F5 or F9 are common for continuing the debugging process once started - I find this super confusing, to be honest.
- As previously mentioned, start and stop are ill-defined from a debugging perspective and is probably more due to the psychology involved with a click vs a keyboard shortcut than anything else (IMHO).

Anyway, this was a fun little survey in the area of popular IDEs and is by on means exhaustive, but more a way of getting an understanding for what is being used out there. Happy keyboard coding!
