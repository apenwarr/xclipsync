== xclipsync ==

xclipsync is a trivial little script for synchronizing the X11 "clipboard"
selection (the one you use with Ctrl-C/Ctrl-V, as opposed to
highlight/middleclick) between two X11 servers.

For example, if you use Xnest or Xephyr (nested X11 servers inside your main
one), you can use xclipsync to keep their clipboards tied together.  I use
this with ChromeOS+crostini, so that I can use a "full" X11 session with my
favourite window manager.

Some people suggest using the "synergy" daemon for this, but unfortunately
it seems to have many longstanding bugs related to clipboard
synchronization.  In particular, it only seems to sync the clipboard once
when one X server acquires the selection; it doesn't seem to sync again
until you make a selection on the other X server, and so on, alternating
back and forth.  Also, statically transfers the clipboard data as soon as it
changes owners, which is not quite correct: if you transfer the clipboard to
rxvt, for example, then rxvt can change the *content* of the clipboard
numerous times without sending any notifications.  It would be better to
retrieve the clipboard data from the remote screen only at paste time, which
is what xclipsync does.


=== How do I use it? ===

First, install tcl/tk (yes!  In 2018!) and xclip.  Then just put something
like this in your .xsession:

    if [ "$DISPLAY" != ":0" ]; then
        xclipsync &
    fi

This will sync your nested X server's clipboard with the clipboard on :0. 
You can create any number of X servers synced this way, and they will all
sync smoothly with :0 and therefore with each other.


== How does it work? ===

xclipsync is a shell script that just alternates between two steps:

1. Acquire clipboard ownership on server A and, when someone on server A
   tries to paste the clipboard, send them the data from server B.  If
   someone else acquires the clipboard on server A (ie. by copying text),
   go to step 2.

2. Vice versa.

The actual work of step 1 and 2 are done by a silly tcl/tk script called
xclipfrom.  Why tcl/tk?  Because it was the easiest way to listen for
clipboard events.  xclip *almost* does everything we want, but you have to
provide it with the clipboard text immediately when acquiring the clipboard,
and that's not quite valid: the clipboard owner could change the clipboard
content as often as it wants, without sending any notifications.  With
tcl/tk, we can acquire clipboard ownership, and handle the request each time
someone wants to paste.

Other people write scripts which just fetch the clipboard content
periodically from each node and then see if it has changed, but that creates
race conditions (especially when syncing multiple screens) and requires you
to pick a delay between syncs, which means you have to choose between
wasting CPU time and waiting before you can paste.  xclipsync is
instantaneous but doesn't waste time polling when you're not using the
clipboard.


== Room for improvement ==

This set of scripts really sucks!  There's plenty of room to improve. 
Patches are welcome.  Suggestions:

- Remove hardcoded CLIPBOARD and let people sync the PRIMARY (ie.
  middleclick) selection as well.  I tested it already, and it works fine,
  but we'll have to add command line options to make it work.

- Better error handling if xclip is missing, etc.

- Send log messages to stderr instead of stdout.

- Don't hardcode :0.  Let the caller decide which screen to sync with.

- Convert from tcl/tk to something less obsolete, like C or python.

- Modify xclip itself to add its one missing feature: be able to run a
  command to retrieve text, rather than reading stdin at startup.

- etc.
