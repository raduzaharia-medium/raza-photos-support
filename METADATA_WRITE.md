# Photo Metadata: What Raza Photos Writes Back to Your Files

When you organize photos in Raza Photos — tagging people, assigning albums, correcting a date — that information can be saved back into the image files themselves. This keeps your library portable: the metadata travels with the photos no matter where they move or what app opens them next.

🔒 **Your originals are safe.** Raza Photos never touches your image data. A photo file has two distinct areas: the compressed image (your actual pixels) and the metadata (everything else — dates, names, locations). During the metadata export, Raza Photos operates exclusively in the metadata area. The compressed image is never decoded, never re-encoded, never even read. It is as if that part of the file does not exist from Raza Photos's perspective. Only the metadata portion is changed, and only the fields Raza Photos knows about. Every other field is left exactly as it was.

---

## When does a write happen?

**Never automatically.** Raza Photos queues up your changes internally as you work, but nothing is written to disk until you explicitly trigger a save by selecting **Save Metadata** from the **Library** menu.

At that point, Raza Photos works through the queue — only the photos that have pending changes — and writes each one. You can see progress and cancel at any time. Photos that have not changed are never touched.

_If you never run Save Metadata, your original files remain completely unmodified._

---

## Supported file formats

Metadata can be written back to **JPEG, PNG, HEIC/HEIF, PSD, and TIFF** files.

**AVIF** is not supported for write-back at this time — Raza Photos can read metadata from AVIF files, but changes to those photos are held in the library database and not written to disk.

---

## What is written and where

For each field, the table shows which metadata standard(s) receive the value. Multiple standards are written simultaneously so the information is readable by as wide a range of apps as possible.

---

### 📅 Date taken

| Standard | Field written |
|---|---|
| **EXIF** | `DateTimeOriginal` |
| **TIFF** | `DateTime` |

Dates are written in the EXIF format `yyyy:MM:dd HH:mm:ss`. Both fields are updated together so that the date appears correctly in every app and operating system that reads it.

_**Technical note:** Writing the date requires a separate internal pass from all other metadata, because Apple's ImageIO framework does not allow date fields and general XMP metadata to be written in the same operation. Raza Photos handles this automatically with two sequential writes to the same file — both lossless._

---

### 🏷️ Title

| Standard | Field written |
|---|---|
| **IPTC** | `ObjectName` |
| **XMP** | `dc:title` |

Written to both IPTC and XMP so the title is visible in Adobe Lightroom, Capture One, Finder's Get Info panel, and any other tool that reads either standard.

---

### 📍 GPS coordinates

| Field | Standard | Technical detail |
|---|---|---|
| Latitude | **EXIF GPS** | `GPSLatitude` + `GPSLatitudeRef` (N/S) |
| Longitude | **EXIF GPS** | `GPSLongitude` + `GPSLongitudeRef` (E/W) |

GPS is written to the standard EXIF GPS dictionary — the same location cameras and smartphones use.

---

### 🗺️ Country and city

| Standard | Fields written |
|---|---|
| **IPTC** | `CountryPrimaryLocationName`, `City` |
| **XMP** (IPTC 4 XMP Extension) | `Iptc4xmpExt:CountryName`, `Iptc4xmpExt:City` inside `LocationCreated` |
| **ACDSee categories** | `Places` hierarchy inside `acdsee:categories` XML |

Written across all three standards so that apps using any of them pick up the location correctly.

---

### 👥 People

| Standard | Field written |
|---|---|
| **XMP** (IPTC 4 XMP Extension) | `Iptc4xmpExt:PersonInImage` array |

Person names are written as a standard array, readable by Lightroom, Capture One, digiKam, and any other app that supports the IPTC 4 XMP Extension specification.

---

### 🧩 Face regions (bounding boxes)

When a photo has people tagged with known face positions, the rectangles are written in two formats for maximum compatibility:

| Standard | Field written | Used by |
|---|---|---|
| **MWG Regions** (XMP) | `mwg-rs:Regions` | Lightroom, Picasa, digiKam, most modern apps |
| **ACDSee Regions** (XMP) | `acdsee-rs:Regions` | ACDSee Photo Studio |

Coordinates are stored normalized (relative to image dimensions) so they remain correct if the image is ever resized.

---

### 🗂️ Albums and events

Album and event membership is written in three places, covering different ecosystems:

| Standard | Field written | Format |
|---|---|---|
| **IPTC** | `Keywords` | Flat list of all album and event names |
| **XMP** | `dc:subject` | Same flat list in Dublin Core format |
| **XMP** (Lightroom) | `lr:hierarchicalSubject` | Hierarchical, e.g. `Albums\Vienna 2005`, `Events\Summer Holiday` |
| **ACDSee categories** | `acdsee:categories` XML | Full hierarchy: `Albums`, `Events`, `Places` subtrees |

This means your album structure is fully readable in Lightroom's Keywords panel, in ACDSee, and in any tool that supports standard subject keywords.

---

### 📸 Burst photography

| Field | Standard | Field written |
|---|---|---|
| Burst group ID | **XMP** (custom) | `razaphotos:BurstID` |
| Burst representative flag | **XMP** (custom) | `razaphotos:BurstRepresentative` |

These are stored in a custom XMP namespace registered to Raza Photos (`http://ns.razaphotos.app/1.0/`). Other apps will not interpret these fields, but they will not delete or corrupt them either — unknown XMP fields are preserved by all conforming XMP implementations.

The reason these are written at all is so your burst groupings survive if you ever move your photos to a new drive or re-import your library from scratch — Raza Photos reads them back on the next import and restores the groupings without any manual work.

---

## What is never written

Raza Photos only writes the fields listed above. The following are never touched:

| Category | Examples |
|---|---|
| **Pixel data** | Never decoded, never read, never written — Raza Photos works in a completely separate area of the file |
| **Camera settings** | Aperture, shutter speed, ISO, white balance, lens |
| **Camera identity** | Make, model, serial number, firmware |
| **Maker notes** | The camera manufacturer's private metadata block |
| **Copyright and credits** | Artist, copyright, software |
| **Color information** | Color profile, color space |
| **File timestamps** | Creation date and modification date on disk |
| **Any field not listed above** | All other EXIF, IPTC, XMP fields are preserved untouched |

---

## How the write works (the technical part)

Understanding the mechanics may help reassure you that nothing is being destroyed.

**The image data is never touched.**  
Raza Photos uses Apple's `CGImageDestinationCopyImageSource()` — a function specifically designed to rewrite a file's metadata sections while leaving the image data section completely alone. The compressed image (JPEG coefficients, HEIC frames, PNG chunks — whatever the format stores) is never decoded, never read, and never written by Raza Photos. It passes through the operation as an opaque block that neither Raza Photos nor Apple's framework inspects. There is no generation loss, no re-compression, and no risk of corruption from the image side. The only bytes that change are in the metadata area of the file.

**Merge, not replace.**  
The write uses merge semantics (`kCGImageDestinationMergeMetadata`). This means the existing metadata in the file is read first, then only the specific fields Raza Photos is writing are updated. Everything else remains exactly as it was — unknown fields, camera-specific data, and any fields Raza Photos doesn't manage are all preserved.

**Atomic write.**  
The updated file is assembled entirely in memory first. Only when the full result is ready does Raza Photos replace the file on disk — in a single atomic operation. There is no window where the file could be left in a partial or corrupted state. If anything goes wrong before the write completes, your original file is untouched.

**Security-scoped access.**  
On macOS, Raza Photos accesses your photo folders through security-scoped bookmarks — a macOS sandbox mechanism that grants access only to the folders you have explicitly added to your library. Files outside those folders cannot be touched.

---

## At-a-Glance Summary

| Data type | Written to |
|---|---|
| Date taken | EXIF `DateTimeOriginal`, TIFF `DateTime` |
| Title | IPTC `ObjectName`, XMP `dc:title` |
| GPS coordinates | EXIF GPS dictionary |
| Country | IPTC, XMP `Iptc4xmpExt`, ACDSee Places |
| City / locality | IPTC, XMP `Iptc4xmpExt`, ACDSee Places |
| People (names) | XMP `Iptc4xmpExt:PersonInImage` |
| Face regions | XMP `mwg-rs:Regions`, XMP `acdsee-rs:Regions` |
| Albums | IPTC Keywords, XMP `dc:subject`, XMP `lr:hierarchicalSubject`, ACDSee categories |
| Events | IPTC Keywords, XMP `dc:subject`, XMP `lr:hierarchicalSubject`, ACDSee categories |
| Burst ID | XMP `razaphotos:BurstID` (custom) |
| Burst representative | XMP `razaphotos:BurstRepresentative` (custom) |
| Pixel data | **Never modified** |
| Camera settings | **Never modified** |
| All other metadata | **Never modified** |
