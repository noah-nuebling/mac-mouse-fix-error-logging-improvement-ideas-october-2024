# Error Logging Improvements Ideas for Mac Mouse Fix (October 2024)

> [!NOTE]
> This repo is just a note to jot down my ideas. Not sure why I'm creating a GitHub repo for this instead of just an Apple Note or something. I hope It's not a terrible idea to make 'Note Taking Repos'. Might be bad that we can't sort / filter them?

### Background

So today (Oct 22 2024) I got sucked into researching how to improve bug reports for Mac Mouse Fix. I don't want to work on this right now, since I don't want to get sidetracked from working on the Licensing System. So I'm putting my ideas here so I don't forget them.

## Motivation: We never get Console Logs

So for mac-mouse-fix bug reports there are generally 3 tiers of actionable diagnostic data we request from users:

1. Reproduction Steps
2. Crash Reports
3. Console Logs

(These 3 type of data are also requested by mac-mouse-fix-feedback-assistant)

For **hard-to-reproduce crashes**, we really want the **Console Logs**, it's the most detailed way to introspect an application, short of attaching a debugger. 

However - **We almost never get Console logs from users.**

In the last days I've had several reports about hard-to-reproduce issues and every time, once I tell them how to gather Console Logs, I just don't get any replies anymore. (This is really funny for some reason.) \
Currently, we just tell users to always run Console.app in the background and then note down the time and copy-paste the logs once the error occurs. (We don't even tell them that you can select all of the (potentially millions) of logs with Command-A, so if they don't know that, they're in a bad spot. Lmao. I didn't think this through.)

I think this is clearly prohibitively hard to do. 

**-> If we want to effectively address hard-to-reproduce bugs, we need to make it easier for people to gather and send Console Logs.**

## How do we make it easier 
... for people to gather / send Diagnostic Data (primarily Console Logs)

### Steps (that need to happen)

So, in order for us to receive actionable Console Logs, the following things need to happen:

1. The user needs to enable production *and* storage of full, verbose logs
2. The user needs to reproduce the issue
3. The user needs to make a timestamp for when the issue occured (Otherwise we're unlikely to find the relevant logs among millions)
4. The user needs to retrieve the stored logs
5. The user needs to send the retrieved logs to us alongside the timestamp
6. The user needs to disable production and storage of full, verbose logs after they have sent the logs (as not to waste resources)

^ These things are invariants - they need to happen no matter how we design our system. 

The question is - how can we design our system such that these steps as simple as possible (without taking an unreasonable amount of time and effort for us to implement.)

### Enabling full, verbose logs

Currently we set a logLevel inside CocoaLumberJack. However, these logs end up in the unified system log, which has a separate log-level-based-filtering system. 

If we end up using the unified system log for storing logs (which seems like the best way, although I haven't investigated alternatives much), then this is redundant and nonsensical! 
For example, if might set the CocoaLumberJack logLevel to `debug`, (meaning the app produces `Debug` and `Info`) but if the OS logLevel is `default` then all `debug` and `info` messages would just be thrown away a

Here are all the ways to enable verbose logs that I know of:
- 
