# Error Logging Improvements Ideas for Mac Mouse Fix (October 2024)

> [!NOTE]
> This repo is just a note to jot down my ideas. Not sure why I'm creating a GitHub repo for this instead of just an Apple Note or something. I hope It's not a terrible idea to make 'Note Taking Repos'. Might be bad that we can't sort / filter them if we create a lot of different ones?

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

#### Implementation Details: How to change OS logLevel?

Quoting Quinn "The Eskimo" [1]:
```
You can customise [the OS logLevel] in [...] different ways:

- Add an OSLogPreferences property to your app’s Info.plist (all platforms).
- Run the log tool with the config command (macOS only)
- Create and install a custom configuration profile with the com.apple.system.logging payload (macOS only).
```

Which of these 3 approaches should we choose? 

1. Configuration Profile

Apple seems to use profiles (They sent me a profile for this purpose when I experienced a bug with iCloud)
However, configuration profiles take quite a few steps to install and they seem a bit scary. I think this might deter some users.

2. Info.plist

This would require users to download a separate 'debug build'. It would be nicer if they could just keep using their app normally. Also, I'm not totally sure how to automate changing the Info.plist content based on build type. (Probably using plistbuddy inside a build script should work though - that's how we increment the build number currently)

3. `log` command-line-tool

Having the user run terminal commands to change their system configuration is probably a bit scary and not ideal.

However, we could use this CLT directly from inside Mac Mouse Fix using NSTask. This would be the easiest UX - we could simply have a menu item in the app that says something like 'Start Collecting Logs for a Bug Report...'. 

(On Private Data)

However, I have not seen a way to disable stripping private data from logs using `log` (When using a Configuration Profile or Info.plist you can use the `Enable-Private-Data` key for this purpose.) 

But based on Quinn's post [2] it looks like a configuration profile or Info.plist are the only ways to enable private data, and therefore there's no way tod do this using `log`

However, Mac Mouse Fix basically has access to zero sensitive data (at least that I can think of?) except for the user's license key. So perhaps we could just tag all the logged data as `{public}` inside the app and we won't ever have to worry about data privacy, and we could just use `log` without a problem.

## Steps 2. to 6.

I'm getting to tired to write stuff in detail, but here are a few more thoughts I had:

2. The user needs to reproduce the issue
   -> This can't be simplified
3. The user needs to make a timestamp for when the issue occured (Otherwise we're unlikely to find the relevant logs among millions)
4. The user needs to retrieve the stored logs
5. The user needs to send the retrieved logs to us alongside the timestamp
6. The user needs to disable production and storage of full, verbose logs after they have sent the logs (as not to waste resources)
   -> 3 - 6. could all be turned into one or two steps for the user: Perhaps we could have a special status-bar-item while recording debug logs, and the user could click on that and then choose an option called: 'The Bug Just Happened!...' This would make a timestamp, then gather diagnostic data, and then automatically compose an email addressed to me with the diagnostic data as an attachment, and then turn the logging off. (We'd also have to inform the user about the privacy implications of sending the data, see Quinn's post [0] and the `sudo sysdiagnose` privacy message [11] for more). How could we implement step 4. (retrieving stored logs)? We could programmatically gather logs for the user using the `sudo sysdiagnose` command, this gathers extremely extensive diagnostic data and seems to be what Apple requests from users (and what Quinn recommends for debug hard-to-reproduce problems [0]). Alternatively, we can find some diagnostic data like crash reports directly in the library, and if we use another backend for CocoaLumberJack we could probably store logs in a custom file and then just read that. Sysdiagnose is super extensive, but its complications are that it takes a few minutes to gather the info, and it requires administrator priviledges. This wouldn't be the case for a different CocoaLumberJack backend I think, so that might be easier to implement. I can't really think of any other ways to do Step 5. (retrieving stored logs) I believe implementing the status-bar-item I mentioned might be quite hard. Alternatively, we could give the user step by step instructions - Apple did this for me when I had iCloud issues. Here are their instructions: (Their instruction about taking a screenshot to gather a timestamp for the bug is pretty smart.)
   1. Download the logging profile onto your Mac from the URL address below.
https://beta.apple.com/download/1017668
   2. Double click on the downloaded profile to install it. You will need to authenticate as your admin user.
   3. When the profile is installed (and you can see it installed in System Settings >Privacy & Security > Profiles), reboot your machine to make it take effectj.
   4. When you're logged back in, create a new note in Obsidian.
   5. Wait a few minutes for logs to accrue and then trigger a screenshot with Shift-Cmd-3. (The timestamp of the screenshot will help us know where in the log files to look for events.)
   6. Immediately after that, capture a new system diagnostics report by hitting Shift-Ctrl-Cmd-Opt-period to start.  The screen will flash white to show that it's started.
   7. When the sysdiagnose is complete (it may take a few minutes), Finder will open a window to the directory where the log file has been created in /var/tmp/. Attach that back to us, along with the screenshot.
   8. Turn off the extra logging by removing the debug logging profile in the Profiles pref pane, using the "-" button. Then reboot your machine again

# Sources

- [-1] Quinn "The Eskimo reply to "Xcode 15 Structured log always redacting <private> strings": https://developer.apple.com/forums/thread/738648?answerId=766975022#766975022
- [0] Quinn "The Eskimo" post "Using a Sysdiagnose Log to Debug a Hard-to-Reproduce Problem" https://developer.apple.com/forums/thread/739560
- [1] Quinn “The Eskimo" post "Your Friend the System Log": https://forums.developer.apple.com/forums/thread/705868
- [2] Quinn "The Eskimo" post "Recording Private Data in the System Log": https://developer.apple.com/forums/thread/705810
- [3] Apple Article "Generating Log Messages from Your Code": https://developer.apple.com/documentation/os/logging/generating_log_messages_from_your_code?language=objc
- [4] Apple Article "Viewing Log Messages": https://developer.apple.com/documentation/os/logging/viewing_log_messages?language=objc
- [5] Apple Article "Customizing Logging Behavior While Debugging": https://developer.apple.com/documentation/os/logging/customizing_logging_behavior_while_debugging?language=objc
- [6] WWDC 2016 Session 721 "Unified Logging and Activity Tracing" https://devstreaming-cdn.apple.com/videos/wwdc/2016/721wh2etddp4ghxhpcg/721/721_hd_unified_logging_and_activity_tracing.mp4?dl=1
- [7] `man 1 log`
- [8] `man 3 os_log` - overview and c api
- [9] `man 5 os_log` - logging configuration profiles
- [10] Apple Developer > Bug Reporting > Profiles and Logs: https://developer.apple.com/bug-reporting/profiles-and-logs/?platform=macos
- [11] `sysdiagnose` privacy message. Afaik, this shows up when running the command for the first time or when running it with the `-F` argument.
