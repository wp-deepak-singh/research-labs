## Purpose

This document states what we plan to add to the eKYC product to catch tampered ID documents, and why. It is for the engineers and stakeholders who work on the web app, the Android app, and the backend.
## Components

### 1. Record GPS location at capture

The app records the device location with each selfie and each ID capture. The backend stores the location with the record, as a watermark or as metadata.

### 2. Enforce live capture

The app captures from the camera only. It offers no gallery or file upload. It uploads the original camera JPEG untouched. This is to reduce the ground for tampering with the image.
### 3. CAT-Net, if live capture cannot be enforced

CAT-Net runs on the backend. It finds spliced or edited regions from JPEG compression traces and outputs a heatmap ([CAT-Net paper](https://arxiv.org/abs/2108.12947), [CAT-Net repository](https://github.com/mjkwon2021/CAT-Net)). We derive a score from the heatmap and flag images above a threshold.
### 4. Read the passport chip with JMRTD, in case if we can are using mobile devices to capture passports

The Android app uses JMRTD to read ICAO 9303 chips in passports and eIDs over NFC. It reads the MRZ data (DG1) and the face photo (DG2). 

Chip reading works only in the Android app, because Web NFC does not support ISO-DEP ([Web NFC specification](https://w3c-cg.github.io/web-nfc/)).

JMRTD deprecated its own passive authentication. The backend verifies the chip signature instead, with [gmrtd](https://github.com/gmrtd/gmrtd) (MIT) or BouncyCastle. It checks the signature against the [BSI CSCA master list](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/ElekAusweise/CSCA/GermanMasterList.html), which BSI permits for commercial use with conditions.

### 5. Record a tilt video for human review

The app can record a 2 to 3 second video while the user tilts the document. A human reviewer watches the video.
A single photo cannot prove they are there. A tilted video of the front and back, plus review by a human expert in case if the ID has been tagged as problematic.
### 6. Train a custom model on IDNet and FantasyID

- [IDNet](https://www.kaggle.com/datasets/chitreshkr/idnet-identity-document-analysis) (49GB) has 837,060 images covering portrait substitution, text alteration, and morphing.
- [FantasyID](https://zenodo.org/records/17063366)  2.6GB.

We can also use these datasets to train OCR.

### 7. Check passport expiry

A rule check confirms that the expiry date has not passed and that the issue date comes before expiry date.

### 8. Validate the passport structure and MRZ numbers

https://www.icao.int/sites/default/files/publications/DocSeries/9303_p4_cons_en.pdf
section 4.2 is all about how the MRZ is structured.