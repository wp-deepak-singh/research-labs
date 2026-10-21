## Purpose

This document states what we plan to add to the eKYC product to catch tampered ID documents.
## Components

### 1. Record GPS location at capture

The app records the device location with each selfie and each ID capture. The backend stores the location with the record, as a watermark or as metadata.
### 2. Enforce live capture

The app captures from the camera only. It offers no gallery or file upload. It uploads the original camera JPEG untouched. This limits the chances to edit the image before upload, and it gives CAT-Net the original file it needs.

### 3. Detect digital edits with CAT-Net

CAT-Net runs on the backend. It finds spliced or edited regions from JPEG compression traces and outputs a heatmap ([CAT-Net paper](https://arxiv.org/abs/2108.12947), [CAT-Net repository](https://github.com/mjkwon2021/CAT-Net)). We derive a score from the heatmap and flag images above a threshold.

CAT-Net needs the original camera JPEG, with no resize, crop, or re-encode. It works best together with live capture. Files picked from a gallery are often already re-encoded, so CAT-Net catches less on them.

### 4. Read the passport chip with JMRTD (Android only)

The Android app uses JMRTD to read ICAO 9303 chips in passports and eIDs over NFC. It reads the MRZ data (DG1) and the face photo (DG2).

Chip reading works only in the Android app, because Web NFC does not support ISO-DEP ([Web NFC specification](https://w3c-cg.github.io/web-nfc/)).

JMRTD deprecated its own passive authentication. The backend verifies the chip signature instead, with [gmrtd](https://github.com/gmrtd/gmrtd) (MIT) or BouncyCastle. It checks the signature against the [BSI CSCA master list](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/ElekAusweise/CSCA/GermanMasterList.html), which BSI permits for commercial use with conditions.

### 5. Record a tilt video for human review

The app can record a 2 to 3 second video of the front and back while the user tilts the document.

Holograms and other security features change with the viewing angle, so a single photo cannot prove they are there. If the ID is flagged as suspicious, a human expert reviews the video.

### 6. Train a custom model on IDNet and FantasyID

We fine-tune a tamper detection model on two datasets:

- [IDNet](https://www.kaggle.com/datasets/chitreshkr/idnet-identity-document-analysis) is 49 GB. It has 837,060 images covering portrait substitution, text alteration, and morphing.
- [FantasyID](https://zenodo.org/records/17063366) is 2.6 GB and licensed CC-BY-4.0.

We also want to use these datasets to train OCR.

### 7. Check passport expiry

A rule check confirms that the expiry date has not passed and that the issue date comes before the expiry date.

### 8. Validate the MRZ structure and check digits

The backend checks the MRZ layout and its check digits against [ICAO Doc 9303 Part 4](https://www.icao.int/sites/default/files/publications/DocSeries/9303_p4_cons_en.pdf), section 4.2. It also confirms that the MRZ matches the printed fields.

### 9. Matching Face in selfie & ID document

https://github.com/opencv/opencv_zoo/tree/main/models/face_recognition_sface