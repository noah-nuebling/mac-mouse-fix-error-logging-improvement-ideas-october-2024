# Error Logging Improvements Ideas for Mac Mouse Fix (October 2024)

> [!NOTE]
> This repo is just a note to jot down my ideas. Not sure why I'm creating a GitHub repo for this instead of just an Apple Note or something. I hope It's not a terrible idea to make 'Note Taking Repos'. Might be bad that we can't sort / filter them?

Background:
So today (Oct 22 2024) I got sucked into researching how to improve bug reports for Mac Mouse Fix. I don't want to work on this right now, since I don't want to get sidetracked from working on the Licensing System. So I'm putting my ideas here so I don't forget them

---

# Intro / Motivation

So for mac-mouse-fix-feedback-assistant, bug reports there are 3 tiers of actionable diagnostic data we request from users:

1. Reproduction Steps
2. Crash Reports
3. Console Logs

For hard-to-reproduce crashes, we really want the Console Logs, but we almost never get them from users. \
In the last days I've had several reports about hard-to-reproduce issues and every time, once I tell them how to gather Console Logs, I just don't get any replies anymore. 
(This is really funny for some reason.)

Currently, we just tell users to always run Console.app in the background and then note down the time and copy-paste the logs once the error occurs. (We don't even tell them that you can select all of the (potentially millions) of logs with Command-A)

I think this is clearly prohibitively hard / clunky to do. 

-> If we want to effectively address hard-to-reproduce bugs, we need to make it easier for people to gather and send Console Logs.
