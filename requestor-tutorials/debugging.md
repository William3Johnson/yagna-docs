---
description: >-
  Introducing log files as a tool to identify and fix issues in your requestor
  applications.
---

# Debugging with the use of log files

When developing applications on Golem, sooner or later you will get to a point where something is not working as expected. Executing commands on the provider-hosted container seems to be one of the most error-prone areas. If something is going wrong on the provider side, we need to have a tool that helps us in the identification of the problem. When we know what the error details are, it usually is much easier to correct our code and eliminate the bug.

## Introducing log files

The good news is that Golem's high-level API supports log files. The log files contain all sorts of interesting stuff. Among others you will find there:

* Requestor &gt; Provider file transfer details
* Output streams of the commands executed on the Provider: `stdout` and `stderr`
* Provider &gt; Requestor file transfer details

## How to enable logging?

The logging infrastructure in Golem's high-level API is quite universal and thus somewhat complex, but for most of the cases using the `SummaryLogger` will do the job.

{% hint style="info" %}
The `SummaryLogger`is an event listener that listens to all the atomic events and combines them into the more aggregated track of events that is easier to grasp by humans.
{% endhint %}

{% tabs %}
{% tab title="Python" %}
To enable logging to a file we need two things.

```python
enable_default_logger(log_file=args.log_file)
```

**1.** The`enable_default_logger` call which enables the default logger, which:

* outputs to the Requestor application's `stderr` all the log messages with the level `INFO`
* if the `log_file` is specified, outputs to the file specified all the log messages with the level `INFO` and `DEBUG`

**2.** Then in the `Executor`we add:

```python
async with Executor(
    ...
    event_consumer=log_summary(log_event_repr),
    ...
) as executor:
```

Here, the `log_summary` creates `SummaryLogger` instance that is passed as event consumer to the Executor we are creating.
{% endtab %}

{% tab title="JS" %}
To enable logging to a file we need two things:

```javascript
utils.changeLogLevel("debug");
```

This command performs the following:

* Enables debugging into a file. The time-stamped `*.log` files are stored in `/logs` directory.
* All the log messages on the screen are of `info` and `debug` levels. 
* All the log messages in `*.log` file are of `info`, `debug`, `warn` and `silly` levels.
* `*.log` file additionally contains string content of all the `stderr` and `stdout` outputs from the Provider. This is what we are looking for.

Then in the `Executor`we need to:

```javascript
new Executor({
    ...
    event_consumer: logUtils.logSummary(),
}),
```
{% endtab %}
{% endtabs %}

## Let's have an error

{% hint style="info" %}
We are going to simulate some bug in the code now. We will take a fully working piece of code, modify it a bit \(create the bug in the code that produces and error on the provider's side\) and then, by analyzing the log file, we will track what the root cause of the problem is. In the end, we will fix the code and check the details of its work with the analysis of the log file.
{% endhint %}

{% tabs %}
{% tab title="Python" %}
Let's look at the code of yacat that is part of the Golem high-level Python API repository. All the code that enables logging is already there. We just need to use `--log-file` command-line switch.

{% embed url="https://github.com/golemfactory/yapapi/blob/master/examples/yacat/yacat.py" caption="" %}

In line 29 we are generating the body of `keyspace.sh` script to be executed on the provider:

```python
def write_keyspace_check_script(mask):
    with open("keyspace.sh", "w") as f:
        f.write(f"hashcat --keyspace -a 3 {mask} -m 400 > /golem/work/keyspace.txt")
```

Now, we'll simulate a typo in the code. Instead of:

`hashcat --keyspace -a 3 {mask} -m 400 > /golem/work/keyspace.txt`

let's have the following:

`hashcat --keeyspace -a 3 {mask} -m 400 > /golem/work/keyspace.txt`

So just an additional character - `e` - that has passed unnoticed. A typical developer day.

Save the file and run yacat with some example mask and hash:

```python
python3 yacat.py '?a?a?a' '$P$5ZDzPE45CLLhEx/72qt3NehVzwN2Ry/' --log-file log.txt
```

On Windows please use:

```text
python yacat.py ?a?a?a $P$5ZDzPE45CLLhEx/72qt3NehVzwN2Ry/ --log-file log.txt
```
{% endtab %}

{% tab title="JS" %}
Currently in our JS examples, we do not have one that will suit this tutorial, so we will work on some imagined JS port the yacat example. Let's assume we have the following code:

```javascript
  async function* worker(ctx, tasks) {
    for await (let task of tasks) {
      ctx.run("/bin/sh", ["-c", "hashcat --keyspace -a 3 '?a?a?a' -m 400"]);
      yield ctx.commit({timeout: dayjs.duration({ seconds: 120 }).asMilliseconds()});
      task.accept_result();
    }
    return;
  }
```

Now we are simulating a typo in the code. Instead of:

`hashcat --keyspace -a 3 {mask} -m 400`

let's have the following code:

`hashcat --keeyspace -a 3 {mask} -m 400`

So, just an additional `e` character that has passed unnoticed. A typical developer day.
{% endtab %}
{% endtabs %}

## Reading the log file

{% tabs %}
{% tab title="Python" %}
Let's try to track the effect of our typo in the contents of `log.txt` - the file that has been generated by yacat as a result of us using `--log-file log.txt`.

First, let's find the entry corresponding with the first `ctx.run` call. We can do it by searching the log.txt file \(from the top of the file, for example by hitting Ctrl-F in our favourite file viewer\) in search of

```python
command={'run'
```

This way, we can find a log entry that looks like this:

```python
[2021-01-22 15:20:16,273 DEBUG yapapi.events] 
CommandStarted(agr_id='b39318a4c682010ea9be37201b8f428fad81b28cfe949ca11cca9912a465d34c', 
task_id='1', cmd_idx=3, command={'run': {'entry_point': '/bin/sh', 'args': 
['/golem/work/keyspace.sh'], 'capture': {'stdout': {'stream': {}}, 'stderr': {'stream': {}}}}})
```

Now, below it, there are the log entries we were looking for:

```python
[2021-01-22 15:20:16,383 DEBUG yapapi.events] 
CommandStdErr(agr_id='b39318a4c682010ea9be37201b8f428fad81b28cfe949ca11cca9912a465d34c', 
task_id='1', cmd_idx=3, 
output="hashcat: unrecognized option '--keeyspace'\n\x1b[31mInvalid argument specified.\x1b[0m\n\n")
[2021-01-22 15:20:16,385 DEBUG yapapi.events] 
CommandExecuted(agr_id='b39318a4c682010ea9be37201b8f428fad81b28cfe949ca11cca9912a465d34c', 
task_id='1', cmd_idx=3, command={'run': {'entry_point': '/bin/sh', 'args': 
('/golem/work/keyspace.sh',), 'capture': {'stdout': {'stream': {}}, 'stderr': {'stream': {}}}}}, 
success=False, message='ExeScript command exited with code 255')
```

There are two pieces of those log entries that are the most important to us:

`hashcat: unrecognized option '--keeyspace'\n\x1b[31mInvalid argument specified.\x1b[0m\n\n`

This is a `stderr` message generated by running the `hashcat` binary. It seems that `hashcat` does not recognize the `--keeyspace` parameter. This is indeed the error we simulated.
{% endtab %}

{% tab title="JS" %}
Let's try to track the effect of our typo in the contents of `*.log` files which reside in the `/log` directory. You will be able to identify the correct `*.log` file by the timestamp in the filename.

So what would there be in the `*.log` file content? First, let's find the entry corresponding with the `ctx.run` call. We can do it by searching the log file from the top of the file, for example by hitting Ctrl-F in our favorite file viewer and looking for:

```python
command: {"run"
```

This way we can find a line that looks like this:

```python
2021-03-01 17:38:53 [yajsapi] warn: Command failed on provider 'reqc.4', 
command: {"run":{"entry_point":"/bin/sh","args":["-c",
"hashcat --keeyspace -a 3 '?a?a?a' -m 400"],"capture":{"stdout":{"atEnd":{"format":"str"}},
"stderr":{"atEnd":{"format":"str"}}}}}, 
msg: ExeScript command exited with code 255
```

Below it, there are the log entries we were looking for:

```python
2021-03-01 17:38:53 [yajsapi] debug: Command {"run":{"entry_point":"/bin/sh",
"args":["-c","hashcat --keeyspace -a 3 '?a?a?a' -m 400"],
"capture":{"stdout":{"atEnd":{"format":"str"}},"stderr":{"atEnd":{"format":"str"}}}}}: 
stdout: null, stderr: hashcat: unrecognized option '--keeyspace'
Invalid argument specified.
```

There are two pieces of those entries that are the most important to us:

```bash
hashcat: unrecognized option '--keeyspace'
Invalid argument specified.
```

This is the `stderr` message generated by running the `hashcat` binary. It seems that `hashcat` does not recognize the `--keeyspace` parameter. This is the error we simulated.
{% endtab %}
{% endtabs %}

{% hint style="success" %}
Congratulations! You have just tracked the bug by analyzing the requestor log file!
{% endhint %}

The other interesting piece of information here is:

`ExeScript command exited with code 255`

From this message, we know that the effect of this `ctx.run` resulted in the exit code having a value of `255` which has been returned by `hashcat` on exit. If there was any additional information in the command exit code, we would be able to see it here.

{% hint style="info" %}
The `255` value, we have in the log file here is generated by equivalent of POSIX `exit(-1)` executed on the `hashcat` termination. The `-1` was converted to `unsigned int` and came back as`255`.
{% endhint %}

## Fixing the error

{% tabs %}
{% tab title="Python" %}
Now, we can use that knowledge to fix the `write_keyspace_check_script` function by bringing it back to the correct form:

```python
def write_keyspace_check_script(mask):
    with open("keyspace.sh", "w") as f:
        f.write(f"hashcat --keyspace -a 3 {mask} -m 400 > /golem/work/keyspace.txt")
```

After saving the corrected `yacat.py` file and executing with the same command as above:

```python
python3 yacat.py '?a?a?a' '$P$5ZDzPE45CLLhEx/72qt3NehVzwN2Ry/' --log-file log.txt
```

```text
python yacat.py ?a?a?a $P$5ZDzPE45CLLhEx/72qt3NehVzwN2Ry/ --log-file log.txt
```

the problematic log fragment we analyzed above, becomes:

```python
[2021-01-22 16:22:17,217 DEBUG yapapi.events] 
CommandStarted(agr_id='bf8c484e362af417cbf5fc440281debd5c5cf772b433f53e0ecb90b4faac55f4', 
task_id='1', cmd_idx=3, command={'run': {'entry_point': '/bin/sh', 'args': 
['/golem/work/keyspace.sh'], 'capture': {'stdout': {'stream': {}}, 'stderr': {'stream': {}}}}})
[2021-01-22 16:22:18,303 DEBUG yapapi.events] 
CommandExecuted(agr_id='bf8c484e362af417cbf5fc440281debd5c5cf772b433f53e0ecb90b4faac55f4', 
task_id='1', cmd_idx=3, command={'run': {'entry_point': '/bin/sh', 'args': ('/golem/work/keyspace.sh',), 
'capture': {'stdout': {'stream': {}}, 'stderr': {'stream': {}}}}}, success=True, message=None)
```

As you can see, we have `success=True` there, which means that the `ctx.run` execution was successful.
{% endtab %}

{% tab title="JS" %}
To fix the issue, we would replace the erroneous code with:

```javascript
  async function* worker(ctx, tasks) {
    for await (let task of tasks) {
      ctx.run("/bin/sh", ["-c", "hashcat --keyspace -a 3 '?a?a?a' -m 400"]);
      yield ctx.commit({timeout: dayjs.duration({ seconds: 120 }).asMilliseconds()});
      task.accept_result();
    }
    return;
  }
```

After running the app, the problematic log fragment we analyzed above, would become the following:

```bash
2021-03-01 17:36:22 [yajsapi] debug: Command successful on provider '2rec-ubuntu.4', 
command: {"run":{"entry_point":"/bin/sh","args":["-c","hashcat --keyspace -a 3 '?a?a?a' -m 400"],
"capture":{"stdout":{"atEnd":{"format":"str"}},"stderr":{"atEnd":{"format":"str"}}}}}.
2021-03-01 17:36:22 [yajsapi] silly: Command {"run":{"entry_point":"/bin/sh","args":["-c",
"hashcat --keyspace -a 3 '?a?a?a' -m 400"],"capture":{"stdout":{"atEnd":{"format":"str"}},
"stderr":{"atEnd":{"format":"str"}}}}}: stdout: 9025, stderr: null, msg: null.
```

Here, we see `stdout: 9025`, which is the output from the hashcat command call.
{% endtab %}
{% endtabs %}

{% hint style="success" %}
Congratulations! The bug has been corrected!
{% endhint %}

## Log files - what else is there?

In the log file, there is all the information you are already familiar with, because it is displayed as `INFO` messages in the `stdout` of the requestor application examples. Between the familiar `INFO` entries, you will find additional `DEBUG` messages. Let's have a look at the most interesting ones.

### Requestor &gt; Provider file transfer details

{% tabs %}
{% tab title="Python" %}
The transfers of files from the Requestor to the Provider can be identified in the log as pairs of `CommandStarted` and `CommandExecuted` lines similar to:

```python
[2021-01-22 16:22:16,773 DEBUG yapapi.events] 
CommandStarted(agr_id='bf8c484e362af417cbf5fc440281debd5c5cf772b433f53e0ecb90b4faac55f4', 
task_id='1', cmd_idx=2, command={'transfer': {'from': 
'gftp://0xf378a153b24fe98227ee153a0cb056faed155ba7/050a02b6976e59fbc1b9dfa6292d4f63942a820ee7dcc859af0a5efaea80c959', 
'to': 'container:/golem/work/keyspace.sh', 'format': None, 'depth': None, 'fileset': None}})
[2021-01-22 16:22:17,217 DEBUG yapapi.events] 
CommandExecuted(agr_id='bf8c484e362af417cbf5fc440281debd5c5cf772b433f53e0ecb90b4faac55f4', 
task_id='1', cmd_idx=2, command={'transfer': {'from': 
'gftp://0xf378a153b24fe98227ee153a0cb056faed155ba7/050a02b6976e59fbc1b9dfa6292d4f63942a820ee7dcc859af0a5efaea80c959', 
'to': 'container:/golem/work/keyspace.sh'}}, success=True, message=None)
```

The source file name is identified by `'from':'gftp://0xf378a153b24fe98227ee153a0cb056faed155ba7/050a02b6976e59fbc1b9dfa6292d4f63942a820ee7dcc859af0a5efaea80c959'`

{% hint style="info" %}
The `gftp` is internal Golem protocol for transferring files in the Golem network.
{% endhint %}

The target file name is identified by

`'to': 'container:/golem/work/keyspace.sh'`

`success=True` informs us that the operation was successful.
{% endtab %}

{% tab title="JS" %}
The transfers of files from the Requestor to the Provider can be identified in the log as:

```bash
2021-03-01 20:12:55 [yajsapi] debug: Command successful on provider 'optiplexican_gn-02', command: 
{"transfer":{"from":"gftp://0x33c908b3987dc7c74f62600fadb7eb103e2ac9f4/3185b7bb49f5ac7d089e05d2e1a185b189801988f7087e200081d25ed5c98a7e",
"to":"container:/golem/work/testfile.txt","args":{}}}.
2021-03-01 20:12:55 [yajsapi] silly: Command {"transfer":{"from":"gftp://0x33c908b3987dc7c74f62600fadb7eb103e2ac9f4/3185b7bb49f5ac7d089e05d2e1a185b189801988f7087e200081d25ed5c98a7e",
"to":"container:/golem/work/testfile.txt","args":{}}}: stdout: null, stderr: null, msg: null.
```

The source file name is identified by `"from":"gftp://0x33c908b3987dc7c74f62600fadb7eb103e2ac9f4/3185b7bb49f5ac7d089e05d2e1a185b189801988f7087e200081d25ed5c98a7e"`

{% hint style="info" %}
The `gftp` is internal Golem protocol for transferring files in the Golem network.
{% endhint %}

The target file name is identified by

`"container:/golem/work/testfile.txt"`
{% endtab %}
{% endtabs %}

### stdout of executing commands on the Provider

{% tabs %}
{% tab title="Python" %}
Any execution of the `ctx.run` function - which is what we use to run commands inside provider's execution units - will result in the following entries in the log file: a `CommandStarted`, followed by any number of `CommandStdOut` depending on the output produced by the command and finally, one `CommandExecuted`. For example, this is the result of yacat's main `ctx.run` :

```python
[2021-01-22 16:22:36,986 DEBUG yapapi.events]
CommandStarted(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4', 
task_id='2', cmd_idx=3, command={'run': {'entry_point': '/bin/sh', 'args': 
['-c', 'rm -f /golem/work/*.potfile ~/.hashcat/hashcat.potfile; touch /golem/work/hashcat_0.potfile; 
hashcat -a 3 -m 400 /golem/work/in.hash ?a?a?a --skip=0 --limit=3009 --self-test-disable -o 
/golem/work/hashcat_0.potfile || true'], 'capture': {'stdout': {'stream': {}}, 'stderr': {'stream': {}}}}})
[2021-01-22 16:22:37,018 DEBUG yapapi.events] 
CommandStdOut(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4', 
task_id='2', cmd_idx=3, output='hashcat (v5.1.0) starting...\n\n')
[2021-01-22 16:22:37,018 DEBUG yapapi.events] 
CommandStdOut(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4',
task_id='2', cmd_idx=3, output='\x1b[2K\rParsed Hashes: 1/1 (100.00%)\x1b[2K\rParsed Hashes: 1/1 (100.00%)')
[2021-01-22 16:22:37,018 DEBUG yapapi.events] 
CommandStdOut(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4', 
task_id='2', cmd_idx=3, output='\x1b[2K\rCounted lines in /golem/work/in.hash...')
[2021-01-22 16:22:37,019 DEBUG yapapi.events] 
CommandStdOut(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4', 
task_id='2', cmd_idx=3, output='Counting lines in /golem/work/in.hash...')
[2021-01-22 16:22:37,019 DEBUG yapapi.events] 
CommandStdOut(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4', 
task_id='2', cmd_idx=3, output='OpenCL Platform #1: Intel(R) Corporation\n========================================\n* 
Device #1: Intel(R) Core(TM) i7-4790K CPU @ 4.00GHz, 5347/21388 MB allocatable, 4MCU\n\n')
[2021-01-22 16:22:37,048 DEBUG yapapi.events] 
CommandStdOut(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4', 
task_id='2', cmd_idx=3, output='\x1b[2K\rSorting hashes...\x1b[2K\rSorted hashes...\x1b[2K\r
Removing duplicate hashes...\x1b[2K\rRemoved duplicate hashes...\x1b[2K\rSorting salts...\x1b[2K\rSorted salts...\x1b[2K\r
Comparing hashes with potfile entries...\x1b[2K\rCompared hashes with potfile entries...')
[2021-01-22 16:22:37,048 DEBUG yapapi.events] 
CommandStdOut(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4', 
task_id='2', cmd_idx=3, output='\x1b[2K\rGenerated bitmap tables...')
[2021-01-22 16:22:37,048 DEBUG yapapi.events] 
CommandStdOut(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4', 
task_id='2', cmd_idx=3, output='\x1b[2K\rGenerating bitmap tables...')
[2021-01-22 16:22:37,963 DEBUG yapapi.events] 
CommandStdOut(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4', 
task_id='2', cmd_idx=3, output='\x1b[2K\rHashes: 1 digests; 1 unique digests, 1 unique salts\n
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates\n\nApplicable optimizers:\n* 
Zero-Byte\n* Single-Hash\n* Single-Salt\n* Brute-Force\n\nMinimum password length supported by kernel: 0\n
Maximum password length supported by kernel: 256\n\n\x1b[33mATTENTION! Pure (unoptimized) 
OpenCL kernels selected.\x1b[0m\n\x1b[33mThis enables cracking passwords and salts > length 32 but for the 
price of drastically reduced performance.\x1b[0m\n\x1b[33mIf you want to switch to optimized OpenCL kernels, 
append -O to your commandline.\x1b[0m\n\x1b[33m\x1b[0m\nWatchdog: Hardware monitoring interface not found 
on your system.\nWatchdog: Temperature abort trigger disabled.\n\n
Initializing device kernels and memory...\x1b[2K\rInitializing OpenCL runtime for device #1...')

...

[2021-01-22 16:22:40,649 DEBUG yapapi.events] 
CommandExecuted(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4', 
task_id='2', cmd_idx=3, command={'run': {'entry_point': '/bin/sh', 'args': 
('-c', 'rm -f /golem/work/*.potfile ~/.hashcat/hashcat.potfile; touch /golem/work/hashcat_0.potfile; 
hashcat -a 3 -m 400 /golem/work/in.hash ?a?a?a --skip=0 --limit=3009 --self-test-disable -o 
/golem/work/hashcat_0.potfile || true'), 'capture': {'stdout': {'stream': {}}, 'stderr': {'stream': {}}}}}, success=True, message=None)
```

The `output` parts of the `CommandStdOut` contain the `stdout` of the `ctx.run`. For example the beginning of the `hashcat` execution can be traced by `output='hashcat (v5.1.0) starting...\n\n'` .
{% endtab %}

{% tab title="JS" %}
The output from a successful `ctx.run` call will result in a log file that contains something like:

```bash
2021-03-01 17:43:11 [yajsapi] debug: 
Command successful on provider 'optiplexican_gn-02', command: {"run":{"entry_point":"/bin/sh","args":["-c",
"hashcat -a 3 -m 400 '$P$5ZDzPE45CLLhEx/72qt3NehVzwN2Ry/' '?a?a?a'"],"capture":{"stdout":{"atEnd":{"format":"str"}},
"stderr":{"atEnd":{"format":"str"}}}}}.
2021-03-01 17:43:11 [yajsapi] silly: Command {"run":{"entry_point":"/bin/sh","args":["-c",
"hashcat -a 3 -m 400 '$P$5ZDzPE45CLLhEx/72qt3NehVzwN2Ry/' '?a?a?a'"],"capture":{"stdout":{"atEnd":{"format":"str"}},
"stderr":{"atEnd":{"format":"str"}}}}}: stdout: hashcat (v5.1.0) starting...

OpenCL Platform #1: Intel(R) Corporation
========================================
* Device #1: AMD A8-6500B APU with Radeon(tm) HD Graphics, 1488/5954 MB allocatable, 4MCU
```

The beginning of the `hashcat` execution can be traced by `stdout: hashcat (v5.1.0) starting...`.
{% endtab %}
{% endtabs %}

### Provider &gt; Requestor file transfer details

{% tabs %}
{% tab title="Python" %}
Getting files back from the provider to the requestor will leave a trace such as:

```python
[2021-01-22 16:22:18,304 DEBUG yapapi.events] 
CommandStarted(agr_id='bf8c484e362af417cbf5fc440281debd5c5cf772b433f53e0ecb90b4faac55f4', 
task_id='1', cmd_idx=4, command={'transfer': {'from': 'container:/golem/work/keyspace.txt', 'to': 
'gftp://0xf378a153b24fe98227ee153a0cb056faed155ba7/0GdLl2vUcEYgO7ExgkVuttc5oqeAmU90Im911Eqs5LZXzD9ehlYCUGH7p5oII9jSg', 
'format': None, 'depth': None, 'fileset': None}})
[2021-01-22 16:22:18,513 DEBUG yapapi.events] 
CommandExecuted(agr_id='bf8c484e362af417cbf5fc440281debd5c5cf772b433f53e0ecb90b4faac55f4', 
task_id='1', cmd_idx=4, command={'transfer': {'from': 'container:/golem/work/keyspace.txt', 'to': 
'gftp://0xf378a153b24fe98227ee153a0cb056faed155ba7/0GdLl2vUcEYgO7ExgkVuttc5oqeAmU90Im911Eqs5LZXzD9ehlYCUGH7p5oII9jSg'}}, 
success=True, message=None)
[2021-01-22 16:22:18,517 DEBUG yapapi.events] 
GettingResults(agr_id='bf8c484e362af417cbf5fc440281debd5c5cf772b433f53e0ecb90b4faac55f4', task_id='1')
[2021-01-22 16:22:18,519 DEBUG yapapi.events] DownloadStarted(path='/golem/work/keyspace.txt')
[2021-01-22 16:22:18,521 DEBUG yapapi.events] DownloadFinished(path='keyspace.txt')
```

The source filename is identified by `'from': 'container:/golem/work/keyspace.txt'` and the target filename is identified by `DownloadFinished(path='keyspace.txt')`.
{% endtab %}

{% tab title="JS" %}
Getting files back from the provider to the requestor will leave a trace such as:

```bash
2021-03-01 20:18:49 [yajsapi] debug: Command successful on provider 'phillip', command: {"transfer":{
"from":"container:/golem/work/testfile.txt",
"to":"gftp://0x33c908b3987dc7c74f62600fadb7eb103e2ac9f4/yuGs2xwoVsUbCrZCFvFxdO734KCWYZMQ313hTHmM8QSiliRkgRp8gd8v8jAnrtA7f"}}.
2021-03-01 20:18:49 [yajsapi] silly: Command {"transfer":
{"from":"container:/golem/work/testfile.txt",
"to":"gftp://0x33c908b3987dc7c74f62600fadb7eb103e2ac9f4/yuGs2xwoVsUbCrZCFvFxdO734KCWYZMQ313hTHmM8QSiliRkgRp8gd8v8jAnrtA7f"}}: 
stdout: null, stderr: null, msg: null.
```

The source filename is identified by `"from":"container:/golem/work/testfile.txt"`.
{% endtab %}
{% endtabs %}

## Docker exec - an alternative approach

Analyzing the requestor logs gives us the full picture of what is going on under the hood of Golem, but in some cases, it might be easier to start with executing a simple `docker exec`that runs commands on the container. The syntax of this docker command is:

```bash
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

This command can be used to execute the `ctx.run` scripts directly on the provider container. This is regular Docker usage, so we'll recommend you consulting the Docker's reference: [https://docs.docker.com/engine/reference/commandline/exec/](https://docs.docker.com/engine/reference/commandline/exec/)

## Closing words

Besides requestor logs, there are also provider-side logs. The suggested approach for the app developer to analyze the provider-side logs is to utilize our interactive testing environment:

{% page-ref page="interactive-testing-environment/" %}

