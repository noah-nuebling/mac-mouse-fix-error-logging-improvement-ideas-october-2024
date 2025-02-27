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

### Steps

So, in order for us to receive actionable Console Logs, the following **steps** need to happen:

1. The user needs to enable production *and* storage of full, verbose logs
2. The user needs to reproduce the issue
3. The user needs to make a timestamp for when the issue occured (Otherwise we're unlikely to find the relevant logs among millions)
4. The user needs to retrieve the stored logs
5. The user needs to send the retrieved logs to us alongside the timestamp (through a private channel if the logs might contain sensitive data, and we also need to inform users about the privacy implications before they send the data)
6. The user needs to disable production and storage of full, verbose logs after they have sent the logs (as not to waste resources)

^ These things are invariants - they need to happen no matter how we design our system. 

The question is - how can we design our system such that these steps as simple as possible (without taking an unreasonable amount of time and effort for us to implement.)

### Step 1: Enabling full, verbose logs (Elevating logLevel)

Currently, in Mac Mouse Fix, we set a logLevel inside CocoaLumberJack based on the type of build of Mac Mouse Fix (IIRC, the CCLJ logLevel is `default` for release builds and `debug` for `prerelease builds` which are builds that either have the `DEBUG` compiler flag set or that Have 'Beta' or 'Alpha' in their version string.)

#### Simplification: Unifying OS logLevel with CocoaLumberJack logLevel
   However, these logs from CocoaLumberJack end up in the unified system log, which has a separate log-level-based-filtering system from CCLJ.
   Both systems have the same concept of hierarchical logLevels, and their names are also the same IIRC.
   Here's a description of the 4 logLevels in the unified system log from `man log`:
      `level: {off | default | info | debug} The level is a hierarchy, e.g. debug implies debug, info, and default.`

   If we end up using the unified system log for storing logs (which seems like the best way, although I haven't investigated alternatives much), then the CCLJ logLevels are redundant and they complicate things! 
   For example, if we set the CocoaLumberJzack logLevel to `debug`, but the OS logLevel is `default` then the app would produce `debug` and `info` messages but they would all immediately be thrown away by the unified system log, without being stored anywhere. (Update: Actually, `info` messages would get stored to memory first, and get written to disk when an `error` or `fault` message occurs - According to the WWDC 2016 Talk [6]) If the situation was reversed, then the unified system log would be expecting `debug` and `info` messages but the app would never produce them.

   -> Therefore, (to simplify the process of enabling full, verbose logs for users) we should probably **synchronize the logLevel** between the app and the OS ('OS' meaning the unified system log)
   (That is, unless we end up using - instead of the unified system log - some custom 'logging backend' for CocoaLumberJack (This custom backend would (perhaps among other things I can't remember rn) manage writing the logs to a file instead of the unified system log managing that.)

  To 'synchronize' the log levels, and clean things up, we could do the following:\
  1. We could remove CocoaLumberJack from our codebase, and use the unified system log directly. To do this we could simply redefine the CCLJ logging macros to invoke os_log() instead. E.g. DDLogDebug(...) would invoke os_log_debug(OS_LOG_DEFAULT, ...). These new macros might cause some additional overhead, since they wouldn't be stripped out by the compiler in release builds - which I think is what CocoaLumberJack does - but I believe the overhead would be very small.
  2. We could remove the dependency on the unified system log and use a custom logging backend for CocoaLumberJack instead. Then we could fully control the logging behavior by modifying the state of CocoaLumberJack while the app is running.
  3. We could configure our Info.plist to always enable the highest logLevel (`debug`) on the OS level, so that the OS would never filter our logs. (More on that below) Then we could control the logLevel using just the app-internal CocoaLumberJack logLevel.

#### Simplification: No need for DEBUG builds

Currently, when compiling the app with the DEBUG compiler flag, this has primarily (purely?) 3 effects: 
1. Making it possible to attach a debugger
2. Doing some extra validation such as 'assert()'
3. Enabling verbose logging

When we currently distribute 'debug builds' to users with hard-to-reproduce issues, we build the app with the DEBUG flag in order to activate verbose logging. However, in these cases, we don't need the other features of the DEBUG flag - attaching a debugger and assert().

It seems better to control logging verbosity separately from the DEBUG compiler flag - this would then make it possible to enable verbose logging for any build of Mac Mouse Fix by just running a terminal command. (or clicking a button in the app, or installing a profile, more on that later.)

To make this a reality, we'd have to say goodbye to using compiler flags to control logging behavior at all, and instead we'd have to use a fully dynamic 'logLevel'. We could do this by either managing logLevels internally by using CocoaLumberJack logLevels / using `static` variables inside the app, or we could simply rely on the OS logLevels that the unified system log assigns to our process.

To remove reliance on the DEBUG complier flag and let the app's logging be fully controlled by the OS we could make 2 simple changes to the codebase: 

- 1. The refactor propsed above, where we remove CocoaLumberJack alongside its logLevels in favour of `os_log()` - relying on the logLevels that the unified system log provides.
- 2. We already have a way to control logging-verbosity independent of the DEBUG flag it's called `runningPrerelease()`. Currently it checks the DEBUG flag as well as whether the app's version string contains 'Alpha' or 'Beta'. We could simply replace `runningPrerelease()` with a new function called something like `verboseLogsEnabled()` and define that in terms of `os_log_info_enabled(OS_LOG_DEFAULT)` and/or `os_log_debug_enabled(OS_LOG_DEFAULT)`. (These macros let us monitor the current logLevel that the unified system log assigned to our process.)

-> With these 2 changes, the app's logging behavior would be fully controlled by the unified system log! Meaning that we could enable/disable full, verbose logs at runtime with a single `log config` terminal command (* with one caveat - stripping of private data - more on that later)

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

The upside of using configuration profiles is that we could custom-craft them for the specific issue a user is experiencing - enabling verbose logs for all the processes that might be involved in the user's bug. 

... The only relevant process I could think of is `launchd` which has failed to start "Mac Mouse Fix Helper" in a myriad of ways over the years. So I feel like this flexibility to elevate logLevels for arbitrary processes isn't needed, (instead we could just hardcode elevated logLevels for relevant processes like `launchd`) But I'm not sure. 

2. Info.plist

If we we use this feature as intended, this would require users to download a separate 'debug build'. It would be nicer if they could just keep using their app normally. Also, I'm not totally sure how to automate changing the Info.plist content based on build type. (Probably using plistbuddy inside a build script should work though - that's how we increment the build number currently)

Alternatively we could just hardcode Mac Mouse Fix's logLevel to `debug` using the Info.plist and then use internal logLevels to control logging behavior.

-> HOWEVER, if we avoid modifying OS logLevels at all, (instead managing logLevels purely internally) we couldn't control the logLevel of other processes like `launchd` at all, which might be really helpful for some bugs! \
💡 This is a pretty strong argument to *not* manage logLevels internally and instead try to make the app adhere to OS logLevels, and then give users an as-easy-as-possible way to control OS logLevels.

3. `log` command-line-tool

Having the user run terminal commands to change their system configuration is probably a bit scary and not ideal.

However, we could use this CLT directly from inside Mac Mouse Fix using NSTask. This would be the easiest UX - we could simply have a menu item in the app that says something like 'Start Collecting Logs for a Bug Report...'. 

(On Private Data)

However, I have not seen a way to disable stripping private data from logs using `log` (When using a Configuration Profile or Info.plist you can use the `Enable-Private-Data` key for this purpose.) 

But based on Quinn's post [2] it looks like a configuration profile or Info.plist are the only ways to enable private data, and therefore there's no way tod do this using `log`

However, Mac Mouse Fix basically has access to zero sensitive data (at least that I can think of?) except for the user's license key. So perhaps we could just tag all the logged data as `{public}` inside the app and we won't ever have to worry about data privacy, and we could just use `log` without a problem.

## Steps 2. to 6.

I'm getting too tired to write stuff in detail, but here are a few more thoughts I had on the rest of the steps:

2. The user needs to reproduce the issue
   -> This can't be simplified
3. The user needs to make a timestamp for when the issue occured (Otherwise we're unlikely to find the relevant logs among millions)
4. The user needs to retrieve the stored logs
5. The user needs to send the retrieved logs to us alongside the timestamp
6. The user needs to disable production and storage of full, verbose logs after they have sent the logs (as not to waste resources)\
   -> 3 - 6. could all be turned into one or two steps for the user: Perhaps we could have a special status-bar-item while recording debug logs, and the user could click on that and then choose an option called: 'The Bug Just Happened!...' This would make a timestamp, then gather diagnostic data, and then automatically compose an email addressed to me with the diagnostic data as an attachment, and then turn the logging off. (We'd also have to inform the user about the privacy implications of sending the data, see Quinn's post [0] and the `sudo sysdiagnose` privacy message [11] for more). How could we implement step 4. (retrieving stored logs)? We could programmatically gather logs for the user using the `sudo sysdiagnose` command, this gathers extremely extensive diagnostic data and seems to be what Apple requests from users (and what Quinn recommends for debug hard-to-reproduce problems [0]). Alternatively, we can find some diagnostic data like crash reports directly in the library, and if we use another backend for CocoaLumberJack we could probably store logs in a custom file and then just read that. Sysdiagnose is super extensive, but its complications are that it takes a few minutes to gather the info, and it requires administrator priviledges. This wouldn't be the case for a different CocoaLumberJack backend I think, so that might be easier to implement. I can't really think of any other ways to do Step 5. (retrieving stored logs) (Update: Actually we could use `log show` or `log collect` or `OSLogStore` programmatically as alternatives. More on that below.) I believe implementing the status-bar-item I mentioned might be quite hard. Alternatively, we could give the user step by step instructions - Apple did this for me when I had iCloud issues. Here are their instructions: (Their instruction about taking a screenshot to gather a timestamp for the bug is pretty smart.)
   1. Download the logging profile onto your Mac from the URL address below.
https://beta.apple.com/download/1017668
   2. Double click on the downloaded profile to install it. You will need to authenticate as your admin user.
   3. When the profile is installed (and you can see it installed in System Settings >Privacy & Security > Profiles), reboot your machine to make it take effectj.
   4. When you're logged back in, create a new note in Obsidian.
   5. Wait a few minutes for logs to accrue and then trigger a screenshot with Shift-Cmd-3. (The timestamp of the screenshot will help us know where in the log files to look for events.)
   6. Immediately after that, capture a new system diagnostics report by hitting Shift-Ctrl-Cmd-Opt-period to start.  The screen will flash white to show that it's started.
   7. When the sysdiagnose is complete (it may take a few minutes), Finder will open a window to the directory where the log file has been created in /var/tmp/. Attach that back to us, along with the screenshot.
   8. Turn off the extra logging by removing the debug logging profile in the Profiles pref pane, using the "-" button. Then reboot your machine again

## Other

Stuff I picked up from watching the WWDC 2016 Session: [6] which was unclear to me before:
- The os_log docs speak about logs that are stored to disk and logs that are stored to memory. As far as I understand the purpose of the 'stored in memory' messages is that they do also get written to disk IF an `OS_LOG_TYPE_ERROR` or `OS_LOG_TYPE_FAULT` message is sent afterwards. OS_LOG_TYPE_ERROR denotes process-level errors, while OS_LOG_TYPE_FAULT denotes errors on the scope of multiple processes. OS_LOG_TYPE_FAULT might cause even more extensive debug information to be saved than OS_LOG_TYPE_ERROR Quote: "Use `os_log_error` to cause additional information capture from app" "Use `os_log_fault` to cause additional information capture from system". Perhaps we should think of FAULT and ERROR as explicit ways to trigger such information capture and use default log level for 'normal' errors.
- Not only strings are private by default: "Dynamic strings, collections, and objects are assumed to be private."

Update (next day, Oct 23 2024): 

Another thing I haven't looked much into is exporting logs from inside the app. 
If we use the unified system log we could
- Use `sudo sysdiagnose -u` programmatically - but it would require the user to enter the admin password and would take a few minutes.
- Use OSLogStore API [13], but it only seems to be able to retrieve logs from the current process unless you're on macOS 12.0 or later where there's an option to retrieve the entire system log.
- Use the `log collect` or `log show` command line tool [7]  programmatically. It should be able to the same things as `OSLogStore` but it's perhaps more powerful. It doesn't even require `sudo` IIRC.
- -> If we can retrieve the system log using the `log` clt or `OSLogStore`, then we could just also programmatically gather crash reports from the library and that should be all the debug info we need. The other stuff inside a full sysdiagnose archive doesn't seem too useful (but I haven't thought about this much, so maybe it's better to heir on the side of collecting more information - which sysdiagnose would do? ... sysdiagnose also collects the attached USB Devices, which might be useful? so we can see what mouse the user is using ... but the user could also just tell us or we could gather that info separately inside MMF through IOKit...). However, I heard a few times that Apple encourages collecting sysdiagnose reports because, if it turns out to be a bug on Apple's side you can just send them the sysdiagnose and they'll have all the info to debug things.

Alsooo, we might want to think about the problem in terms of **things we could optimize for**

Basically, there are 3 things we wanna optimize the bug-reporting-process for:

1. Optimize to receive maximally detailed information (If we fully optimize for this, that would see us being able to enable verbose logs for arbitrary external processes like `launchctl` and then use `sysdiagnose` to extract all diagnostic info the system can provide.)
2. Optimize for the easiest possible user experience (If we fully optimize for that, we would probably have a simple toggle in the app to enable verbose logging, which would create a menu-bar-item with a "The bug just occured! Send the Info..." button, which would handle everything automatically - so the user might just have to click 3 or 4 UI items (measured without the reproduction of the issue) to complete the entire process. 
3. Optimize for development time and stability (If we fully optimize for this we would just do nothing, since that's the least bug-prone and time-consuming option.)

Now we could try to find the optimal balance between these 3 optimization points. But we could also do a sort of 'hybrid approach' or 'gradual approach'. 
For example, perhaps we could have a UX-optimized process be the default. And for cases where that doesn't provide enough information, we can implement a second information-optimized approach. Or we could roll out the information-optimized approach first (since that should also be relatively easy to implement) and then roll out a UX-optimized approach if we find that we still don't get enough actionable feedback.

Or we could try to find a good compromise between the 3 optimization-points and then adjust it based on how it's working.

... I'd have to think about this some more.

# Sources

- [-1] Quinn "The Eskimo reply to "Xcode 15 Structured log always redacting <private> strings": https://developer.apple.com/forums/thread/738648?answerId=766975022#766975022
- [0] Quinn "The Eskimo" post "Using a Sysdiagnose Log to Debug a Hard-to-Reproduce Problem" https://developer.apple.com/forums/thread/739560
- [1] Quinn “The Eskimo" post "Your Friend the System Log": https://forums.developer.apple.com/forums/thread/705868
- [2] Quinn "The Eskimo" post "Recording Private Data in the System Log": https://developer.apple.com/forums/thread/705810
- [3] Apple Article "Generating Log Messages from Your Code": https://developer.apple.com/documentation/os/logging/generating_log_messages_from_your_code?language=objc
- [4] Apple Article "Viewing Log Messages": https://developer.apple.com/documentation/os/logging/viewing_log_messages?language=objc
- [5] Apple Article "Customizing Logging Behavior While Debugging": https://developer.apple.com/documentation/os/logging/customizing_logging_behavior_while_debugging?language=objc
- [6] WWDC 2016 Session 721 "Unified Logging and Activity Tracing" https://devstreaming-cdn.apple.com/videos/wwdc/2016/721wh2etddp4ghxhpcg/721/721_hd_unified_logging_and_activity_tracing.mp4
- [7] `man 1 log`
- [8] `man 3 os_log` - overview and c api
- [9] `man 5 os_log` - logging configuration profiles
- [10] Apple Developer > Bug Reporting > Profiles and Logs: https://developer.apple.com/bug-reporting/profiles-and-logs/?platform=macos
- [11] `sysdiagnose` privacy message. Afaik, this shows up when running the command for the first time or when running it with the `-F` argument.
- [12] CocoaLumberJack Discussion "Should everyone migrate to OSLog"? https://github.com/CocoaLumberjack/CocoaLumberjack/discussions/1363
- [13] OSLogStore Documentation https://developer.apple.com/documentation/oslog/oslogstore?language=objc


---

# Update [Feb 2025] - Short Term Solution

While we haven't implemented a better debugging system, yet, we're using some of the learnings from above to debug hard-to-reproduce bugs in the short-term.

We do that by creating and distributing special 'debug builds' that produce and store verbose logs.

## 1. Create a 'verbose-debug-logging' build of the app

### 1.1) Set the DDLogLevel 

-> to DDLogLevelAll / .all (for both Swift and objc)

### 1.2) Maybe add (Debug) to the Version String

- ... So you can more easily differentate the debug version.
- Also helps users not accidentally keep using the debug build long-term.
- E.g. `3.0.4 Beta 1 (Debug)`

### 1.3) Remove the NDEBUG flags

From the Swift `SWIFT_ACTIVE_COMPILATION_CONDITIONS` and the C `GCC_PREPROCESSOR_DEFINITIONS`.

-> That way assert() calls get enabled.
	(Actually, isn't it better to not have assert() crash the app so it behaves exactly like a non-'verbose-debug-logging' build?)

### 1.4) Add this dict to Info.plist under the `OSLogPreferences` key.
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>com.nuebling.mac-mouse-fix</key>
	<dict>
		<key>DEFAULT-OPTIONS</key>
		<dict>
			<key>Signpost-Enabled</key>
			<false/>
			<key>Enable-Private-Data</key>
			<false/>
			<key>Signpost-Scope</key>
			<string>Process</string>
			<key>Enable-Oversize-Messages</key>
			<true/>
			<key>Signpost-Persisted</key>
			<false/>
			<key>Level</key>
			<dict>
				<key>Persist</key>
				<string>Debug</string>
				<key>Enable</key>
				<string>Debug</string>
			</dict>
			<key>Signpost-Backtraces-Enabled</key>
			<false/>
		</dict>
	</dict>
</dict>
</plist>

```
Notes:
- This assumes that all logs from mainApp and helperApp are using the `com.nuebling.mac-mouse-fix` subsystem.
  - -> Make sure to use that subsystem when setting up the DDOSLogger.
- You can simply copy-paste this into an Info.plist in the Xcode project editor!
- This `OSLogPreferences` dict is 'fully specified' with all the options visible at `man 5 os_log` as of [Feb 2025]
  - We're enabling and persisting all logs to disk
  - We strip 'private' data. (Not sure about this. But it's better to strip 'private' data if we don't need it for debugging I guess.)
  - We enable 'Oversize' messages. (Not sure about this, but we don't want long logs to be cut off, I guess?)
  - We disable all performance analytics (signpost)

Testing Notes [Feb 2025] [macOS 15.3 (24D60)] [MMF build 23676 (right after 3.0.4 Beta 1)]
 - Add the `OSLogPreferences` dict both for mainApp and helperApp Info.plist's – Otherwise it doesn't seem to work for both apps. 
 - When you check the logLevel using `log config --status`, it doesn't seem to be affected by Info.plist – however, the Info.plist dict logLevels still seem to apply in practice when checking `log show`.
- Commands:
  - I used `log config` to check the 'official' (?) logLevels:
    `sudo log config --status --subsystem com.nuebling.mac-mouse-fix`
  - I used `log show` to see all the (persisted?) logs since the last boot:
    `log show --debug --info --last boot --predicate 'subsystem == "com.nuebling.mac-mouse-fix"'`

## 2. Export and Test

- Archive and export the new debug build – Use the 'Release' configuration with optimizations enabled, so app doesn't run slow, (and cause we don't gotta attach a debugger.)
- Perhaps verify that verbose logs appear in the `log show` command (see above) before distributing.
  - Afaik, even at low log levels, high-level logs (like debug and info) are still persisted and possibly show up in `log show`, if they are followed by an error or fault log. Keep that in mind. Wrote more about this somewhere above.

Other:
- Afterwards, simply `git stash`. Don't think we need these debug builds in our git history.
- dsym file for symbolication should be available in Xcode organizer for the forseeable future.

## 3. Distribute

First we can upload the debug build (e.g. to MegaUpload), and then we send the user an email, containing a download link and instructions on how to gather diagnostics. 

(We need MegaUpload or similar, because Gmail doesn't allow attaching .app files – not even when zipped – due to security reasons.)

Here's an example email (`message:<A184CC1B-1128-4F49-BBB4-A4939504E2FE@gmail.com>`)
```
Hello <redacted>,

Thanks for the details, and for being willing to help debug this! 
I've created a special debug version of the app. What's special about it, is that it creates very detailed logs about everything it's doing. This will help track down what's happening when the scrolling issue occurs.

Here's what to do:

1. Install, and enable the special debug build, which you can download <fill in link>.
2. When the scrolling issue occurs:
	- Immediately take a screenshot (Shift-Command-3) – the screenshot's timestamp will help me navigate the (very numerous) logs. (Possibly millions!)
	- Then press Control-Option-Shift-Command-Period to collect the recorded logs and other system diagnostics (This is called sysdiagnose)
       	- The screen will flash white to indicate that the sysdiagnose has started.
3. When the sysdiagnose is done (might take a few minutes), the resulting sysdiagnose file will be revealed in Finder. It will be placed in the /var/tmp/ folder.
4. Please send both the screenshot and the sysdiagnose files to me.
5. To stop detailed logs from being created (Which might take up unnecessary CPU resources and disk space), simply go back to using a regular release of Mac Mouse Fix instead of the special debug build I've sent you.

About Privacy:
The sysdiagnose file contains very detailed information about your system and the activity of the software that has been running on it recently. Personal information is normally stripped from a sysdiagnose file, but I'm not sure if some personal info could slip through if software developers make a mistake. (I assume that can happen, but I haven't looked super deeply into this). In any case, I will not share your sysdiagnose file with anyone else without your express permission. If you're not comfortable with this, I also understand that of course!

If you do decide to send the sysdiagnose file and the screenshot, that will help me understand exactly what's happening on your system, when scrolling stops working.

Thanks again for your help!

Best
Noah
```

Here's a later example message we posted as a [GitHub comment](https://github.com/noah-nuebling/mac-mouse-fix/issues/1299#issuecomment-2687638866):
```
Hey @jzetterman, thanks for the quick reply.

After thinking about this more carefully, I don't think it would be caused by conflicts with other software. The behavior you're describing (Settings temporarily reverting to default, but then automatically going back to your custom settings) is quite unusual.

First some quick questions:
1. How frequently does this happen? Every sleep/wake cycle, or just sometimes?
2. Do you notice any pattern to when it occurs?

I've tried to reproduce this with my MX Master 2S connected via Bluetooth, and sleep/wake cycles – but I couldn't reproduce the bug. Therefore, it's hard to figure out what's going on. 

If you'd like, you can help investigate this further:

I've created a special debug version of the app. What's special about it, is that it creates very detailed logs about everything it's doing. This will help track down what's happening when the issue occurs.

Here's what to do:

1. Install, and enable the special debug build, which you can [download here](https://mega.nz/file/W5pnCbJJ#dg4kIAwQ1nsdmw9xzJODkaRXW7aQkrei6C6-2Gc0Wnc).
2. When the issue occurs, and you notice that the settings have reverted:
    - Immediately take a screenshot (Shift-Command-3) – the screenshot's timestamp will help me navigate the (very numerous) logs.
    - Then press Control-Option-Shift-Command-Period to collect the recorded logs and other system diagnostics (This is called sysdiagnose)
    - The screen will flash white to indicate that the sysdiagnose has started.
3. When the sysdiagnose is done (might take a few minutes), the resulting sysdiagnose file will be revealed in Finder. It will be placed in the /var/tmp/ folder.
4. Please send both the screenshot and the sysdiagnose files to me.
    - You should probably [send them via email](http://redirect.macmousefix.com/?target=mailto-noah), not via GitHub. That's because the sysdiagnose file might contain personal data. See the 'About Privacy' section below.
5. To stop detailed logs from being created (Which might take up unnecessary CPU resources and disk space), simply go back to using a regular release of Mac Mouse Fix instead of the special debug build I've linked here.

About Privacy:
The sysdiagnose file contains very detailed information about your system and the activity of the software that has been running on it recently. Personal information is normally stripped from a sysdiagnose file, but I'm not sure if some personal info could slip through if software developers make a mistake. (I assume that can happen, but I haven't looked super deeply into this). In any case, I will not share your sysdiagnose file with anyone else without your express permission. If you're not comfortable with this, I also understand that of course!

If you do decide to send the sysdiagnose file and the screenshot, that will help me understand exactly what's happening on your system, when the bug occurs.

Thank you for your help!
```
