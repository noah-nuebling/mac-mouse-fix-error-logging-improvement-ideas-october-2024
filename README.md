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

### Step 1: Enabling full, verbose logs (Elevating logLevel)

Currently, in Mac Mouse Fix, we set a logLevel inside CocoaLumberJack based on the type of build of Mac Mouse Fix (IIRC, the CCLJ logLevel is `default` for release builds and `debug` for `prerelease builds` which are builds that either have the `DEBUG` compiler flag set or that Have 'Beta' or 'Alpha' in their version string.)

#### Simplification: OS logLevel vs CocoaLumberJack logLevel
   However, these logs from CocoaLumberJack end up in the unified system log, which has a separate log-level-based-filtering system from CCLJ.
   Both systems have the same concept of hierarchical logLevels, and their names are also the same IIRC.
   Here's a description of the 4 logLevels in the unified logging system from `man log`:
      `level: {off | default | info | debug} The level is a hierarchy, e.g. debug implies debug, info, and default.`

   If we end up using the unified system log for storing logs (which seems like the best way, although I haven't investigated alternatives much), then the CCLJ logLevels are redundant! 
   For example, if we set the CocoaLumberJack logLevel to `debug`, but the OS logLevel is `default` then the app would produce `debug` and `info` messages but they would all immediately be thrown away by the unified system log, without being stored anywhere. If the situation was reversed, then the unified system log would be expecting `debug` and `info` messages but the app would never produce them.

   -> Therefore, (to simplify the process of enabling full, verbose logs for users) we should probably **synchronize the logLevel** between the app and the OS ('OS' meaning the unified system log)
   (That is, unless we end up using - instead of the unified logging system - some custom 'logging backend' for CocoaLumberJack (This custom backend would (perhaps among other things I can't remember rn) manage writing the logs to a file instead of the unified logging system managing that.)

  To 'synchronize' the log levels, and clean things up, we could do the following:
    - We could remove CocoaLumberJack from our codebase, and then redefine the CCLJ logging macros to invoke os_log() instead. E.g. DDLogDebug(...) would invoke os_log_debug(OS_LOG_DEFAULT, ...). These new macros would always cause some overhead, instead of being stripped by the compiler in release builds - which I think is how CocoaLumberJack works - but I believe the overhead would be very small, and the simplification might be worth it.

#### Simplification: No need for DEBUG builds

Currently, when compiling the app with the DEBUG compiler flag, this has primarily (purely?) 3 effects: 
1. Making it possible to attach a debugger
2. Doing some extra validation such as 'assert()'
3. Enabling verbose logging

When we currently distribute 'debug builds' to users with hard-to-reproduce issues, we build the app with the DEBUG flag in order to activate verbose logging. However, in these cases, we don't need the other features of the DEBUG flag - attaching a debugger and assert().

It seems better to control logging verbosity separately from the DEBUG compiler flag - this would then make it possible to enable verbose logging for any build of Mac Mouse Fix by just running a terminal command. (or clicking a button in the app, or installing a profile, more on that later.)

To make this a reality we need to make 2 changes to the codebase: 
- 1. The refactor propsed above, where we remove CocoaLumberJack logLevels and just rely on the logLevels that the unified logging system provides.
- 2. We already have a way to control logging-verbosity independent of the DEBUG flag it's called `if (runningPrerelease()) {`. Currently it checks the DEBUG flag as well as whether the app's version string contains 'Alpha' or 'Beta'. We could simply replace `runningPrerelease()` with a new function called something like `verboseLogsEnabled()` and define that in terms of `os_log_info_enabled(OS_LOG_DEFAULT)` and/or `os_log_debug_enabled(OS_LOG_DEFAULT)`. (These macros let us monitor the current logLevel that the unified logging system assigned to our process.)

-> With these 2 changes, the app's logging behavior would be fully controlled by the unified logging system! Meaning that we could enable full, verbose logs at runtime with a single `log config` terminal command (* with one caveat - stripping of private data - more on that later)

Here are all the ways to enable verbose logs that I know of:
- 
