<p align="right"><sub>October 8, 2026</sub></p>

# Investigating `/tmp` on macOS: How I Traced a Runaway LaunchAgent

Temporary directories are easy to overlook. Most applications create temporary files as part of their normal operation, and users rarely need to interact with them. However, temporary directories can also provide useful information during a security investigation. Misconfigured applications may leave sensitive information behind, improperly secured files can introduce security risks, and attackers sometimes use writable temporary locations to stage tools or payloads.

That is why I decided to investigate the `/tmp` directory on my MacBook, Waddles, which I hadn't restarted in almost a week. My initial goal was simply to see what had accumulated there and whether anything deserved closer inspection.

During my investigation, I discovered that a forgotten LaunchAgent I had created months ago was repeatedly launching a malfunctioning Chrome Remote Desktop process. By the time I found it, `launchd` had recorded over 57,000 executions, and the process had generated a log containing more than 115,000 lines.

This write-up documents how I found the issue, traced it to its source, and verified that disabling the responsible service stopped the log growth.

## 1. Understanding macOS Temporary Directories

I began by checking Waddles' uptime and temporary directories.

```
uptime
ls -ld /tmp /private/tmp /private/var/tmp
readlink /tmp
echo "$TMPDIR"
```

The machine had been running for approximately **6 days and 22 hours**.

One of the first things I confirmed was that `/tmp` is not a separate directory on macOS. It is a symbolic link pointing to `/private/tmp`.

```
/tmp -> private/tmp
```

I also discovered that macOS uses several locations for temporary storage:

- `/private/tmp` is the traditional shared temporary directory. 
- `/private/var/tmp` is intended for temporary data that may need to persist longer, including across reboots. 
- **$TMPDIR** points to a user-specific temporary directory, typically somewhere under `/var/folders/`.

In my case, $TMPDIR pointed to a directory resembling:

```
/var/folders/.../T/
```

I compared their disk usage:

```
du -sh /private/tmp
du -sh "$TMPDIR"
du -sh /private/var/tmp
```

And the results were:

|Directory|Disk usage|
|---|---|
|`/private/tmp`|13 MB|
|$TMPDIR|119 MB|
|`/private/var/tmp`|176 KB|

My user-specific temporary directory occupied considerably more space than the traditional `/tmp`.

I also counted the immediate entries in each directory:

```
find /private/tmp -mindepth 1 -maxdepth 1 | wc -l
find "$TMPDIR" -mindepth 1 -maxdepth 1 | wc -l
find /private/var/tmp -mindepth 1 -maxdepth 1 | wc -l
```

Results:

```
/private/tmp:       10
$TMPDIR:          2193
/private/var/tmp:    6
```

Most entries in $TMPDIR appeared to belong to applications I recognized, particularly OpenAI Codex.

This illustrated an important point: **examining only `/tmp` does not provide a complete picture of temporary-file activity on macOS.**

### File permissions and the sticky bit

I also inspected the directory permissions.

```
drwxrwxrwt  root  wheel  /private/tmp
drwx------  user  staff  $TMPDIR
```

`/private/tmp` is _world-writable_, meaning ordinary users can create files there, but its permission string ends with `t`, indicating that the _sticky bit_ is enabled. The sticky bit restricts users from deleting or renaming files belonging to other users, subject to ownership and privilege rules, which is important for shared temporary directories. Without the sticky bit, a user could potentially remove another user's temporary files simply because both users have write permission on the containing directory.

- The sticky bit does not prevent users from reading files, since read access is still controlled by the permissions on individual files and directories.

My $TMPDIR had permissions of `700`, restricting access to my user account under ordinary UNIX permission rules.

## 2. Enumerating Files and Identifying Potentially Interesting Artifacts

Next, I examined the contents of `/private/tmp`.

```
ls -lah /private/tmp
```

The directory contained several recognizable artifacts, including:

```
codex-browser-use/
com.openai.sky.CUAService/
crd-waddles-error.log
crd-waddles.log
dumps/
logitech_kiros_agent-...
logitech_kiros_updater
node-compile-cache/
powerlog/
steam.pipe
```

I used the `file` command to identify the types of these entries.

```
file /private/tmp/*
```

Among the results were regular text files, directories, UNIX domain sockets, and a FIFO (named pipe).

The Logitech entries were sockets associated with Logi Options+, while `steam.pipe` was a named pipe.

These special files are worth distinguishing from ordinary files. UNIX domain sockets support communication between local processes, while named pipes provide another form of interprocess communication (IPC). Their presence in a temporary directory is not inherently suspicious, though.

### Checking for executable files and symbolic links

Temporary directories are interesting from a security perspective because attackers who already have access to a system may use writable locations to store scripts or other payloads, which was one reason I wanted to inspect my own temporary directories.

I began by searching for executable files...

```
find /private/tmp "$TMPDIR" -type f -perm -111 -print
```

...which returned no results.

One limitation of this command is that `-perm -111` requires all three execute-permission bits to be present. Therefore, the empty result does not establish that no executable files exist. A more comprehensive search would test the owner, group, and other execute bits individually.

I also looked for symbolic links:

```
find /private/tmp "$TMPDIR" -type l -ls
```

Several results were named `SingletonCookie`, inside application-specific directories associated with Chromium apps, which were consistent with my ordinary usage.

- Symbolic links can be relevant to temporary-file vulnerabilities when applications handle predictable filenames insecurely.

### Checking world-writable entries

I then searched for entries with the write bit enabled for other users.

```
find /private/tmp "$TMPDIR" -perm -0002 -ls
```

One result stood out:

```
drwxrwxrwx  user  wheel  /private/tmp/dumps
```

Unlike `/private/tmp` itself, this directory had no sticky bit.

I inspected it further:

```
ls -laeO@ /private/tmp/dumps
du -sh /private/tmp/dumps
find /private/tmp/dumps -maxdepth 2 -ls
```

The directory was empty and contained an extended attribute named:

```
org.chromium.crashpad.database.initialized
```

This suggested that it was associated with Chromium's Crashpad crash-reporting infrastructure. Although its permissions were unusually permissive, I found no sensitive files inside it and no evidence of an exploitable vulnerability.

- This was a useful reminder that unusual permissions warrant investigation (but do not automatically indicate a security incident).

## 3. Discovering an Unexpectedly Large Log File

While reviewing the directory, I noticed that one file accounted for almost all the space used by `/private/tmp`.

```
-rw-r--r--  user  wheel  12M  crd-waddles-error.log
```

The filename suggested a connection to Chrome Remote Desktop.

I inspected its metadata and contents:

```
stat /private/tmp/crd-waddles-error.log
file /private/tmp/crd-waddles-error.log
wc -l /private/tmp/crd-waddles-error.log
tail -n 20 /private/tmp/crd-waddles-error.log
```

The log contained **115,714 lines** and was approximately **12.7 MB** in size.

Its final entries contained messages resembling:

```
Host started for user <username> ...
Daemon is disabled.

Host started for user <username> ...
Daemon is disabled.
```

The same two messages appeared repeatedly, roughly every ten seconds, and the log's creation timestamp was October 1—shortly after the system had started.

My initial hypothesis was that Chrome Remote Desktop was repeatedly attempting to start despite being disabled, and the next question was whether a single process was producing these messages or whether a service manager was continuously restarting the process.

## 4. Identifying the Process Responsible for the Log

I used `lsof` to determine which process had the log open.

```
sudo lsof /private/tmp/crd-waddles-error.log
```

The output identified a process named `remoting_me2me_host_service`.

```
COMMAND     PID    USER    FD    TYPE    NAME
remoting_    35103  user    2u    REG     /private/tmp/crd-waddles-error.log
```

I also searched for related processes:

```
pgrep -fl 'remoting|chrome.*remote|crd'
```

This revealed the full executable path:

```
/Library/PrivilegedHelperTools/ChromeRemoteDesktopHost.app/
Contents/MacOS/remoting_me2me_host_service --run-from-launchd
```

The process indeed belonged to Chrome Remote Desktop.

To investigate its behavior, I sampled its PID repeatedly:

```
for i in {1..20}; do
    date '+%H:%M:%S'
    pgrep -fl remoting_me2me_host_service
    sleep 2
done
```

During this observation period, the PID (`35363`) remained unchanged. However, an earlier instance of the same service (PID `35103`) had terminated by the time I checked it again. This suggested that two mechanisms might be involved: the service could retry internally, while `launchd` could restart it after termination.

I needed to investigate how the process was launched.

## 5. Investigating Persistence with `launchd`

On macOS, `launchd` is responsible for managing many background services and processes. It uses configuration files called _property lists_, or `.plist` files, to describe how jobs should start and under what conditions they should run. _LaunchAgents_ generally operate in user-session contexts, while _LaunchDaemons_ typically provide system-level background services. Because these mechanisms can automatically start programs, they are relevant to both legitimate system administration and persistence investigations.

I searched the usual third-party LaunchAgent and LaunchDaemon locations for references to Chrome Remote Desktop:

```
grep -RilE \
'ChromeRemoteDesktop|remoting_me2me_host_service|ChromeRemoteDesktopHost' \
"$HOME/Library/LaunchAgents" \
/Library/LaunchAgents \
/Library/LaunchDaemons \
2>/dev/null
```

Three files appeared:

```
~/Library/LaunchAgents/com.egesu.crd-workaround.plist
/Library/LaunchAgents/org.chromium.chromoting.plist
/Library/LaunchDaemons/org.chromium.chromoting.broker.plist
```

The first was particularly interesting: it was a custom LaunchAgent that I had apparently created earlier that summer, when I had first used Chrome Remote Desktop on Waddles, and forgotten about.

I inspected it using:

```
plutil -p ~/Library/LaunchAgents/com.egesu.crd-workaround.plist
```

Its important configuration values were:

```
KeepAlive          = true
RunAtLoad          = true
ThrottleInterval   = 10

StandardErrorPath  = /tmp/crd-waddles-error.log
StandardOutPath    = /tmp/crd-waddles.log
```

The LaunchAgent was configured to execute Chrome Remote Desktop's host service:

- `RunAtLoad` instructed `launchd` to start the job when loaded.
- `KeepAlive` instructed it to maintain the job's execution, restarting it when necessary.
- `ThrottleInterval` limited how frequently `launchd` would attempt to restart a failing job.
- Lastly, `StandardErrorPath` pointed directly to the log file I had been investigating.

**I had found the configuration responsible for launching the process and directing its error output into `/tmp`.** 😠

### Checking the LaunchAgent's status

I queried the job using `launchctl`:

```
launchctl print gui/$(id -u)/com.egesu.crd-workaround
```

The output included:

```
state = running
runs = 57869
last exit code = 1

properties = keepalive | runatload | ...
```

Here was the most revealing result of the investigation.

According to `launchd`, the custom job had been executed **57,869 times**, and its last recorded exit code was `1`, indicating an unsuccessful termination.

Combined with the configuration's `KeepAlive` setting, this was strong evidence that `launchd` had repeatedly restarted a failing process... though it still did not explain why Chrome Remote Desktop was failing.

## 6. Tracing the Chrome Remote Desktop Failure

I examined the Chrome Remote Desktop configuration and checked whether its official services were registered in the relevant `launchd` domains.

```
launchctl print gui/$(id -u)/org.chromium.chromoting
```

and:

```
sudo launchctl print system/org.chromium.chromoting.broker
```

Both queries reported that the specified services could not be found. The configuration files existed on disk, but the corresponding services were not loaded in the domains I checked.

I then returned to the Chrome Remote Desktop log and examined the more detailed error messages.
Among them were:

```
Cannot connect to IPC through server name
chromoting.agent_process_broker_mojo_ipc.
Endpoint is invalid.

Failed to connect to agent process broker.

SendHostOfflineReason: sending INITIALIZATION_FAILED.
```

The host process was attempting to communicate with its agent process broker through an IPC endpoint, but the connection failed.

The log also showed:

```
Host returned non-permanent exit code
INITIALIZATION_FAILED (2)
Host ran for 2.68581 s
Host failure count 1/10
Relaunching in 57.3142 s
```

Now _this_ clarified the behavior.

The custom LaunchAgent was configured to keep the Chrome Remote Desktop service running. Meanwhile, Chrome Remote Desktop itself could retry failed host initialization internally. I was observing different parts of this behavior at different times.

So, the chain looked like this:

```
Custom LaunchAgent
        |
        v
launchd starts Chrome Remote Desktop
        |
        v
Host attempts initialization
        |
        v
IPC connection to broker fails
        |
        v
INITIALIZATION_FAILED
        |
        v
Host retries or terminates
        |
        v
launchd may restart the job
        |
        v
More messages written to /tmp
```

The actual reason the official broker was unavailable was (and still is, if I need to be honest) a mystery. At least the repeated launch attempts and the source of the log output were clear.

## 7. Disabling the LaunchAgent and Verifying the Result

Before changing anything, I preserved a copy of the log.

```
cp /private/tmp/crd-waddles-error.log \
   ~/Desktop/crd-waddles-error-before-fix.log
```

I then unloaded the custom LaunchAgent:

```
launchctl bootout \
  gui/$(id -u) \
  ~/Library/LaunchAgents/com.egesu.crd-workaround.plist
```

To verify that it was no longer loaded, I ran:

```
launchctl print gui/$(id -u)/com.egesu.crd-workaround
```

The response indicated that the service could no longer be found. Nice.

I also checked for the Chrome Remote Desktop host process:

```
pgrep -fl remoting_me2me_host_service
```

No matching process was returned.

### Measuring log growth before and after

Before disabling the LaunchAgent, I had measured how much the log grew during a 30-second period.

The results were:

```
Bytes added in 30s: 2444
Lines added in 30s: 20
```

I repeated the measurement after unloading the LaunchAgent.

```
LOG=/private/tmp/crd-waddles-error.log

before=$(stat -f '%z' "$LOG")
lines_before=$(wc -l < "$LOG")

sleep 30

after=$(stat -f '%z' "$LOG")
lines_after=$(wc -l < "$LOG")

echo "Bytes added: $((after-before))"
echo "Lines added: $((lines_after-lines_before))"
```

This time, the output was:

```
Bytes added: 0
Lines added: 0
```

The log had finally stopped growing, which strongly suggested that the custom LaunchAgent was responsible for the repeated Chrome Remote Desktop activity and associated logging.

Finally, I renamed the configuration file:

```
mv ~/Library/LaunchAgents/com.egesu.crd-workaround.plist \
   ~/Library/LaunchAgents/com.egesu.crd-workaround.plist.disabled
```

I deliberately kept the file instead of deleting it, since it was useful evidence of the original configuration (and I like to keep logs of everything).

## 8. Security Implications and Lessons Learned

I did not discover malware or confirm an exploitable vulnerability, thankfully, but the investigation highlighted several security concepts.

### Temporary-file permissions

The Chrome Remote Desktop error log had permissions of `644`:

```
-rw-r--r--
```

Under normal UNIX permission rules, other local users could read this file. It matters because logs can contain information about application configurations, usernames, internal errors, and occasionally sensitive identifiers. In fact, the Chrome Remote Desktop log contained account information and signaling identifiers.

Applications should not unnecessarily expose diagnostic information through shared temporary directories.

### Persistence mechanisms

LaunchAgents are a legitimate macOS feature. However, the same mechanisms that allow administrators and applications to start background services automatically can also be abused for persistence. Finding an unfamiliar LaunchAgent is therefore a reason to investigate its executable path, configuration, owner, and behavior. In Waddles' case, the LaunchAgent was my own forgotten workaround rather than an unauthorized persistence mechanism. Nevertheless, this experiment is an example of how a legitimate autostart configuration can continue affecting a system long after its original purpose has been _forgotten_. 

### A large log is a symptom rather than a root cause

Initially, the 12 MB log seemed to be a problem. Tracing the file to its writing process and then to the responsible LaunchAgent helped me identify why the log kept growing. Deleting it would have temporarily recovered disk space, however little, but the underlying behavior would have been left undiscovered.

## Conclusion

I started this investigation hoping to find something interesting among the temporary files accumulated during a week of macOS uptime. Instead, I discovered a forgotten custom LaunchAgent that had executed more than 57,000 times and generated a repetitive Chrome Remote Desktop error log. After disabling the agent, the Chrome Remote Desktop process disappeared and the log stopped growing during the verification period. If only I had collected CPU/memory/power measurements before disabling it. I wonder if it had any significant effect on system performance.

At least the investigation helped me understand how macOS separates temporary storage (since I had gotten the idea after learning about the `/tmp` directory on Linux), how UNIX permissions protect shared directories, and why persistence mechanisms deserve attention even when they belong to _legitimate software_.

---

### Commands used

|Command|Purpose|
|---|---|
|`uptime`|Check system uptime|
|`ls -lah`|List files, ownership, permissions, and sizes|
|`du -sh`|Measure directory disk usage|
|`find`|Search files by type and permissions|
|`file`|Identify filesystem object types|
|`stat`|Inspect file metadata|
|`wc -l`|Count log lines|
|`head` / `tail`|Inspect portions of logs|
|`lsof`|Identify processes using files|
|`pgrep`|Locate running processes|
|`ps`|Inspect process information|
|`plutil -p`|Read macOS property-list configurations|
|`launchctl print`|Inspect launchd services|
|`launchctl bootout`|Unload a launchd job|
