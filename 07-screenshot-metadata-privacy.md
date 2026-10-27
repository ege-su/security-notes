<p align="right"><sub>October 27, 2026</sub></p>

# Does Screenshotting Remove Photo Metadata? A Privacy Experiment on iOS and macOS

Digital photographs can contain information beyond what is visible. Embedded metadata can include the camera model, software version, capture timestamp, and geographic coordinates. Taking a screenshot before sharing an image is often considered a quick way to remove this information. But does it really work to remove identifying metadata, or can it introduce new information about the device or environment used to create the screenshot?

To investigate, I compared an original photograph with several versions created using iOS, macOS, and ExifTool.

## 1. Environment and Tools

- **Operating systems:** iOS and macOS
- **Original camera:** Apple iPhone 13
- **Metadata analysis:** ExifTool
- **Interface:** macOS Terminal
- **Additional hardware:** GIGABYTE M27UP external monitor

## 2. Methodology

I prepared five versions of the same photograph for the primary comparison:

| File                              | Description                                                                       |
| --------------------------------- | --------------------------------------------------------------------------------- |
| `01-original.JPG`                 | Baseline photograph downloaded from iCloud Photos                                 |
| `02-location-disabled-export.JPG` | Photograph shared from iOS to macOS via AirDrop with the Location option disabled |
| `03-mac-screenshot.png`           | Screenshot taken on macOS using an external monitor                               |
| `04-iphone-screenshot.HEIC`       | Uncropped screenshot of the photograph, taken on iOS                              |
| `05-exiftool-stripped.JPG`        | Copy of the baseline with writable metadata removed using ExifTool                |

Later, I added another to investigate display-related metadata:

- `07-mac-built-in-screenshot.png` — Screenshot captured using the MacBook's built-in display.

### Metadata extraction

I examined each file using ExifTool:

	exiftool -a -G1 -s 01-original.JPG

The relevant options were:

- `-a`: Display duplicate metadata tags.
- `-G1`: Show metadata group names.
- `-s`: Display technical tag names.

To create the metadata-stripped version of image `01`:

	cp 01-original.JPG 05-exiftool-stripped.JPG
	exiftool -all= -overwrite_original 05-exiftool-stripped.JPG


To inspect the GPS metadata separately:

	exiftool -a -G1 -s -GPS:All 01-original.JPG

I repeated this inspection for every sample mentioned above. (GPS values are obviously excluded from the published results.)

## 3. Results

### 3.1 Original photograph

The baseline JPEG, `01-original.JPG`, contained the following metadata:

|Field|Value|
|---|---|
|Camera manufacturer|Apple|
|Camera model|iPhone 13|
|Software|18.6.2|
|Original capture time|2025-10-21 15:36:13|
|Resolution|4032 × 3024|
|File size|Approximately 6.3 MB|

The file also contained 15 GPS-related tags, including latitude, longitude, altitude, speed, image direction, and positioning error. Not the kind of information you'd want to hand out to third parties.

**Observation:** The original photograph contained information about both the camera and its geographic context.

### 3.2 Location-disabled export

`02-location-disabled-export.JPG` is the same photograph shared from my iPhone to my MacBook via AirDrop, with the Location option disabled while sharing. The ExifTool inspection revealed that it contained no GPS tags. However, the original camera model, software version, and capture timestamp were still present.

There were also additional XMP timestamp fields. The file was approximately 7.2 MB, compared with the original's 6.3 MB.

**Observation:** Disabling location sharing removed the GPS metadata from the shared file, but other potentially identifying information was carried over.

### 3.3 macOS screenshot

The macOS screenshot, `03-mac-screenshot.png`, did not have the original camera model, capture timestamp, or any GPS tags in its metadata.

But upon further inspection I saw that it contained new fields I had not expected:

```
[ExifIFD]     UserComment        : Screenshot
[XMP-exif]    UserComment        : Screenshot
[XMP-x]       XMPToolkit         : XMP Core 6.0.0
[ICC_Profile] ProfileDescription : M27UP
[ICC-header]  ProfileCreator     : Apple Computer Inc.
```

 `M27UP` was the model name of the GIGABYTE external monitor on which the screenshot was taken.  The embedded ICC color profile in the screenshot announced the display environment.

### 3.4 iOS screenshot

`04-iphone-screenshot.HEIC` did not have any GPS tags or explicit camera make/model fields either, and its embedded timestamp had updated to the moment I took the screenshot:

```
[ExifIFD] DateTimeOriginal : 2026:10:27 14:37:31
```

Additional metadata included:

```
[ExifIFD]     UserComment        : Screenshot
[ICC_Profile] ProfileDescription : Display P3
[ICC-header]  ProfileCreator     : Apple Computer Inc.
```

Fortunately, the `Display P3` profile does not explicitly identify a specific iPhone model.

### 3.5 ExifTool-stripped JPEG

When I inspected `05-exiftool-stripped.JPG`, the stripped photograph, I saw that while it retained its original dimensions of 4032 × 3024 pixels, which did not come as a shock, its camera model, software version, original timestamp, GPS fields, and ICC color profile were absent, which was the intended result of removing its metadata.

Its file size went down from 6.3 MB (`01-original.JPG`) to 5.9 MB.

### 3.6 Additional experiment: Does the display affect screenshot metadata?

After discovering that the macOS screenshot reported its `M27UP` color profile description, I decided to capture another screenshot using the MacBook's built-in display this time.

I examined both screenshots using:

```
exiftool -G1 -s \
  -ProfileDescription \
  03-mac-screenshot.png \
  07-mac-built-in-screenshot.png
```

The results were:

```
======== 03-mac-screenshot.png
[ICC_Profile] ProfileDescription : M27UP

======== 07-mac-built-in-screenshot.png
[ICC_Profile] ProfileDescription : Display
```

The built-in-display screenshot contained the generic description `Display`.

**Observation:** The embedded color-profile description differed depending on which display was used to capture the screenshot. Since ICC profiles can be assigned or modified independently of physical hardware, though, their descriptions are not always reliable device identifiers.

## 4. Comparison

| Image variant                     | GPS tags | Original camera model | Other notable metadata              |
| --------------------------------- | -------- | --------------------- | ----------------------------------- |
| Original photograph               | Present  | iPhone 13             | Capture date, software, GPS context |
| Location-disabled export          | Absent   | iPhone 13             | Original timestamp, software        |
| Mac screenshot (external display) | Absent   | —                     | `Screenshot`, `M27UP`               |
| iPhone screenshot                 | Absent   | —                     | New capture timestamp, `Display P3` |
| ExifTool-stripped JPEG            | Absent   | —                     | Basic image properties              |

*The additional built-in-display screenshot was used only to compare ICC profile descriptions between Mac screenshots.*

## 5. Discussion

The experiment showed that while taking a screenshot can remove an original photograph's GPS and camera identification metadata, it does not ensure that there will be no identifying information. For example, in the macOS screenshot, the embedded ICC profile contained a description matching my external monitor's model name.

Exporting a photo with Apple's location setting disabled offers a more targeted approach. In my experiment, it removed the GPS tags while leaving the original camera model, software version, and capture timestamp untouched. 

It’s also worth separating the idea of metadata privacy (hidden data attached to the file) and visual privacy (what you can actually see in the image). A screenshot, for example, might show open app windows, file names, notifications, or other information directly through its pixels.

From a privacy and digital forensics perspective, these differences matter because image metadata may reveal information about a file's creation or processing history, even when its original location and camera details have been removed.

## 6. Conclusion

Screenshotting changes what information an image reveals, but it does not guarantee anonymity. Screenshots can remove the original photo’s GPS and camera data while introducing new metadata of their own. For privacy-conscious sharing, check the metadata against what you want to protect, and also consider what the image itself shows.
