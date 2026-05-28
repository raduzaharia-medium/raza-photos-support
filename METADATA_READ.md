# Photo Metadata: What Raza Photos Reads From Your Files

When you import a folder of photos, Raza Photos reads the metadata already embedded inside each image file—the structured data that cameras, photo editors, and organizing tools write alongside the pixels.

🔒 Privacy First: Raza Photos operates entirely locally. Nothing is invented, inferred, or sent to the cloud (except on-device reverse-geocoding, described below). Raza Photos simply reads what is already there and leaves your original files completely untouched.

---

## Background: Metadata standards

Photos can carry metadata in several overlapping standards. Most images contain several of these at once.

| Standard | Full name | Written by |
|---|---|---|
| **EXIF** | Exchangeable Image File Format | Cameras, smartphones |
| **IPTC** | International Press Telecommunications Council | Photo editing software, DAMs |
| **XMP** | Extensible Metadata Platform | Adobe and most modern editors |
| **ACDSee XML** | ACDSee's proprietary embedded XML fields | ACDSee software |
| **GPS dictionary** | Part of EXIF, stored separately | Cameras, smartphones with GPS |
| **Apple MakerNote** | Apple-private EXIF extension | iOS, macOS |
| **Filesystem** | File system attributes, not embedded in pixels | Operating system |

When the same piece of information (e.g., the date a photo was taken) appears in more than one standard, Raza Photos uses a strict priority order to pick the most reliable value. Raza Photos gathers all this information itself, without involving third-party metadata libraries. 

---

## What is read

### 📅 Date taken

There are many sources for this piece of metadata. Raza Photos tries each one in order, settling on the first that yields a valid date.

| Priority | Source | Technical detail |
|---|---|---|
| 1 | **EXIF** — original capture time | `kCGImagePropertyExifDateTimeOriginal` |
| 2 | **EXIF** — digitization time | `kCGImagePropertyExifDateTimeDigitized` |
| 3 | **TIFF** — general date/time | `kCGImagePropertyTIFFDateTime` |
| 4 | **XMP** — create date | `xmp:CreateDate` |
| 5 | **IPTC** — date created + time created | `kCGImagePropertyIPTCDateCreated` + `kCGImagePropertyIPTCTimeCreated` |
| 6 | **Filesystem** — file creation date | Used as a last resort if no embedded date exists |

- **Formating & Timezones:** EXIF dates are stored by cameras as 2024:10:31 12:38:24. IPTC dates use a compact YYYYMMDD form. XMP dates use the standardized ISO 8601 format. Raza Photos handles all of these automatically. When a timezone is absent, the time is treated as local time (no UTC conversion is applied).
- **Old Scans:** If your photos have no date metadata at all, the file's creation date on disk is used. You can always manually correct dates inside the app after import.

---

### 📍 GPS coordinates (location)

Coordinates are read from the **EXIF GPS dictionary** embedded in the file. This is the standard location for GPS data written by every camera and smartphone that records location.

| Field | Source | Technical detail |
|---|---|---|
| Latitude | EXIF GPS | `kCGImagePropertyGPSLatitude` + `kCGImagePropertyGPSLatitudeRef` (N/S) |
| Longitude | EXIF GPS | `kCGImagePropertyGPSLongitude` + `kCGImagePropertyGPSLongitudeRef` (E/W) |

- **Non-standard formats:** A small number of cameras write GPS in a non-standard DD,MM.mmmmH format. Raza Photos automatically detects and handles this variant.
- **On-Device Reverse Geocoding:** After coordinates are read, Raza Photos can optionally resolve them to a place name (country, city). This is done entirely locally on your device via Apple's framework. No network calls are ever made to a third-party service.

---

### 🗺️ Country and city

Place names can come from three sources. Raza Photos checks them in this order:

| Priority | Source | Technical detail |
|---|---|---|
| 1 | **IPTC** | `kCGImagePropertyIPTCCountryPrimaryLocationName` / `kCGImagePropertyIPTCCity` |
| 2 | **XMP** (IPTC 4 XMP Extension) | `Iptc4xmpExt:CountryName` / `Iptc4xmpExt:City` inside `LocationCreated` |
| 3 | **ACDSee categories** | Country and city encoded as a `Places` hierarchy in ACDSee's XML |

_**Note:** If none of these are present but GPS coordinates exist, the local reverse-geocoding feature will fill in the country and city automatically._

---

### 🏷️ Title

The photo's title (a short name, distinct from a description). Sources checked in order:

| Priority | Source | Technical detail |
|---|---|---|
| 1 | **IPTC** | `kCGImagePropertyIPTCObjectName` |
| 2 | **XMP** | `dc:title` (default language) |
| 3 | **Filename** | The file's name on disk, used as a fallback if no title is embedded |

_**Note:** The filename fallback ensures every imported photo always has a visible title instead of a blank space._

---

### 👥 People and face regions

Raza Photos reads person names that you have already tagged using other editing software. Face data is _not_ inferred by Raza Photos's own face recognition engine during import. 

| Priority | Source | Technical detail |
|---|---|---|
| 1 | **XMP** (IPTC 4 XMP Extension) | `Iptc4xmpExt:PersonInImage` array |
| 2 | **ACDSee regions** | Person names embedded in `acdsee-rs:Regions` XML face data |

- **Conflicts & Overlaps:** If a photo carries both XMP person names and ACDSee face regions, the XMP names take precedence for the library text, while the ACDSee regions provide the bounding boxes (the coordinates indicating where the face is in the frame).
- **Face Bounding Boxes:** ACDSee stores face rectangles relative to AppliedToDimensions using center-point coordinates (Center X, Center Y, Width, Height). Raza Photos converts these to standard top-left-origin bounding boxes when storing them in your local database.

---

### 🗂️ Albums and Events

Album and Event membership is read from **ACDSee's category system**, which stores hierarchies as an XML string in the `acdsee:categories` XMP field.

| Type | Source | Technical detail |
|---|---|---|
| **Albums** | ACDSee categories XML | Extracts names from the `Albums` subtree |
| **Events** | ACDSee categories XML | Extracts names from the `Events` subtree |

Each detected album or event name automatically creates a corresponding collection inside Raza Photos.

---

### 📸 Burst photography (iPhone burst mode)

When a photo is part of an iPhone burst sequence, Raza Photos identifies and groups them together so your main library stays clean.

| Field | Priority | Source | Technical detail |
|---|---|---|---|
| Burst ID | 1 | **Apple MakerNote** | Key 11 in the `MakerApple` raw properties dictionary |
| Burst ID | 2 | **XMP** (custom) | `razaphotos:BurstID` (written by Raza Photos itself on a previous import) |
| Burst representative | — | **XMP** (custom) | `razaphotos:BurstRepresentative` flag (written by Raza Photos) |

_The razaphotos: fields are custom tags written by the app so that your burst groupings survive seamlessly if you ever export and re-import the photo later._

---

### 💾 File dates

These attributes come directly from your computer's filesystem, not from inside the image file itself.

| Field | Source |
|---|---|
| File creation date | Filesystem (`creationDate` resource value) |
| File last modified date | Filesystem (`contentModificationDate` resource value) |

_These are highly useful as fallbacks for assets that pre-date digital cameras, such as scanned physical family photos._

---

## At-a-Glance Summary

Here is the quick-reference fallback order Raza Photos uses when scanning your library:

| Data Type | Primary Source | Fallback Order |
|---|---|---|
| Date taken | EXIF DateTimeOriginal | EXIF DateTimeDigitized → TIFF DateTime → XMP CreateDate → IPTC → file creation date |
| GPS coordinates | EXIF GPS dictionary | — |
| Country | IPTC | XMP (Iptc4xmpExt) → ACDSee → reverse geocoding |
| City / locality | IPTC | XMP (Iptc4xmpExt) → ACDSee → reverse geocoding |
| Title | IPTC ObjectName | XMP dc:title → filename |
| Person names | XMP PersonInImage | ACDSee region names |
| Face bounding boxes | ACDSee regions | — |
| Albums | ACDSee categories | — |
| Events | ACDSee categories | — |
| Burst ID | Apple MakerNote | XMP razaphotos:BurstID |
| File creation date | Filesystem | — |
| File modified date | Filesystem | — |
