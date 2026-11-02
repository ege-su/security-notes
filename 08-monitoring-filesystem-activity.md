<p align="right"><sub>October 29, 2026</sub></p>

# Monitoring Filesystem Activity on macOS: An LM Studio Case Study

## 1. Objective

Applications often read, write, and temporarily store data in locations that may not be immediately obvious to users. Even when a specific storage directory is configured, an application may use additional locations for temporary files, caches, or supporting components.

After downloading several large language models (LLMs) through **LM Studio** to an external SSD, I noticed an unexpected amount of storage being consumed on my Mac's internal drive. Upon investigating, I discovered hundreds of gigabytes of files within macOS's `useractivityd/shared-pasteboard/archives` directory, which I found out that is associated with Apple's Universal Clipboard functionality.

Because the issue appeared around the time I had been downloading and handling large model files, I wondered whether LM Studio's filesystem operations could be contributing to the unexpected storage usage. Although I had no evidence connecting the application to these archives, the discovery prompted me to investigate where LM Studio actually reads, writes, and temporarily stores its data.

My main questions were:

- Does LM Studio load models directly from its configured external storage?
- Where does it write newly downloaded models?
- Does it temporarily store downloaded files on the internal SSD?
- Which processes are responsible for these operations?

I used my machine's native macOS command-line utilities to observe application behavior during model loading and downloading.

## 2. Environment and Tools

| Component        | Description                                           |
| ---------------- | ----------------------------------------------------- |
| Operating system | macOS (Apple Silicon)                                 |
| Application      | LM Studio 0.4.26                                      |
| Model storage    | Samsung T7 Shield external SSD, APFS                  |
| Model directory  | `/Volumes/Shield/Models` (the said external SSD)      |
| Monitoring tools | `ps`, `fs_usage`, `lsof`, `find`, `du`, `grep`, `awk` |

_Note: The external volume name and user paths are anonymized._ `Shield` _represents my configured external SSD._

## 3. Experiment 1: Monitoring Model Loading

### 3.1 Identifying application processes

Before loading a model, I verified that LM Studio had no models running:

	lms ps

The output was:

```
No models are currently loaded.
```

Nice.

I then identified the application's running processes with the following command:

	ps -axo pid,comm,args |
	  grep -Ei 'LM Studio|llmster|llama|mlx' |
	  grep -v grep

The output included the main LM Studio process, Electron helper processes, and a Node.js worker. Nothing unusual.

This established a baseline for the experiment.

### 3.2 Capturing filesystem activity

I started monitoring filesystem operations using 
	`sudo fs_usage -w -f filesys`
and loaded **TheDrummer Cydonia 24B v4.3**, which I had as a Q4_K_M GGUF file, on LM Studio.

Since I hadn't added enough filters in my command, the initial capture generated **266.1 MB of logging data**, which was mostly made of unrelated activity. To isolate relevant events, I filtered the results for the external volume, model files, and associated processes afterwards.

### 3.3 Observations

The trace showed LM Studio opening the GGUF file from the external SSD:

```
open
/Volumes/Shield/Models/.../TheDrummer_Cydonia-24B-v4.3-Q4_K_M.gguf
LM Studio
```

Shortly afterward, I saw a separate `llama-server` process accessing the same file:

```
open ... /Volumes/Shield/Models/...gguf
llama-server

RdData[A] ... /Volumes/Shield/Models/...gguf
llama-server
```

Then I used the following command to find out which processes held model files open:

```
sudo lsof -nP +c 0 |
  grep -Ei '/Volumes/Shield/Models/|\.gguf'
```

The result showed that `llama-server` held a reference to the GGUF file on the external SSD. The file size reported by `lsof` was **14,333,910,432 bytes**, approximately 13.35 GiB.

I also counted filesystem operations associated with the model filename:

|Operation|Recorded events|
|---|---|
|`open`|9|
|`access`|1|
|`RdData[A]`|27|
|`PgIn[A]`|39,433|

**Observation:** During model loading, LM Studio's `llama-server` backend accessed the GGUF file directly on the external SSD. I found no evidence of model duplication.

## 4. Experiment 2: Monitoring Model Downloads

In the next experiment, I wanted to find out whether LM Studio downloaded model files directly to the external storage I specified or staged them internally first.

### 4.1 Test models

I initiated downloads of two small models through LM Studio, choosing different formats so that I could compare their download behavior:

|Model|Format|
|---|---|
|Qwen3-4B-Instruct-2507|MLX, 4-bit|
|Qwen3-VL-4B-Instruct|GGUF, Q4_K_M|

### 4.2 Monitoring open files

While both downloads were in progress, I inspected open files using:

```
sudo lsof -nP +c 0 |
  grep -E 'downloading_.*\.(gguf|safetensors)\.part'
```

The output showed me that there were three partial download files on Shield, all held open for writing by the app's main process:

```
downloading_model.safetensors.part
downloading_Qwen3-VL-4B-Instruct-Q4_K_M.gguf.part
downloading_mmproj-Qwen3-VL-4B-Instruct-F16.gguf.part
```

I then checked their sizes at two different points during the downloads. The file sizes had increased. Together with the open write descriptors, I could say that the model data was downloading directly on the external SSD.

|Partial file|First observation|Second observation|
|---|---|---|
|MLX weights|576.7 MB|615.3 MB|
|GGUF weights|223.0 MB|232.5 MB|
|Multimodal projector|138.4 MB|168.2 MB|

### 4.3 Capturing a more focused trace

The earlier 266 MB log file bothered me a lot, so I decided to do another filesystem capture with only relevant processes and operations targeted. In the end, I had **12,934 lines** and **3,259,855 bytes** of data. The log file was approximately 3.1 MiB.

Filtering events involving temporary download files with the  `downloading_` filter: 

	grep -F 'downloading_' "$LOG" |
	  awk '{count[$2]++}
	       END {
	         for (op in count)
	           print op, count[op]
	       }' |
	  sort

showed me:

```
WrData[A] 126
open 14
rename 11
```

The trace I'd captured showed LM Studio opening temporary `.part` files, writing data to them, and performing rename operations.

**Observation:** Both MLX and GGUF downloads created partial files that grew as the downloading process went on in my configured external directory, contradicting my hypothesis that LM Studio necessarily downloads an entire model to internal storage before copying it externally.

## 5. Experiment 3: Investigating Internal Temporary Storage

Although model files were written to Shield, the filesystem trace actually revealed activity under LM Studio's application directory on the internal storage.

One example was:
	~/.lmstudio/.internal/temp-downloads/
	downloading_plugin.qwen.qwen3-4b-2507.3.tar.gz.part
The log showed `open`, `WrData[A]`, and `rename` operations all on the Mac’s built-in drive.

So, it turned out that configuring the model directory on an external drive does not prevent LM Studio from using internal storage for other downloaded components.

### 5.1 Checking temporary files

I inspected the directory using:
	find "$HOME/.lmstudio/.internal/temp-downloads" \
	  -type f -exec ls -lh {} \; 2>/dev/null

Before the downloads had completed, the directory contained `plugin.qwen.qwen3-vl-4b.4.tar.gz` which was approximately **167 KB**, and disk usage, which I checked with `du -sh "$HOME/.lmstudio/.internal/temp-downloads"` was **168K**.

After the downloads completed, I repeated both commands. This time, the directory contained no files and disk usage had decreased to **0B**.

**Observation:** LM Studio did use internal temporary storage for Qwen-related plugin downloads, but after it completed its downloads, the directory it used was empty. I could not say whether the temporary archive was deleted or moved elsewhere during installation.

## 6. Security Relevance

Although this was not a malware investigation or a real security incident, I believe it demonstrated several techniques that matter for endpoint security and application auditing.

### Application behavior monitoring

Just because you tell an app where to store its files does not mean that will be the only file system location it accesses. Watching the actual file operations helps you spot divergence from the app's expected behavior.

### Process attribution

I could tie filesystem activity to specific processes using `lsof` and `fs_usage` . In this case, `llama-server` handled loading the model while the partial downloads were held open by LM Studio's main process.

### Temporary file visibility

The experiment showed that temporary directories can contain application components even when primary data is configured to reside elsewhere. This matters for privacy, storage management, forensic investigations, and organizational data-handling requirements depending on the application and its data.

### Limitations of monitoring

Filesystem monitoring requires **careful filtering** and interpretation, as proved by my first log of ~266 MB. A large log does not necessarily provide better evidence, but adding too many filters may cost you relevant events too. Also filesystem events alone cannot prove whether data was transmitted over a network or whether an application's behavior was malicious.

## 7. Limitations

This experiment has several limitations:
- All monitoring was done on a single macOS system with one specific version of LM Studio.
- The captures covered only selected processes and short time windows.
- I did not monitor every location where temporary files could be stored.
- Not finding a copy of the model on the built-in drive does not prove that such copies can never appear.

These results therefore describe what I observed during these specific loading and downloading operations. They are not a guarantee about every LM Studio workflow.

## 8. Conclusion

I used native macOS tools to examine how LM Studio reads and stores model data. Three findings stood out:

1. **Model loading:** The `llama-server` backend read an existing GGUF file directly from external storage.
2. **Model downloads:** Both GGUF and MLX downloads created growing partial files on the configured external SSD.
3. **Temporary storage:** LM Studio used its internal application directory for Qwen-related plugin downloads. The temporary directory I monitored was empty once the downloads finished.

I was not able to link LM Studio to the unusually large shared-pasteboard archives that first prompted this investigation. Even so, the experiment showed how native macOS filesystem tools can be used to examine an application’s behavior, tie operations to specific processes, and separate what I observed from what I had assumed at the start.

One last practical lesson was the value of **targeted logging**. My first capture produced roughly 266 MB of data, while the more selective one produced later was 3.1 MB large and far easier to analyze.

The same approach can be applied to other desktop applications, unexplained disk usage, or suspicious-looking filesystem activity.
