# Build brief: blur and image-quality checks with retake prompts in the eKYC web and Android apps

Earlier note: [Blur detection at capture](2026-10-19-blur-detection-at-capture.md).

## Short answer

Check every selfie and ID photo three times: on the live preview, on the captured still, and on the backend before approval, and show a specific retake reason on failure.
Android has first-party face and focus signals but no blur score, and browser camera control is uneven, so blur and glare scoring is either your own code or a commercial SDK such as BlinkID, Regula, Onfido or Jumio.
In-house costs nothing in licences but is the larger build; with either route, thresholds must be tuned on your own dealer devices and logs.

## How it works

The flow has four steps. The same flow applies to the selfie and to each side of the ID document.

1. Live preview checks, before the shutter. The app reads camera frames and checks them while the dealer frames the shot. Typical checks are one face in frame, face size, head pose, eyes open, document fully in frame, sharpness, and glare or darkness. The shutter stays disabled, or auto-capture waits, until the checks pass. This step runs on the device, because a round trip to a server for every frame is too slow. It is also the cheapest place to stop a bad photo.
2. Check on the captured photo. The full-resolution still can differ from the preview frame. The hand can move at the moment of capture, and the still has a different resolution ([CameraX configuration](https://developer.android.com/media/camera/camerax/configuration)). So the app re-scores the still before upload. This step also runs on the device.
3. Server-side re-check, before submission or approval. The backend scores the uploaded image again with its own checks. Three reasons make this step necessary. Old app versions and browsers you cannot control will send images. Client checks can be bypassed. And DoT does not allow activation when the photo is "non-recognizable" ([DoT letter 800-09/2023-AS.II, 31.08.2023, as enclosed by Railway Board](https://telerb.railnet.gov.in/raw-attachment/wiki/UpLoads/CUGKYCInstruction.pdf)). The backend is the only place that sees every submission. It is also where thresholds can change without an app release.
4. Retake prompt. A failed check names the problem: "Hold still", "Move closer", "Tilt the card away from the light", "Open your eyes". NIST separates a single quality score, which only says "recapture", from a quality vector, which tells the user what to fix ([NIST FATE Quality](https://pages.nist.gov/frvt/html/frvt_quality.html)). Vendor SDKs return reason codes for the same purpose, for example Jumio's BLURRY and GLARE ([Jumio Android integration guide](https://documentation.jumio.ai/docs/developer-resources/SDKs/mobile-sdk/mobile-sdk-android-master/docs/integration_guide)) and Onfido's `blurred_photo` and `glare_on_photo` ([Entrust document report](https://documentation.identity.entrust.com/guide/document-report/)). After a set number of failed attempts, the app should let the dealer submit with a flag for manual review instead of blocking the sale. This is a recommendation with no primary source.

Where each check runs, by platform:

| Step | Android app | Web app in a mobile browser | Backend |
|---|---|---|---|
| Live preview | CameraX frames plus ML Kit face detection plus your own sharpness and glare score, or a vendor SDK | Camera stream from `getUserMedia`, frames scored in the page with OpenCV.js or MediaPipe, or a vendor web SDK | Not used |
| Captured still | Same checks on the still | Same checks on the still | Not used |
| Re-check | Not used | Not used | OpenCV or OFIQ on your servers, or a vendor server API |
| Retake prompt | App UI | Page UI | Returns reason codes the apps show |

No standard or regulator document sets a numeric blur threshold. Focus measures are device-dependent, so thresholds come from your own data ([earlier note, section 3](2026-10-19-blur-detection-at-capture.md); [Pertuz, CVIA focus measure page](https://sites.google.com/view/cvia/focus-measure)).

## Options

### Option A. Build in-house with open-source tools

| Piece | What it gives you | Trade-off |
|---|---|---|
| OpenCV on Android and backend | Laplacian, Sobel and statistics functions to build sharpness, motion and glare scores ([OpenCV imgproc](https://docs.opencv.org/4.x/d4/d86/group__imgproc__filter.html)). Release 5.0.0 ships an Android SDK build ([OpenCV GitHub release 5.0.0](https://github.com/opencv/opencv/releases/tag/5.0.0)). Apache 2 licence from 4.5.0 onward ([OpenCV licence](https://opencv.org/license/)). | Free, and images stay with you. You write every check, the document detector and the reason codes. |
| OpenCV.js in the browser | The same functions compiled for the browser. A prebuilt `opencv.js` is downloadable from the docs site ([Using OpenCV.js](https://docs.opencv.org/4.x/d0/d84/tutorial_js_usage.html)). Memory must be freed by hand with `delete()` ([same page](https://docs.opencv.org/4.x/d0/d84/tutorial_js_usage.html)). | One scoring method on web and backend. Large download and speed on low-end phones are untested here (unverified). |
| BSI OFIQ for face quality | Reference implementation of ISO/IEC 29794-5 with per-measure scores, including Sharpness, eyes open, pose and exposure ([OFIQ README](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/README.md); [OFIQ config](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/data/ofiq_config.jaxn)). Builds for Android and iOS ([OFIQ mobile/BUILD.md](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/mobile/BUILD.md)). Code is MIT. Model files download from the ISO portal under their own licence files ([OFIQ LICENSE.md](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/LICENSE.md)). | Standards-aligned selfie scores on the backend. Selfie only. Speed on a phone is unmeasured (unverified). |

### Option B. First-party platform SDKs plus your own scoring

| Piece | What it gives you | Trade-off |
|---|---|---|
| Google ML Kit face detection | Face box, head angles, eye-open probabilities, on Android and iOS ([ML Kit face detection](https://developers.google.com/ml-kit/vision/face-detection)). No blur value ([`Face` reference](https://developers.google.com/android/reference/com/google/mlkit/vision/face/Face)). Free, on-device, needs Android API level 23 or higher ([ML Kit overview](https://developers.google.com/ml-kit/guides)). | Strong selfie geometry checks on Android. Does not run in a browser. ML Kit is described as a mobile SDK for Android and iOS ([ML Kit overview](https://developers.google.com/ml-kit/guides)). |
| Google ML Kit document scanner | Ready-made scan UI with auto-capture, edge detection and cropping, Android only. Needs Google Play services and at least 1.7 GB device RAM ([ML Kit document scanner on Android](https://developers.google.com/ml-kit/vision/doc-scanner/android)). Its default mode can erase stains and fingers by ML. `SCANNER_MODE_BASE` limits edits to crop and rotate ([ML Kit document scanner](https://developers.google.com/ml-kit/vision/doc-scanner)). | Quick document capture on Android. No quality score, no web version, and some low-end dealer phones fall below the RAM floor. |
| CameraX | Tap-to-focus and metering, focus result, exposure compensation, 640x480 analysis frames by default. Applied "with best effort" per device ([CameraX configuration](https://developer.android.com/media/camera/camerax/configuration)). `isFocusSuccessful()` returns true on cameras without autofocus ([FocusMeteringResult](https://developer.android.com/reference/androidx/camera/core/FocusMeteringResult)). | Camera control and focus state, no blur score. Fixed-focus front cameras give no focus signal. |
| MediaPipe Face Detector for web | Google's in-browser face detector with face boxes and six key points, through the `@mediapipe/tasks-vision` package ([MediaPipe Face Detector, web guide](https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector/web_js)). Detection blocks the main thread unless moved to a web worker ([same page](https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector/web_js)). Apache 2.0 repository licence ([google-ai-edge/mediapipe](https://github.com/google-ai-edge/mediapipe)). | Fills the web gap that ML Kit leaves. Gives fewer face attributes than ML Kit, and no blur score. |
| Browser camera APIs | `getUserMedia` works only on HTTPS or localhost ([MDN getUserMedia](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia)). The Image Capture spec defines `takePhoto()`, `grabFrame()` and camera constraints such as `focusMode`, `focusDistance` and `pointsOfInterest` ([W3C MediaStream Image Capture, Editor's Draft 23 April 2025](https://w3c.github.io/mediacapture-image/)). | Lets the page read frames and take a still. Focus control is not dependable across browsers, as listed below. |

Browser coverage for the Image Capture API, from MDN's compatibility data ([MDN browser-compat-data, ImageCapture](https://github.com/mdn/browser-compat-data/blob/main/api/ImageCapture.json); [MDN ImageCapture](https://developer.mozilla.org/en-US/docs/Web/API/ImageCapture)):

- Chrome 59+ and Chrome for Android support `ImageCapture`. `takePhoto()` with settings came in Chrome 60.
- Safari and Safari on iOS support `ImageCapture` from 18.4. `grabFrame()` arrived in Safari 26.
- Firefox has it only behind the `dom.imagecapture.enabled` preference. `getPhotoCapabilities()` and `grabFrame()` are not supported.
- Samsung Internet and Android WebView mirror Chrome.
- On iOS, every browser uses WebKit, so iOS Chrome behaves like Safari ([App Review Guideline 2.5.6](https://developer.apple.com/app-store/review/guidelines/)). Apple allows other engines only by entitlement in the EU and Japan.
- MDN lists `focusMode`, `focusDistance` and `pointsOfInterest` as image-track constraints ([MDN MediaTrackConstraints](https://developer.mozilla.org/en-US/docs/Web/API/MediaTrackConstraints)). I found no per-browser compatibility entry for these constraints in MDN's data, so per-browser focus support is unverified. The page must ask the camera with `getCapabilities()` and treat focus control as optional.
- The spec's `sharpness` is a camera processing setting, not a measurement of how sharp the image is ([W3C Image Capture, section 10](https://w3c.github.io/mediacapture-image/)). The browser gives no blur score.
- The W3C document is an Editor's Draft and says not to cite it other than as work in progress ([W3C Image Capture](https://w3c.github.io/mediacapture-image/)).

### Option C. Commercial eKYC and IDV SDKs

| Vendor | Quality checks, per its own docs | Platforms | Where images are processed | Pricing |
|---|---|---|---|---|
| Microblink BlinkID | Scores every frame for blur, glare and poor lighting. Sensitivity `off`, `low`, `mid`, `high`, and flags to reject blurred, glared or badly lit frames ([BlinkID image quality](https://docs.microblink.com/blinkid/image-quality); [Scanning settings](https://docs.microblink.com/blinkid/scanning-settings)). Built-in UX tells the user to move closer, hold steady or avoid glare ([same page](https://docs.microblink.com/blinkid/image-quality)). Documents only. I found no selfie quality check in BlinkID. | Web, Android, iOS, React Native, Flutter ([BlinkID intro](https://docs.microblink.com/blinkid)). Web needs Chrome 112+, Firefox 115+ or Safari 16.4+, and a 720p camera ([web minimum requirements](https://docs.microblink.com/blinkid/sdk/web/min-reqs)). Lists Aadhaar, PAN, Voter ID, Indian passport and several state driving licences ([supported documents JSON v8001.0.0](https://docs.microblink.com/blinkid/supported-docs/v8001.0.0/supported-documents.json)). | "All extraction happens locally; no data from the document leaves the device" ([BlinkID intro](https://docs.microblink.com/blinkid)). | Pricing not public. The plans page asks you to contact the team ([Microblink plans](https://microblink.com/plans/)). |
| Regula Document Reader SDK and Face SDK | Document checks for resolution, focus, glare, perspective angle, bounds, portrait presence, brightness, colour and occlusion ([Regula image quality assessment](https://docs.regulaforensics.com/develop/doc-reader-sdk/overview/image-quality/)). Web Components expose `focusCheck`, `glaresCheck`, `dpiThreshold`, default 150, and `angleThreshold`, default 5 degrees, ([Web Components parameters](https://docs.regulaforensics.com/develop/doc-reader-sdk/web-components/configuration/parameters/)). Face SDK reports `BlurLevel`, eyes, pose, lighting and background per face ([Face image quality characteristics](https://docs.regulaforensics.com/develop/face-sdk/web-service/development/usage/face-detection/face-image-quality-check/)). | Mobile SDK, Web Components in the browser, Web Service on Linux, containers or Windows ([Regula licensing](https://docs.regulaforensics.com/develop/doc-reader-sdk/overview/licensing/)). | The Mobile licence works offline on the device. The Web Service can run on your own servers, and the online licence then sends only heartbeat data such as licence ID, IP and version ([Regula licensing](https://docs.regulaforensics.com/develop/doc-reader-sdk/overview/licensing/)). | Pricing not public. Cost depends on deployment size, modules and payment terms ([Regula pricing article](https://support.regulaforensics.com/hc/en-us/articles/360036943712-Pricing-SDK-Licensing-Models)). Free trial of 30 days and 2,000 transactions ([Regula licensing](https://docs.regulaforensics.com/develop/doc-reader-sdk/overview/licensing/)). |
| Onfido (Entrust) | The full Android SDK has "on-device, real-time glare and blur detection". The lighter core SDK does its document validations mostly on the backend ([Onfido Android SDK](https://documentation.identity.entrust.com/sdk/android/)). The Web SDK sends images to the Onfido API for quality feedback and shows the error to the user ([Onfido Web SDK](https://documentation.identity.entrust.com/sdk/web/)). The document report returns `image_quality` reasons such as `blurred_photo`, `glare_on_photo` and `dark_photo` ([Entrust document report](https://documentation.identity.entrust.com/guide/document-report/)). | Web, Android, iOS. | Onfido cloud. Region choices are EU, US and Canada ([Onfido API reference](https://documentation.identity.entrust.com/api/latest/)). No India region is listed. | Pricing not public. I found no price list. entrust.com blocked automated fetches. |
| Jumio | Instant Feedback runs a usability check on each photo and asks for a new one if it is unusable. Reason codes include 2001 BLURRY and 2006 GLARE ([Jumio Android integration guide](https://documentation.jumio.ai/docs/developer-resources/SDKs/mobile-sdk/mobile-sdk-android-master/docs/integration_guide)). Server usability labels include BLURRED, GLARE and BAD_QUALITY_IMAGE, for ID and selfie ([Jumio Usability](https://documentation.jumio.ai/docs/references/capabilities/usability)). The Web SDK package defines an `idSharpness` setting and the same BLURRED and GLARE codes ([`@jumio/websdk` 0.6.2 type definitions](https://cdn.jsdelivr.net/npm/@jumio/websdk@0.6.2/index2.d.ts)). | Web, Android, iOS. | Jumio cloud. Data centres `us`, `eu` and `sgp` ([Jumio Web SDK](https://documentation.jumio.ai/docs/developer-resources/SDKs/web-sdk/introduction-web)). | No public price list. Jumio's own AWS Marketplace listing offers "Jumio Go - 25,000 transactions" at $37,500.00 per 12 months ([AWS Marketplace, sold by Jumio Corporation](https://aws.amazon.com/marketplace/pp/prodview-wyxkbmnbhansc)). That listing does not say which SDK features are included. |
| IDfy | The Android "Right First Time" SDK does blur and lighting detection on documents, and blur, face orientation, multiple-face and closed-eye detection on selfies. It shows a review screen and asks for a recapture. Supported document types include Aadhaar, PAN, voter ID, driving licence and passport ([IDfy Capture SDK, Android](https://eve-docs.idfy.com/pages/capture_sdk/capture_sdk.html)). A server API checks whether a document is "blurry/readable" ([IDfy Document Image Validation](https://www.idfy.com/document-validation/)). | Android SDK. I found no public IDfy web capture SDK with quality checks (unverified). | Images upload to IDfy servers. The sample link points to Google Cloud Storage ([IDfy Capture SDK](https://eve-docs.idfy.com/pages/capture_sdk/capture_sdk.html)). Storage region not stated (unverified). | Pricing not public. Keys come from IDfy by email ([IDfy Capture SDK](https://eve-docs.idfy.com/pages/capture_sdk/capture_sdk.html)). |
| HyperVerge | The product page claims "instant user feedback during document capture for issues like blur, glare, partial, or obscured ID" ([HyperVerge ID card quality checks](https://hyperverge.co/in/integrations-marketplace/id-card-ocr-api/)). Developer docs at documentation.hyperverge.co need a login (HTTP 401 on 2026-10-19). Its Flutter package changelog lists a selfie retake screen for blurred or no-face images ([`hypersnapsdk_flutter` changelog](https://pub.dev/packages/hypersnapsdk_flutter/changelog)). That package has no verified publisher on pub.dev, so treat it as unverified. | Android, iOS and Web SDKs are named on its own pages ([HyperSnap web demo](https://hypersnapweb.hyperverge.co/)). Per-platform quality features are unverified. | Not stated publicly (unverified). | Pricing not public. Plans are Start, Grow and Enterprise. Start has a 1-month free sandbox trial, and pricing is volume-tiered ([HyperVerge pricing](https://hyperverge.co/pricing/)). |
| Signzy | Server API only, in the public docs I found. ID Intelligence returns `imageQuality` with `ImageFocus`, `ImageGlares`, `ImageResolution`, `Perspective` and `Bounds` ([Signzy ID Intelligence Advance](https://docs.signzy.com/us-apis/id-intelligence-advance)). A separate option scores sharpness, text quality, brightness and compression, with a threshold you set ([Signzy image quality analysis](https://docs.signzy.com/us-apis/image-quality-analysis)). | Backend API. These pages sit under "US APIs". Availability for Indian IDs is unverified. I found no public Signzy capture SDK with blur checks (unverified). | Signzy servers. Region not stated (unverified). | Pricing not public. I found no pricing page. |

One-line trade-offs:

- Option A, in-house open source: no licence fees and full data control, but you build and maintain every check on three runtimes.
- Option B, first-party SDKs plus own scoring: free and on-device for faces and camera control on Android, but blur, glare and the whole web path are still yours to build.
- Option C, commercial SDK: checks, reason codes and capture UI ship ready, but you pay per use or per licence, depend on the vendor, and may send images to a foreign cloud.

## Development requirements

### Android app

- Replace or wrap the current capture screen. With Option A or B, add a CameraX analysis stream, ML Kit face detection for selfies, and your own sharpness, glare and document-in-frame checks. Keep only the latest frame for analysis so the preview does not lag ([ML Kit face detection, Android](https://developers.google.com/ml-kit/vision/face-detection/android)).
- With Option C, embed the vendor SDK and its licence. Regula's mobile licence is bound to the application ID ([Regula licensing](https://docs.regulaforensics.com/develop/doc-reader-sdk/overview/licensing/)). IDfy's SDK needs an account ID and token, and Firebase set up in the app ([IDfy Capture SDK](https://eve-docs.idfy.com/pages/capture_sdk/capture_sdk.html)).
- Re-score the full-resolution still before upload.
- Show specific retake messages. Cap retries, then allow a flagged submission.
- Read thresholds from the server at start-up, with defaults built into the app. Firebase Remote Config is one first-party way to do this without a release ([Firebase Remote Config](https://firebase.google.com/docs/remote-config)). Your own config endpoint works too.
- Plan a fallback for devices under ML Kit's limits: API level 23 for ML Kit ([ML Kit overview](https://developers.google.com/ml-kit/guides)), and 1.7 GB RAM plus Play services for the document scanner ([ML Kit document scanner on Android](https://developers.google.com/ml-kit/vision/doc-scanner/android)).

### Web app

- Serve the capture page over HTTPS. If it runs in an iframe, the parent must grant camera access with Permissions Policy ([MDN getUserMedia](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia)).
- Feature-detect `ImageCapture` and camera capabilities. Fall back to drawing video frames on a canvas where `grabFrame()` is missing ([MDN browser-compat-data](https://github.com/mdn/browser-compat-data/blob/main/api/ImageCapture.json)).
- Score frames in the page with OpenCV.js or MediaPipe in a web worker ([MediaPipe web guide](https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector/web_js)), or embed a vendor web SDK. BlinkID's web SDK loads WebAssembly resources you host ([BlinkID docs index](https://docs.microblink.com/llms.txt)). Jumio's web SDK is one `<jumio-sdk>` element with a data-centre and a token ([Jumio Web SDK](https://documentation.jumio.ai/docs/developer-resources/SDKs/web-sdk/introduction-web)). Regula Web Components need each domain licensed ([Regula licensing](https://docs.regulaforensics.com/develop/doc-reader-sdk/overview/licensing/)).
- Use the same retake messages and the same server-fed thresholds as Android.

### Backend

- Add a quality endpoint, or a step in the submission pipeline, that re-scores each image before the CAF goes to approval. Use OpenCV or OFIQ in-house, or a vendor server API: the Regula Web Service, Onfido's document report, Jumio's usability result, IDfy's Document Validation API, or Signzy's image quality option.
- Issue short-lived vendor tokens from the backend. Do not ship vendor API secrets in the web page. Jumio's web SDK takes a token from an account request made by your server ([Jumio Web SDK](https://documentation.jumio.ai/docs/developer-resources/SDKs/web-sdk/introduction-web)). Onfido issues separate sandbox and live API tokens ([Onfido API reference](https://documentation.identity.entrust.com/api/latest/)).
- Store a threshold configuration with a version number, and serve it to both apps.
- Return machine-readable reason codes that the apps map to messages.

### Integration points and data

- Data in: the selfie JPEG and the document front and back images, plus device model, OS and browser version, app or SDK version, and camera facing.
- Data out: pass or fail per check, raw scores, reason codes, threshold-config version, attempt number, and the final decision.
- Logs for tuning: store scores and reasons for every attempt, including failed ones. Join them later with OCR success, face-match result and manual review outcome. Set thresholds from those outcomes, using NIST's approach of measuring how much the error rate drops when you reject the lowest-quality images ([NIST FATE Quality](https://pages.nist.gov/frvt/html/frvt_quality.html)). Keep rejected images only if you have a lawful purpose and a retention period. Scores alone are enough for most tuning. This is a recommendation with no primary source.
- Auth and licences: vendor licence files or keys per app ID and per web domain, server-side API tokens, and a key rotation plan.

### Compliance

These apply if the operator is an Indian telecom licensee. Confirm that first.

- DoT photo rules. A live photo taken at the point of sale, with no other person in frame. Documents captured from above with no skew or tilt, in proper light so that photo and documents are "clearly readable and identifiable". A watermark carries the CAF number, GPS, PoS name and code, date and time ([DoT Alternate Digital KYC instructions, 03.04.2019](https://www.dot.gov.in/static/uploads/2026/02/94c5c5c3b29c65d994a1c0e6dce211fc.pdf)). No activation if the auto-populated photo is "non-recognizable" or does not match the live photo ([DoT letter, 31.08.2023, as enclosed by Railway Board](https://telerb.railnet.gov.in/raw-attachment/wiki/UpLoads/CUGKYCInstruction.pdf)). None of these give a numeric blur threshold. Run the quality check on the image before the watermark is drawn, or keep the watermark band out of the scored region, so the overlay does not distort the score. This is a recommendation with no primary source.
- UIDAI, if Aadhaar card images are captured. UIDAI tells requesting entities not to store copies of Aadhaar without first masking the first 8 digits ([UIDAI circular, 31 October 2022](https://uidai.gov.in/images/DosandDonots_for_Requesting_Entities.pdf)). The Sharing of Information Regulations bar making Aadhaar numbers public unless redacted ([Aadhaar (Sharing of Information) Regulations, 2016, reg. 6](https://www.uidai.gov.in/images/The_Aadhaar_Sharing_of_information_regulation_2016.pdf)). Decide whether the quality check runs before or after masking. A cloud vendor that receives the raw image sees the full number.
- RBI. The RBI KYC Master Direction uses the same "proper light" wording for Digital KYC, but it is addressed to RBI-regulated entities, not telecom licensees ([RBI Master Direction on KYC](https://www.rbi.org.in/scripts/BS_ViewMasDirections.aspx?id=11566)). It matters only if the same flow also serves a bank or wallet.
- DPDP Act 2023. Data fiduciaries must use reasonable security safeguards, including for processing by a Data Processor (s. 8(5)). Processors must be engaged under a valid contract (s. 8(2)). Personal data must be erased once the purpose is served, unless a law requires retention (s. 8(7)) ([DPDP Act 2023](https://www.meity.gov.in/static/uploads/2024/06/2bf1f0e9f04e6fb4f8fef35e82c42aa5.pdf)). The government may restrict transfers to notified countries (s. 16).
- DPDP Rules 2025. Rule 6 sets minimum safeguards: encryption, masking or tokens, access control, access logs, backups, a one-year retention of such logs and data for investigating unauthorised access, and safeguard clauses in processor contracts. Rule 7 requires breach notice to the Board within 72 hours with details. Rule 15 adds requirements on making data available to foreign states ([DPDP Rules 2025](https://www.meity.gov.in/static/uploads/2025/11/53450e6e5dc0bfa85ebd78686cadad39.pdf)).
- DPDP timing. Sections 3 to 17 of the Act, which include s. 8 and s. 16, take effect 18 months after the 13 November 2025 notification ([G.S.R. 843(E)](https://www.meity.gov.in/static/uploads/2025/11/c56ceae6c383460ca69577428d36828b.pdf)). Rules 3 and 5 to 16 follow the same 18 months ([DPDP Rules 2025, rule 1](https://www.meity.gov.in/static/uploads/2025/11/53450e6e5dc0bfa85ebd78686cadad39.pdf)). By my count that is 13 May 2027. Anything built now will run under these duties.
- Data residency. An older DoT amendment to the UASL licence bars licensees from transferring "user information" to any person or place outside India ([DoT UASL amendments, 14.12.2005](https://www.dot.gov.in/static/uploads/2026/06/d6060d507355be6b98182baee7d8b890.pdf)). I did not confirm whether the operator's current licence or authorisation carries the same clause (unverified). Have legal check it before choosing a vendor that processes images in the EU, US, Canada or Singapore. On-device options (ML Kit, BlinkID) and self-hosted options (OpenCV, OFIQ, the Regula Web Service) avoid the question.
- SDK licences. OpenCV is Apache 2 from 4.5.0 ([OpenCV licence](https://opencv.org/license/)). OFIQ code is MIT, and its models carry separate licences ([OFIQ LICENSE.md](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/LICENSE.md)). MediaPipe is Apache 2.0 ([MediaPipe repository](https://github.com/google-ai-edge/mediapipe)). ML Kit is under the Google APIs Terms with no reverse engineering. It processes images on-device, but sends usage metrics to Google, and you must tell users about that where the law requires ([ML Kit terms](https://developers.google.com/ml-kit/terms)). Jumio's web package is marked `UNLICENSED`, which means proprietary ([npm registry, @jumio/websdk](https://registry.npmjs.org/@jumio/websdk/latest)). Commercial terms come from the vendor contract.

## Resources needed

Accounts and access:

- Vendor sandboxes for any Option C shortlist. Regula offers a 30-day, 2,000-transaction trial through its Client Portal ([Regula licensing](https://docs.regulaforensics.com/develop/doc-reader-sdk/overview/licensing/)). HyperVerge offers a 1-month sandbox trial ([HyperVerge pricing](https://hyperverge.co/pricing/)). Microblink licences come from its developer hub ([BlinkID intro](https://docs.microblink.com/blinkid)). Onfido issues sandbox API tokens ([Onfido API reference](https://documentation.identity.entrust.com/api/latest/)). IDfy keys come from eve.support@idfy.com ([IDfy Capture SDK](https://eve-docs.idfy.com/pages/capture_sdk/capture_sdk.html)). I found no self-serve Jumio trial; access appears to go through sales (unverified).
- A Firebase project if you use Remote Config for thresholds.
- Vendor agreements: commercial terms, a data processing contract that meets DPDP s. 8(2) and Rule 6(1)(f), the processing region, and retention and deletion terms.

Test devices and browsers:

- A low-end Android range from your actual dealer base. Include phones with fixed-focus front cameras, phones near Android API level 23, and phones under 1.7 GB RAM.
- Chrome for Android, Samsung Internet and Android WebView, if the web app runs inside another app.
- iPhones on Safari 18.4 or later and on older versions, if dealers use iPhones. `grabFrame()` needs Safari 26 ([MDN compat data](https://github.com/mdn/browser-compat-data/blob/main/api/ImageCapture.json)).
- Firefox for Android, as the weakest Image Capture case.

Data:

- A labelled set of real dealer captures, good and bad, per ID type and per device class, to set thresholds.

Skills and roles:

- An Android developer, a web front-end developer, and a backend developer.
- For Option A or B, an engineer with image-processing experience to build and tune checks.
- A QA engineer with a device lab.
- An analyst to tune thresholds from logs.
- Legal and privacy review for DPDP, UIDAI and licence conditions.
- Procurement for Option C.

Effort size per option. These are judgements, not measured figures.

| Option | Size | Reason |
|---|---|---|
| A. In-house open source | Large | You build sharpness, glare, motion and document-in-frame checks for Android, the browser and the backend, write all the reason codes, and tune them per device class with no vendor help. |
| B. First-party SDKs plus own scoring | Medium | ML Kit and CameraX cover selfie geometry and camera control on Android. Blur and glare scoring is still custom, and the web needs a separate path, because ML Kit does not run in a browser. |
| C. Commercial SDK | Small to medium for engineering, plus procurement and legal time | Capture UI, checks and reason codes ship ready. Work is embedding on two platforms, token and licence handling, backend wiring, and a data-flow review. It grows if the SDK replaces existing capture screens or if images must stay in India. |

Costs from official pages only:

- OpenCV, OFIQ code and MediaPipe: free under open licences.
- ML Kit: offered "at no cost" ([ML Kit overview](https://developers.google.com/ml-kit/guides)).
- Jumio: $37,500.00 per 12 months for 25,000 Jumio Go transactions, on Jumio's own AWS Marketplace listing ([AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-wyxkbmnbhansc)).
- Microblink, Regula, Onfido, IDfy, HyperVerge and Signzy: pricing not public.

## Risks and open questions

Risks:

- Device variance. Focus-measure behaviour depends on the device and scene ([CVIA](https://sites.google.com/view/cvia/focus-measure)). CameraX applies focus "with best effort" ([CameraX](https://developer.android.com/media/camera/camerax/configuration)), and fixed-focus cameras always report focus success ([FocusMeteringResult](https://developer.android.com/reference/androidx/camera/core/FocusMeteringResult)). One global threshold will be wrong for some phones.
- Browser camera limits. Focus control is optional and its support per browser is unverified. Firefox has Image Capture only behind a flag. All iOS browsers run on WebKit. The Image Capture spec is still an Editor's Draft. The web path must work with no camera control at all.
- Threshold tuning. No source gives a numeric cutoff. Vendor sensitivity levels such as BlinkID's `low`, `mid` and `high` also need tuning on your traffic ([BlinkID scanning settings](https://docs.microblink.com/blinkid/scanning-settings)).
- False rejects that hurt dealer throughput. Microblink says rejecting low-quality frames makes capture slower in difficult conditions ([BlinkID image quality](https://docs.microblink.com/blinkid/image-quality)). Start in log-only mode, measure, then turn on blocking with a retry cap and a flagged override. This is a recommendation with no primary source.
- Cost of retries. Per-transaction vendor pricing may count retake attempts (unverified, depends on contract).
- Vendor lock-in. Reason codes, licences and capture UI differ by vendor. Keep your own reason-code layer and threshold config so you can swap vendors.
- Data leaving India. Onfido's and Jumio's listed regions do not include India.
- Aadhaar exposure. A cloud quality check on an unmasked Aadhaar image sends the full number to a third party.
- Low-end Android. Devices under 1.7 GB RAM cannot use the ML Kit document scanner. Phones without Play services cannot use the Play-services delivered ML Kit models.

Facts you need to supply:

- The current stack. What the web app is built with and whether it runs in an iframe or a WebView. The Android minSdk and camera library. The backend language and hosting region.
- Monthly volumes of KYC captures and current retake or rejection rates. These are needed for vendor quotes and for sizing the backend re-check.
- Which IDs you accept, for example Aadhaar, PAN, voter ID, passport or driving licence, and whether both sides are captured.
- The dealer device and browser mix, including the share of iPhones.
- Whether images may leave India under your licence and your contracts.
- Whether the flow must also serve any RBI-regulated partner.
- Whether checks must work offline, and how that fits the offline-first work.

Open questions:

- Does the operator's current licence or authorisation still bar transfer of user information outside India?
- Which browsers support `focusMode` and `pointsOfInterest` on dealers' phones? Test this directly.
- How fast are OpenCV.js, MediaPipe and OFIQ on the low-end phones you pick?
- Do the vendor web SDKs score frames in the browser or on their servers? Onfido's web flow uses the API. Regula and BlinkID imply in-browser work. Confirm each in a trial.
- What is HyperVerge's documented quality-check behaviour behind its login-gated docs?

## Sources

All accessed 2026-10-19.

Standards, specs and browser data

- W3C, MediaStream Image Capture, Editor's Draft 23 April 2025. https://w3c.github.io/mediacapture-image/
- MDN, MediaStream Image Capture API. https://developer.mozilla.org/en-US/docs/Web/API/MediaStream_Image_Capture_API
- MDN, ImageCapture. https://developer.mozilla.org/en-US/docs/Web/API/ImageCapture
- MDN, MediaDevices.getUserMedia(). https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia
- MDN, MediaTrackConstraints. https://developer.mozilla.org/en-US/docs/Web/API/MediaTrackConstraints
- MDN browser-compat-data, api/ImageCapture.json. https://github.com/mdn/browser-compat-data/blob/main/api/ImageCapture.json
- Apple, App Review Guidelines, 2.5.6. https://developer.apple.com/app-store/review/guidelines/
- NIST, FATE Quality. https://pages.nist.gov/frvt/html/frvt_quality.html
- Pertuz, CVIA Focus Measure project page. https://sites.google.com/view/cvia/focus-measure

Open-source and first-party SDKs

- OpenCV, License. https://opencv.org/license/
- OpenCV, imgproc Image Filtering. https://docs.opencv.org/4.x/d4/d86/group__imgproc__filter.html
- OpenCV, Using OpenCV.js. https://docs.opencv.org/4.x/d0/d84/tutorial_js_usage.html
- OpenCV, GitHub release 5.0.0. https://github.com/opencv/opencv/releases/tag/5.0.0
- BSI OFIQ, README.md. https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/README.md
- BSI OFIQ, LICENSE.md. https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/LICENSE.md
- BSI OFIQ, data/ofiq_config.jaxn. https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/data/ofiq_config.jaxn
- BSI OFIQ, mobile/BUILD.md. https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/mobile/BUILD.md
- Google ML Kit, overview. https://developers.google.com/ml-kit/guides
- Google ML Kit, Terms and Privacy. https://developers.google.com/ml-kit/terms
- Google ML Kit, Face detection. https://developers.google.com/ml-kit/vision/face-detection
- Google ML Kit, Detect faces on Android. https://developers.google.com/ml-kit/vision/face-detection/android
- Google ML Kit, Face class reference. https://developers.google.com/android/reference/com/google/mlkit/vision/face/Face
- Google ML Kit, Document scanner. https://developers.google.com/ml-kit/vision/doc-scanner
- Google ML Kit, Document scanner on Android. https://developers.google.com/ml-kit/vision/doc-scanner/android
- Android Developers, CameraX configuration options. https://developer.android.com/media/camera/camerax/configuration
- Android Developers, FocusMeteringResult. https://developer.android.com/reference/androidx/camera/core/FocusMeteringResult
- Google AI Edge, MediaPipe Face Detector guide for Web. https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector/web_js
- Google AI Edge, MediaPipe repository. https://github.com/google-ai-edge/mediapipe
- Firebase, Remote Config. https://firebase.google.com/docs/remote-config

Commercial vendors

- Microblink, BlinkID Introduction. https://docs.microblink.com/blinkid
- Microblink, Handle low-quality or blurry images. https://docs.microblink.com/blinkid/image-quality
- Microblink, Scanning settings. https://docs.microblink.com/blinkid/scanning-settings
- Microblink, Minimum requirements for the web SDK. https://docs.microblink.com/blinkid/sdk/web/min-reqs
- Microblink, docs index. https://docs.microblink.com/llms.txt
- Microblink, supported documents JSON v8001.0.0. https://docs.microblink.com/blinkid/supported-docs/v8001.0.0/supported-documents.json
- Microblink, Plans. https://microblink.com/plans/
- Regula, Image Quality Assessment. https://docs.regulaforensics.com/develop/doc-reader-sdk/overview/image-quality/
- Regula, Service parameters of Document Reader Web Components. https://docs.regulaforensics.com/develop/doc-reader-sdk/web-components/configuration/parameters/
- Regula, Document Reader SDK Licensing Models. https://docs.regulaforensics.com/develop/doc-reader-sdk/overview/licensing/
- Regula, Face Image Quality Characteristics. https://docs.regulaforensics.com/develop/face-sdk/web-service/development/usage/face-detection/face-image-quality-check/
- Regula Help Center, Pricing: SDK Licensing Models. https://support.regulaforensics.com/hc/en-us/articles/360036943712-Pricing-SDK-Licensing-Models
- Entrust (Onfido), Android SDK reference. https://documentation.identity.entrust.com/sdk/android/
- Entrust (Onfido), Web SDK reference. https://documentation.identity.entrust.com/sdk/web/
- Entrust (Onfido), Document report. https://documentation.identity.entrust.com/guide/document-report/
- Entrust (Onfido), API reference v3.6. https://documentation.identity.entrust.com/api/latest/
- Jumio, Usability. https://documentation.jumio.ai/docs/references/capabilities/usability
- Jumio, Android SDK Integration Guide. https://documentation.jumio.ai/docs/developer-resources/SDKs/mobile-sdk/mobile-sdk-android-master/docs/integration_guide
- Jumio, Getting Started with the Web SDK. https://documentation.jumio.ai/docs/developer-resources/SDKs/web-sdk/introduction-web
- Jumio, @jumio/websdk 0.6.2 type definitions. https://cdn.jsdelivr.net/npm/@jumio/websdk@0.6.2/index2.d.ts
- npm registry, @jumio/websdk latest metadata. https://registry.npmjs.org/@jumio/websdk/latest
- AWS Marketplace, Jumio Identity Verification listing. https://aws.amazon.com/marketplace/pp/prodview-wyxkbmnbhansc
- IDfy, Capture SDK (Android). https://eve-docs.idfy.com/pages/capture_sdk/capture_sdk.html
- IDfy, Document Image Validation. https://www.idfy.com/document-validation/
- HyperVerge, ID Card OCR Quality Checks. https://hyperverge.co/in/integrations-marketplace/id-card-ocr-api/
- HyperVerge, Pricing. https://hyperverge.co/pricing/
- HyperVerge, HyperSnap Web SDK demo. https://hypersnapweb.hyperverge.co/
- pub.dev, hypersnapsdk_flutter changelog (no verified publisher). https://pub.dev/packages/hypersnapsdk_flutter/changelog
- Signzy, ID Intelligence Advance. https://docs.signzy.com/us-apis/id-intelligence-advance
- Signzy, Image Quality Analysis. https://docs.signzy.com/us-apis/image-quality-analysis

Regulators and law (India)

- DoT, Instructions for Alternate Digital KYC Process, 03.04.2019. https://www.dot.gov.in/static/uploads/2026/02/94c5c5c3b29c65d994a1c0e6dce211fc.pdf
- DoT letter 800-09/2023-AS.II dated 31.08.2023, as enclosed in Railway Board letter dated 12.09.2024. https://telerb.railnet.gov.in/raw-attachment/wiki/UpLoads/CUGKYCInstruction.pdf
- DoT, Amendments in Unified Access Service Licence Agreement, 14.12.2005. https://www.dot.gov.in/static/uploads/2026/06/d6060d507355be6b98182baee7d8b890.pdf
- MeitY, Digital Personal Data Protection Act, 2023 (Gazette, 11.08.2023). https://www.meity.gov.in/static/uploads/2024/06/2bf1f0e9f04e6fb4f8fef35e82c42aa5.pdf
- MeitY, Digital Personal Data Protection Rules, 2025 (G.S.R. 846(E), 13.11.2025). https://www.meity.gov.in/static/uploads/2025/11/53450e6e5dc0bfa85ebd78686cadad39.pdf
- MeitY, Commencement notification G.S.R. 843(E), 13.11.2025. https://www.meity.gov.in/static/uploads/2025/11/c56ceae6c383460ca69577428d36828b.pdf
- UIDAI, Circular on Dos and Don'ts for Requesting Entities, 31.10.2022. https://uidai.gov.in/images/DosandDonots_for_Requesting_Entities.pdf
- UIDAI, Aadhaar (Sharing of Information) Regulations, 2016. https://www.uidai.gov.in/images/The_Aadhaar_Sharing_of_information_regulation_2016.pdf
- RBI, Master Direction on KYC, 2016, updated. https://www.rbi.org.in/scripts/BS_ViewMasDirections.aspx?id=11566
