# ID document tamper detection in eKYC

## Purpose

This document states what we plan to add to the eKYC product to catch tampered ID documents, and why. It is for the engineers and stakeholders who work on the web app, the Android app, and the backend. It describes intent, not implementation.

## Scope

The product serves users worldwide, not only in India. This document covers eight components, listed below. Aadhaar and UIDAI checks are out of scope because the product is not allowed to use them.

## Where we start

The backend already runs OpenCV blur and glare checks on uploaded photos. The frontend guides the user through capture and runs MediaPipe face detection. Neither one checks whether the document itself is genuine.

## Why we layer several checks

Pixel forensics alone is weak. On the FantasyID dataset, detectors miss about half of the fakes when they are tuned to wrongly flag 10% of genuine documents ([Korshunov et al., IJCB 2025](https://publications.idiap.ch/attachments/papers/2025/Korshunov_IJCB-2_2025.pdf)). So the plan combines different kinds of checks:

- Capture controls make it harder to submit an edited image.
- An edit detector looks for digital changes in the image.
- Chip data gives the backend a signed copy of the passport data.
- A tilt video lets a human reviewer check security features that a still photo cannot show.
- Rule checks catch impossible dates and bad passport numbers.

## Components

### 1. Record GPS location at capture

The app records the device location with each selfie and each ID capture. The backend stores the location with the record, as a watermark or as metadata.

India's Department of Telecommunications (DoT) set a precedent. Its 2019 Digital KYC rules require live photos watermarked with the CAF number, the GPS location, the point of sale, and the date and time ([DoT Digital KYC instructions](https://www.dot.gov.in/static/uploads/2026/02/94c5c5c3b29c65d994a1c0e6dce211fc.pdf)). A DoT amendment dated 21.11.2025 deleted the rule that required network-based location to detect GPS tampering ([DoT amendment](https://www.dot.gov.in/static/uploads/2026/03/c8bcad5a9a92a442074200dae2687c03.pdf)).

GPS can be spoofed, so location alone proves nothing about where the capture happened.

### 2. Enforce live capture

The app captures from the camera only. It offers no gallery or file upload. It uploads the original camera JPEG untouched, because CAT-Net needs that file.

NIST SP 800-63A-4 requires document presence checks ([NIST SP 800-63A-4](https://pages.nist.gov/800-63-4/sp800-63a/ial-general/)). Stronger options exist but are not yet researched in depth: signed uploads, Android device attestation, and C2PA content credentials. Pixel 10 phones already sign their photos with C2PA ([Google security blog](https://security.googleblog.com/2025/09/pixel-android-trusted-images-c2pa-content-credentials.html)).

### 3. Detect digital edits with CAT-Net

CAT-Net runs on the backend. It finds spliced or edited regions from JPEG compression traces and outputs a heatmap ([CAT-Net paper](https://arxiv.org/abs/2108.12947), [CAT-Net repository](https://github.com/mjkwon2021/CAT-Net)). We derive a score from the heatmap and flag images above a threshold.

CAT-Net needs the original JPEG, with no resize, crop, or re-encode. It needs a GPU and an old software stack, PyTorch 1.1 with CUDA 10.

CAT-Net was not trained on IDs. On the DocTamper document benchmark, it scores an F-measure of 0.700 ([DocTamper paper, CVPR 2023](https://openaccess.thecvf.com/content/CVPR2023/papers/Qu_Towards_Robust_Tampered_Text_Detection_in_Document_Image_New_Dataset_CVPR_2023_paper.pdf)). It is probably weak on photos of a screen or a printout. Nobody has tested that yet.

### 4. Read the passport chip with JMRTD

The Android app uses JMRTD to read ICAO 9303 chips in passports and eIDs over NFC. JMRTD handles BAC and PACE access control. It reads the MRZ data (DG1) and the face photo (DG2). The latest version is 0.8.9, from October 2025 ([JMRTD](https://jmrtd.org/about.shtml)).

Chip reading works only in the Android app, because Web NFC does not support ISO-DEP ([Web NFC specification](https://w3c-cg.github.io/web-nfc/)).

JMRTD deprecated its own passive authentication. The backend verifies the chip signature instead, with [gmrtd](https://github.com/gmrtd/gmrtd) (MIT) or BouncyCastle. It checks the signature against the [BSI CSCA master list](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/ElekAusweise/CSCA/GermanMasterList.html), which BSI permits for commercial use with conditions.

### 5. Record a tilt video for human review

The app records a 2 to 3 second video while the user tilts the document. A human reviewer watches the video.

Holograms, optically variable ink (OVI), and multiple laser images (MLI) change with the viewing angle. A single photo cannot prove they are there. ENISA recommends a tilted video of the front and back, plus review by a human expert ([ENISA, remote identity proofing](https://www.enisa.europa.eu/publications/remote-identity-proofing-attacks-countermeasures)). Regula's mobile SDK uses tilt and flash for the same checks ([Regula authenticity docs](https://docs.regulaforensics.com/develop/doc-reader-sdk/mobile/configuration/authenticity/)).

Phones have no UV or IR light, so the video is not full optical authentication ([ICAO Doc 9303 Part 2](https://www.icao.int/sites/default/files/publications/DocSeries/9303_p2_cons_en.pdf)).

### 6. Train a custom model on IDNet and FantasyID

No ready-made model trained on these datasets is licensed for commercial use. The Sunlight model that won DeepID is CC BY-NC, and TruFor and MMFusion are non-commercial. The plan is to fine-tune CAT-Net, which has a permissive licence, on both datasets for tamper detection.

- [IDNet](https://zenodo.org/records/13852207) has 837,060 images covering portrait substitution, text alteration, and morphing. The Zenodo record is CC0. The licence of the IDNet-2025 version on Hugging Face is unconfirmed.
- [FantasyID](https://zenodo.org/records/17063366) is CC-BY-4.0. Its validation split is restricted.

We also intend to use these datasets to train OCR. See the open questions.

### 7. Check passport expiry

A rule check confirms that the expiry date has not passed and that the issue date comes before the expiry date. The check reads the expiry date from the MRZ and compares it with the printed field. The MRZ expiry field has its own check digit ([ICAO Doc 9303](https://www.icao.int/publications/doc-series/doc-9303)).

### 8. Validate the passport number per country

Two checks apply to every country. The backend verifies the MRZ check digit on the document number ([ICAO Doc 9303](https://www.icao.int/publications/doc-series/doc-9303)). It also confirms that the MRZ number matches the printed number, because NIST requires a comparison of MRZ data with printed data.

Each issuing country also has its own number format. We have not researched those formats yet.

## Open questions

- **OCR training data.** IDNet and FantasyID are built for tamper detection. Nobody has checked whether they suit OCR training.
- **Passport number formats.** Validation per country needs a source for each issuing country. Not researched.
- **CAT-Net licence.** The README, dated 2026-08-05, says the code is Apache-2.0 and the weights are CC-BY-4.0, with commercial use allowed with attribution. The repository has no LICENSE file, and the training-data licences are unverified. Legal needs to check.
- **JMRTD licence.** JMRTD is LGPL-3.0. A closed-source commercial app can use it if we show a notice, ship the GPL and LGPL texts, let users replace the library, and allow reverse engineering for debugging ([LGPL-3.0](https://www.gnu.org/licenses/lgpl-3.0.html), [LGPL and Java](https://www.gnu.org/licenses/lgpl-java.html)). Its dependency cert-cvc is LGPL-2.1, and BouncyCastle is MIT. Legal needs to confirm whether APK code shrinking conflicts with the replace requirement.
- **Location data.** GPS spoofing is unsolved, and privacy rules on location data per country are not researched.
- **Capture hardening.** Signed uploads, Android device attestation, and C2PA need research.

## Out of scope

- **Aadhaar and UIDAI.** The product is not allowed to use them.
- **Vendor APIs.** We researched them but did not choose one. Didit offers 500 free checks a month, resale included. Other vendors offer trials only. A vendor API remains a fallback.

All sources accessed 21 October 2026.
