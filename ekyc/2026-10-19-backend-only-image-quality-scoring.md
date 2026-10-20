# Build brief: backend-only quality scoring of eKYC selfie and ID images

Earlier notes: [Blur detection build brief for web and Android](2026-10-19-blur-detection-web-android-build-brief.md) and [Blur detection at capture](2026-10-19-blur-detection-at-capture.md).

## Short answer

A backend-only check is one service, one set of thresholds and nothing in the apps to tamper with, and it catches every image from every app version and browser.
It gives no guidance while the dealer frames the shot, so every bad photo costs a full upload before the retake prompt, and it scores images the phone or browser has already compressed.
It is the smallest build and a sound first step, but keep the free Android signals (ML Kit face detection, CameraX focus state) and a fixed high JPEG quality to cut wasted uploads.

## How it works

### The flow

1. The dealer takes the selfie or the ID photo in the web or Android app as today. The app runs no quality check.
2. The app uploads the image to your backend.
3. The backend scores the image. Document checks are blur, glare, exposure, resolution and document-in-frame. Selfie checks are face found, one face, face size, pose, eyes open, blur and exposure.
4. The backend returns a result per image: pass or fail, the raw scores, and reason codes.
5. On a fail, the app shows a retake message that names the problem, for example "Hold still" or "Tilt the card away from the light". NIST separates a single quality score, which only says "recapture", from a quality vector that tells the user what to fix ([NIST FATE Quality](https://pages.nist.gov/frvt/html/frvt_quality.html)). Reason codes are the vector.
6. After a set number of failures, the app lets the dealer submit with a flag for manual review. This is a recommendation with no primary source.

### Synchronous or asynchronous

Score synchronously, inside the upload request, so the retake prompt appears while the customer is still at the counter. The cloud APIs below are request and response calls that return the scores in the reply. AWS describes DetectFaces as "a stateless API operation" that "does not persist any data" ([AWS DetectFaces](https://docs.aws.amazon.com/rekognition/latest/APIReference/API_DetectFaces.html)). Google Vision online requests are processed in memory and not stored ([Google Vision data usage](https://docs.cloud.google.com/vision/docs/data-usage)).

An asynchronous check that runs after the CAF is submitted cannot ask for a retake in the same visit. It only works as an audit before approval. Keep it, if at all, as a second pass.

### Latency

Wait time for the dealer is upload time, plus scoring time, plus the reply.

- Upload time dominates on slow links. As arithmetic, a 1.5 MB JPEG is 12 megabits, so about 12 seconds at 1 Mbit/s before any scoring starts. I found no primary source for current upload speeds at rural points of sale (unverified).
- Scoring time per image for OpenCV measures, OFIQ or any cloud API is not published in the sources I read (unverified). Measure it in a trial.
- A cloud API adds a second network hop from your backend to the provider. Region choice affects it. India regions exist for AWS Rekognition and Azure Face, not for Google Vision, as listed under Options.

### What you gain

- One implementation. The same checks apply to the web app, the Android app, old app versions and any browser.
- No SDK in the apps. No app-size growth, no per-device performance work and no per-domain or per-app-ID vendor licences in the client.
- Central thresholds. Change a threshold on the server, with a version number, and both apps follow on the next upload without a release.
- Nothing on the client to tamper with. Client checks can be bypassed. The backend sees every submission. DoT bars activation when the photo is "non-recognizable" ([DoT letter 800-09/2023-AS.II, 31.08.2023, as enclosed by Railway Board](https://telerb.railnet.gov.in/raw-attachment/wiki/UpLoads/CUGKYCInstruction.pdf)), so this check is needed in every design, including the three-stage one.
- One log. Every attempt, its scores and its outcome land in one place for threshold tuning.

### What you lose

- No live guidance. The dealer gets no "move closer" or "hold still" while framing. Every problem is found after the shutter and the upload.
- Retakes cost a round trip. Each failed image needs a new capture and a new upload. On slow links this is where dealer time goes.
- Bandwidth spent on bad images. Every rejected image is a full upload, paid by the dealer's data plan.
- The backend scores a processed image. By the time it arrives, the phone or browser has compressed and maybe resized it.
  - Android CameraX encodes JPEG at quality 95 in `CAPTURE_MODE_MINIMIZE_LATENCY` and 100 in `CAPTURE_MODE_MAXIMIZE_QUALITY` unless the app sets another value with `setJpegQuality` ([CameraX `ImageCapture.Builder`](https://developer.android.com/reference/androidx/camera/core/ImageCapture.Builder)). Apps that re-compress or resize before upload lower this further.
  - In a browser, an image drawn to a canvas and exported with `toBlob` or `toDataURL` uses a quality from 0.0 to 1.0. If none is given, the browser uses "an implementation-specific default" ([HTML Standard, canvas](https://html.spec.whatwg.org/multipage/canvas.html)). The web app must set the quality and the size explicitly.
  - Microblink tells customers of its server API to send at least 1080p, not to downscale, to use a high-quality JPEG setting, and to send SDK-captured images as lossless PNG ([Microblink on-prem image formats](https://docs.microblink.com/on-prem/image-formats)). It wants an uncropped, unprocessed camera frame ([Microblink image requirements](https://docs.microblink.com/verify/image-requirements)).
  - Regula's resolution check fails images under a DPI threshold, default 150 ([Regula Web Components parameters](https://docs.regulaforensics.com/develop/doc-reader-sdk/web-components/configuration/parameters/)). Downscaling can push a sharp document under it.
  - Sharpness scores depend on window size and image size. Gradient and Laplacian focus measures are the most sensitive to a smaller window ([Pertuz, CVIA focus measure page](https://sites.google.com/view/cvia/focus-measure)). scikit-image says `blur_effect` results are comparable only with the same `h_size` ([scikit-image `skimage.measure`](https://scikit-image.org/docs/stable/api/skimage.measure.html)). Fix the input size before scoring.
  - JPEG artefacts are a known quality factor. OFIQ includes a `CompressionArtifacts` measure for faces ([OFIQ config](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/data/ofiq_config.jaxn)). I found no primary source that quantifies how much a given JPEG quality setting moves a Laplacian or Tenengrad score (unverified).

### Hybrid: keep what the platform gives for free

You can keep the backend as the only scorer and still add three cheap client steps. None of them needs a vendor SDK.

- Android selfie. ML Kit face detection is free, runs on the device and needs API level 23 or higher ([ML Kit overview](https://developers.google.com/ml-kit/guides)). It returns the face box, head angles and eye-open probabilities, but no blur value ([ML Kit face detection](https://developers.google.com/ml-kit/vision/face-detection); [`Face` reference](https://developers.google.com/android/reference/com/google/mlkit/vision/face/Face)). Use it to block the shutter when there is no face, more than one face, or closed eyes. It sends usage metrics to Google ([ML Kit terms](https://developers.google.com/ml-kit/terms)).
- Android focus. CameraX reports whether focus succeeded, applied "with best effort" ([CameraX configuration](https://developer.android.com/media/camera/camerax/configuration)). `isFocusSuccessful()` returns true on cameras without autofocus ([FocusMeteringResult](https://developer.android.com/reference/androidx/camera/core/FocusMeteringResult)), so a pass means little on fixed-focus front cameras.
- Capture settings, both apps. Keep JPEG quality high, do not downscale below what the backend needs, and send the original frame. On the web, request a camera resolution with `getUserMedia` constraints ([MDN getUserMedia](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia)) and set the `toBlob` quality ([HTML Standard, canvas](https://html.spec.whatwg.org/multipage/canvas.html)).

The web app gets no free face detector from the platform. MediaPipe Face Detector runs in the browser under Apache 2.0, but adding it is a client build, not a platform freebie ([MediaPipe web guide](https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector/web_js)).

### Comparison with the three-stage approach

This compares the design in this note with the design in the [earlier build brief](2026-10-19-blur-detection-web-android-build-brief.md). The judgements are mine and not from a source.

|                             | Backend only       | Backend plus free client signals                       | Three-stage (preview, still, backend)   |
| --------------------------- | ------------------ | ------------------------------------------------------ | --------------------------------------- |
| Live guidance while framing | None               | Face present and focus state on Android only           | Full, on both apps                      |
| Retake found                | After upload       | After upload, except no-face and closed-eye on Android | Before upload, backend catches the rest |
| Wasted uploads              | Every bad image    | Fewer on Android selfies                               | Fewest                                  |
| Client build                | Retake screen only | Small Android change                                   | Largest: two platforms, three runtimes  |
| Thresholds                  | One place          | Mostly one place                                       | Client and server, kept in sync         |
| Tamper resistance           | Full               | Full, the backend still decides                        | Full, the backend still decides         |
| Backend check               | Required           | Required                                               | Required                                |

The backend check is the common part. Building it first does not waste work if you add client checks later.

## Options

Each row lists what the service scores, where it runs and what it costs, from its own pages.

### Self-hosted

| Option                                                  | What it scores                                                                                                                                                                                                                                                                                                                                                                                                                           | Where it runs                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Cost                                                                                                                                                                                                                                                                            | Trade-off                                                                                                    |
| ------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| OpenCV measures                                         | Building blocks for sharpness (Laplacian, Sobel), glare and exposure statistics ([OpenCV imgproc](https://docs.opencv.org/4.x/d4/d86/group__imgproc__filter.html)). No ready score or document detector.                                                                                                                                                                                                                                 | Your servers. Apache 2 from 4.5.0 ([OpenCV licence](https://opencv.org/license/)).                                                                                                                                                                                                                                                                                                                                                                             | Free.                                                                                                                                                                                                                                                                           | Full control and data stays home. You write every check and reason code and tune them yourself.              |
| scikit-image `blur_effect`                              | A no-reference blur score from 0 for none to 1 for maximal ([scikit-image `skimage.measure`](https://scikit-image.org/docs/stable/api/skimage.measure.html)).                                                                                                                                                                                                                                                                            | Your servers, Python.                                                                                                                                                                                                                                                                                                                                                                                                                                          | Free.                                                                                                                                                                                                                                                                           | A ready blur number. Blur only, and comparable only at a fixed `h_size`.                                     |
| BSI OFIQ                                                | ISO/IEC 29794-5 reference measures for faces, including Sharpness, EyesOpen, HeadPose, exposure, SingleFacePresent and CompressionArtifacts ([OFIQ README](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/README.md); [OFIQ config](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/data/ofiq_config.jaxn)).                                                                                                                      | Your servers. Builds on OpenCV and ONNX Runtime ([OFIQ BUILD.md](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/BUILD.md)). Code MIT, models under their own licences ([OFIQ LICENSE.md](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/LICENSE.md)).                                                                                                                                                                                                  | Free code.                                                                                                                                                                                                                                                                      | Standards-aligned selfie vector. Selfie only, C++ integration work, speed per image unmeasured (unverified). |
| Microblink on-prem                                      | `/api/v3/extract` and `/api/v3/verify` return `imageAssessment` with image quality, cropped document and hand presence checks. Sensitivity from Level1 to Level10 or Disabled ([Microblink on-prem](https://docs.microblink.com/on-prem); [on-prem OpenAPI](https://docs.microblink.com/on-prem/api/latest/openapi.json)). Documents only.                                                                                               | One container in your cloud or on your hardware. "Nothing leaves your network" ([Microblink on-prem](https://docs.microblink.com/on-prem)).                                                                                                                                                                                                                                                                                                                    | Not public ([Microblink plans](https://microblink.com/plans/)).                                                                                                                                                                                                                 | Vendor document checks with images in India. Licence cost unknown, no selfie check.                          |
| Regula Document Reader Web Service and Face web service | Documents: resolution, focus, glare, perspective, bounds, brightness and occlusion ([Regula image quality assessment](https://docs.regulaforensics.com/develop/doc-reader-sdk/overview/image-quality/)). Faces: `BlurLevel`, recommended 0 to 0.5, plus eyes, pose and lighting ([Regula face image quality](https://docs.regulaforensics.com/develop/face-sdk/web-service/development/usage/face-detection/face-image-quality-check/)). | Your servers on Linux, containers or Windows. The online licence sends heartbeat data only ([Regula licensing](https://docs.regulaforensics.com/develop/doc-reader-sdk/overview/licensing/)). The Face web service installs as containers, on Linux or on Windows, in CPU or GPU versions, and face detection needs no extra components ([Regula Face web service installation](https://docs.regulaforensics.com/develop/face-sdk/web-service/installation/)). | Not public ([Regula pricing article](https://support.regulaforensics.com/hc/en-us/articles/360036943712-Pricing-SDK-Licensing-Models)). Trial 30 days and 2,000 transactions ([Regula licensing](https://docs.regulaforensics.com/develop/doc-reader-sdk/overview/licensing/)). | Document and selfie checks with reason codes, images stay home. Two products to license and run.             |

### Cloud APIs

| Option                                         | What it scores                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Data region and India                                                                                                                                                                                                                                                                                                                                                                                                         | Official price                                                                                                                                                                                                                                                                                                                                                               | Trade-off                                                                                                                    |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| AWS Rekognition DetectFaces                    | Per face `Quality` with Brightness and Sharpness from 0 to 100, plus pose and landmarks ([ImageQuality](https://docs.aws.amazon.com/rekognition/latest/APIReference/API_ImageQuality.html); [DetectFaces](https://docs.aws.amazon.com/rekognition/latest/APIReference/API_DetectFaces.html)). JPEG or PNG, 5 MB as raw bytes ([Rekognition limits](https://docs.aws.amazon.com/rekognition/latest/dg/limits.html)).                                                                                                                  | Available in ap-south-1 Mumbai ([AWS price list, ap-south-1](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonRekognition/current/ap-south-1/index.json)). AWS AI services may store content for service improvement, possibly in another Region, unless you set an opt-out policy ([AWS AI services opt-out](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_ai-opt-out.html)). | ap-south-1, per image: $0.00125 up to 1M a month, $0.0010 for 1M to 5M, $0.00075 for 5M to 35M, $0.0003125 above ([AWS price list, ap-south-1](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonRekognition/current/ap-south-1/index.json)). Free tier 1,000 images a month for 12 months ([Rekognition pricing](https://aws.amazon.com/rekognition/pricing/)). | India region and numeric face scores. Only sharpness and brightness, so you set the pass marks, and the opt-out must be set. |
| AWS Rekognition DetectLabels, IMAGE_PROPERTIES | Whole-image Sharpness, Brightness and Contrast, and foreground and background sharpness ([Rekognition image properties](https://docs.aws.amazon.com/rekognition/latest/dg/labels-detect-labels-image.html)). Usable on document photos.                                                                                                                                                                                                                                                                                              | Same as above.                                                                                                                                                                                                                                                                                                                                                                                                                | ap-south-1, per image: $0.000938 up to 1M, $0.00075 for 1M to 5M, $0.000563 for 5M to 35M, $0.0002344 above ([AWS price list, ap-south-1](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonRekognition/current/ap-south-1/index.json)). No free tier for image properties ([Rekognition pricing](https://aws.amazon.com/rekognition/pricing/)).                 | Cheap document sharpness in India. No glare, document-in-frame or resolution check.                                          |
| Azure AI Face detection                        | `blur` and `exposure` levels with values from 0 to 1, `noise`, `occlusion`, `headPose` and `qualityForRecognition` low, medium or high ([Azure face detection concepts](https://learn.microsoft.com/en-us/azure/ai-services/face/concept-face-detection); [detection models](https://learn.microsoft.com/en-us/azure/ai-services/face/how-to/specify-detection-model)). JPEG, PNG, GIF or BMP up to 6 MB ([Face detect REST](https://learn.microsoft.com/en-us/rest/api/face/face-detection-operations/detect?view=rest-face-v1.2)). | Central India has price rows ([Azure Retail Prices API](https://prices.azure.com/api/retail/prices)). Whether a Face resource can be created there for you is unverified. Face is Limited Access, for Microsoft managed customers and partners by intake form ([Azure face detection concepts](https://learn.microsoft.com/en-us/azure/ai-services/face/concept-face-detection)).                                             | Central India, per 1,000 transactions: $1.00 up to 1M, $0.80 for 1M to 5M, $0.60 for 5M to 100M, $0.40 above. A free tier exists ([Azure Retail Prices API](https://prices.azure.com/api/retail/prices); [Face pricing](https://azure.microsoft.com/en-us/pricing/details/cognitive-services/face-api/)).                                                                    | A ready "good enough for recognition" verdict. Approval gate, and faces only.                                                |
| Google Cloud Vision face detection             | `blurredLikelihood` and `underExposedLikelihood` per face, bucketed from VERY_UNLIKELY to VERY_LIKELY, plus roll, pan and tilt ([AnnotateImageResponse](https://cloud.google.com/vision/docs/reference/rest/v1/AnnotateImageResponse)). Image Properties returns dominant colours, not sharpness ([same page](https://cloud.google.com/vision/docs/reference/rest/v1/AnnotateImageResponse)).                                                                                                                                        | Global, EU and US endpoints only. No India endpoint ([Vision regional endpoints](https://cloud.google.com/vision/docs/ocr)). Online requests are not stored ([Vision data usage](https://docs.cloud.google.com/vision/docs/data-usage)).                                                                                                                                                                                      | First 1,000 units a month free, $1.50 per 1,000 up to 5M, $0.60 above. Each feature billed separately ([Vision pricing](https://cloud.google.com/vision/pricing)).                                                                                                                                                                                                           | Stable coarse buckets. No India region, faces only, no numeric score.                                                        |
| Microblink Verify cloud API                    | Same `imageAssessment` family as on-prem ([Verify API v3 OpenAPI](https://docs.microblink.com/verify/openapi-v3-cloud.json)).                                                                                                                                                                                                                                                                                                                                                                                                        | US east, EU and Canada ([Microblink Verify API regions](https://docs.microblink.com/verify/api)). No India region.                                                                                                                                                                                                                                                                                                            | Not public ([Microblink plans](https://microblink.com/plans/)).                                                                                                                                                                                                                                                                                                              | Vendor document checks with no hosting. Images leave India.                                                                  |
| Signzy                                         | ID Intelligence `imageQuality` with ImageFocus, ImageGlares, ImageResolution, Perspective and Bounds ([Signzy ID Intelligence Advance](https://docs.signzy.com/us-apis/id-intelligence-advance)). Image quality analysis with a threshold you set ([Signzy image quality analysis](https://docs.signzy.com/us-apis/image-quality-analysis)).                                                                                                                                                                                         | Pages sit under "US APIs". Region and Indian ID support not stated (unverified).                                                                                                                                                                                                                                                                                                                                              | Not public.                                                                                                                                                                                                                                                                                                                                                                  | Ready document reason codes. Region unknown.                                                                                 |
| IDfy Document Image Validation                 | Checks whether a document image is readable or blurry ([IDfy Document Image Validation](https://www.idfy.com/document-validation/)).                                                                                                                                                                                                                                                                                                                                                                                                 | IDfy servers. Region not stated (unverified).                                                                                                                                                                                                                                                                                                                                                                                 | Not public.                                                                                                                                                                                                                                                                                                                                                                  | Indian vendor familiar with Indian IDs. Few documented fields, region unknown.                                               |
| HyperVerge                                     | Product page claims blur, glare, partial and obscured ID checks ([HyperVerge ID card quality checks](https://hyperverge.co/in/integrations-marketplace/id-card-ocr-api/)). Developer docs need a login, so the server API fields are unverified.                                                                                                                                                                                                                                                                                     | Not stated publicly (unverified).                                                                                                                                                                                                                                                                                                                                                                                             | Start, Grow and Enterprise plans. Start has a 1-month sandbox ([HyperVerge pricing](https://hyperverge.co/pricing/)).                                                                                                                                                                                                                                                        | Indian vendor. Server-side behaviour unverifiable without a login.                                                           |
| Onfido (Entrust) and Jumio                     | Onfido document report returns `image_quality` reasons such as `blurred_photo` and `glare_on_photo` ([Entrust document report](https://documentation.identity.entrust.com/guide/document-report/)). Jumio returns usability labels such as BLURRED and GLARE ([Jumio Usability](https://documentation.jumio.ai/docs/references/capabilities/usability)).                                                                                                                                                                             | Onfido EU, US and Canada ([Onfido API reference](https://documentation.identity.entrust.com/api/latest/)). Jumio us, eu and sgp ([Jumio Web SDK](https://documentation.jumio.ai/docs/developer-resources/SDKs/web-sdk/introduction-web)). No India region.                                                                                                                                                                    | Onfido not public. Jumio Go is $37,500.00 per 12 months for 25,000 transactions on its AWS Marketplace listing ([AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-wyxkbmnbhansc)).                                                                                                                                                                            | Full IDV verdicts. Expensive for a quality gate alone, images leave India.                                                   |

A workable pairing is one document scorer and one face scorer. Among the self-hosted options, only Regula covers both. For example, AWS Rekognition DetectFaces plus IMAGE_PROPERTIES in Mumbai with your own glare and document-in-frame checks, or OFIQ for selfies plus Microblink or Regula on-prem for documents. This pairing is my suggestion, not from a source.

## Development requirements

### Scoring service and API contract

- One endpoint, or one step in the upload handler, that takes an image and returns a verdict.
- Input: the image bytes as uploaded, the image type (selfie, document front, document back), the document type, the attempt number, and capture metadata (device model, OS or browser version, app version, camera facing, JPEG quality and pixel size).
- Output: `pass` or `fail`, the raw score per check, a list of reason codes such as `BLUR`, `GLARE`, `TOO_DARK`, `LOW_RESOLUTION`, `DOC_NOT_IN_FRAME`, `NO_FACE`, `MULTIPLE_FACES` and `EYES_CLOSED`, the threshold-config version, and an image ID. The code names are examples, not a standard.
- Normalise before scoring. Decode, fix orientation, crop to the face or document region, and resize to one standard size, so that scores compare across devices ([CVIA focus measure page](https://sites.google.com/view/cvia/focus-measure); [scikit-image `skimage.measure`](https://scikit-image.org/docs/stable/api/skimage.measure.html)).
- Respect provider input limits: 5 MB raw bytes for Rekognition ([Rekognition limits](https://docs.aws.amazon.com/rekognition/latest/dg/limits.html)), 6 MB for Azure Face ([Face detect REST](https://learn.microsoft.com/en-us/rest/api/face/face-detection-operations/detect?view=rest-face-v1.2)), 25 MiB for Microblink on-prem ([Microblink on-prem image formats](https://docs.microblink.com/on-prem/image-formats)).
- Set a timeout. If the scorer or the provider is down, accept the image with a flag for manual review rather than blocking the sale. This is a recommendation with no primary source.

### Place in the submission flow

- Score each image when it is uploaded, before the dealer moves to the next step. Do not wait for the whole CAF.
- Score before the DoT watermark is drawn, or exclude the watermark band, so the overlay does not distort the score. The watermark carries CAF number, GPS, PoS details and time ([DoT Alternate Digital KYC instructions, 03.04.2019](https://www.dot.gov.in/static/uploads/2026/02/94c5c5c3b29c65d994a1c0e6dce211fc.pdf)).
- Keep the final pre-approval check. Only images that passed, or were flagged after the retry cap, go to approval.

### App changes

- Both apps: send the original capture at a fixed, high JPEG quality and a known size. On Android, set `setJpegQuality` or use the quality capture mode ([CameraX `ImageCapture.Builder`](https://developer.android.com/reference/androidx/camera/core/ImageCapture.Builder)). On the web, set the `toBlob` quality ([HTML Standard, canvas](https://html.spec.whatwg.org/multipage/canvas.html)).
- Handle the new response. Show a progress state during scoring, map each reason code to a short retake message, return to the camera for that image only, and keep the images that passed.
- Count attempts and, after the cap, offer a flagged submission.
- Keep the messages in a server-fed table so wording changes do not need a release.
- Optional hybrid: ML Kit face detection and the CameraX focus result on Android.

### Threshold configuration

- A versioned config on the server per check, per image type and, if needed, per device class or document type.
- Start in log-only mode, then switch checks to blocking one at a time. This is a recommendation with no primary source.
- Set cutoffs from outcomes. NIST's method measures how much error falls when the lowest-quality images are rejected ([NIST FATE Quality](https://pages.nist.gov/frvt/html/frvt_quality.html)). Vendor sensitivity levels, such as Microblink's Level1 to Level10, also need tuning on your traffic ([Microblink on-prem OpenAPI](https://docs.microblink.com/on-prem/api/latest/openapi.json)).

### Logging

- Log every attempt with the scores, reason codes, config version, capture metadata, latency of upload and scoring, and the final outcome (OCR success, face match, manual review result).
- Keep access logs as DPDP Rule 6 requires, for one year ([DPDP Rules 2025, rule 6](https://www.meity.gov.in/static/uploads/2025/11/53450e6e5dc0bfa85ebd78686cadad39.pdf)).

### Storage and retention of rejected images

- Rejected images are personal data with no KYC use once a better image passes. DPDP requires erasure once the purpose is served unless a law requires retention (s. 8(7)) ([DPDP Act 2023](https://www.meity.gov.in/static/uploads/2024/06/2bf1f0e9f04e6fb4f8fef35e82c42aa5.pdf)).
- Default to deleting rejected images and keeping only scores. If you keep a sample to tune thresholds, define the purpose, the period and access controls in writing first. This is a recommendation with no primary source.
- Rejected Aadhaar images must not be stored unmasked ([UIDAI circular, 31 October 2022](https://uidai.gov.in/images/DosandDonots_for_Requesting_Entities.pdf)).
- Whether DoT requires keeping rejected attempts as part of the CAF record is unverified.

### Compliance

These apply if the operator is an Indian telecom licensee. Confirm that first.

- DoT. Photos must be clearly readable and identifiable, documents captured without skew or tilt in proper light ([DoT Alternate Digital KYC instructions, 03.04.2019](https://www.dot.gov.in/static/uploads/2026/02/94c5c5c3b29c65d994a1c0e6dce211fc.pdf)). No activation if the photo is "non-recognizable" ([DoT letter, 31.08.2023, as enclosed by Railway Board](https://telerb.railnet.gov.in/raw-attachment/wiki/UpLoads/CUGKYCInstruction.pdf)). Neither sets a numeric threshold.
- UIDAI, when Aadhaar images go to a third party. Do not store Aadhaar copies without masking the first 8 digits ([UIDAI circular, 31 October 2022](https://uidai.gov.in/images/DosandDonots_for_Requesting_Entities.pdf)). Aadhaar numbers must not be published unless redacted, and must not be sent over the internet unless the channel is secure and encrypted ([Aadhaar (Sharing of Information) Regulations, 2016, reg. 6](https://www.uidai.gov.in/images/The_Aadhaar_Sharing_of_information_regulation_2016.pdf)). A cloud API that scores the raw card sees the full number. Either mask before sending, which changes what is scored, or keep Aadhaar scoring self-hosted.
- DPDP Act 2023. Processors under a valid contract (s. 8(2)), reasonable security safeguards (s. 8(5)), erasure (s. 8(7)), and possible restriction of transfers to notified countries (s. 16) ([DPDP Act 2023](https://www.meity.gov.in/static/uploads/2024/06/2bf1f0e9f04e6fb4f8fef35e82c42aa5.pdf)).
- DPDP Rules 2025. Rule 6 safeguards and processor contract clauses, Rule 7 breach notice to the Board within 72 hours, Rule 15 on making data available to foreign states ([DPDP Rules 2025](https://www.meity.gov.in/static/uploads/2025/11/53450e6e5dc0bfa85ebd78686cadad39.pdf)). Sections 3 to 17 and most rules take effect 18 months after 13 November 2025 ([G.S.R. 843(E)](https://www.meity.gov.in/static/uploads/2025/11/c56ceae6c383460ca69577428d36828b.pdf)), which by my count is 13 May 2027.
- Data residency. A DoT amendment to the UASL licence bars transferring "user information" outside India ([DoT UASL amendments, 14.12.2005](https://www.dot.gov.in/static/uploads/2026/06/d6060d507355be6b98182baee7d8b890.pdf)). Whether the operator's current licence carries the same clause is unverified. Self-hosted options and AWS Mumbai keep processing in India. Azure Central India is unconfirmed. Google Vision, Microblink cloud, Onfido and Jumio have no India region. For AWS, set the AI services opt-out so content is not stored for service improvement in another Region ([AWS AI services opt-out](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_ai-opt-out.html)).

## Resources needed

Accounts, keys and sandboxes:

- AWS account with Rekognition in ap-south-1, an Organizations AI opt-out policy, and IAM keys held by the backend only. Free tier of 1,000 images a month for 12 months ([Rekognition pricing](https://aws.amazon.com/rekognition/pricing/)).
- Azure subscription plus the Face Limited Access approval ([Azure face detection concepts](https://learn.microsoft.com/en-us/azure/ai-services/face/concept-face-detection)).
- Google Cloud project with Vision enabled, if used. 1,000 free units a month ([Vision pricing](https://cloud.google.com/vision/pricing)).
- Vendor trials. Regula 30 days and 2,000 transactions ([Regula licensing](https://docs.regulaforensics.com/develop/doc-reader-sdk/overview/licensing/)). HyperVerge 1-month sandbox ([HyperVerge pricing](https://hyperverge.co/pricing/)). Microblink, Signzy and IDfy through sales (unverified).
- OFIQ model files from the ISO portal under their own licences ([OFIQ LICENSE.md](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/LICENSE.md)).
- Processor contracts that meet DPDP s. 8(2) and Rule 6.

Compute:

- OpenCV and scikit-image measures run on CPU. OFIQ builds on ONNX Runtime ([OFIQ BUILD.md](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/BUILD.md)). Regula's Face web service ships CPU and GPU versions, and the GPU version needs an NVIDIA GPU with CUDA compute capability 7.5 or higher ([Regula Face web service installation](https://docs.regulaforensics.com/develop/face-sdk/web-service/installation/)). I found no GPU requirement for OFIQ or Microblink on-prem in the pages I read, and no published throughput per core for any of them (unverified). Size from a load test.
- Cloud APIs need no compute beyond your backend.

Data:

- A labelled set of real dealer captures, good and bad, per ID type and per device class, uploaded through the real apps so it carries the real compression.

Roles:

- A backend developer for the scoring service and contract.
- An engineer with image-processing experience for self-hosted checks and normalisation.
- Android and web developers for the retake response, capture settings and optional hybrid signals.
- A QA engineer with low-end phones and a throttled network profile.
- An analyst to tune thresholds from logs.
- Legal and privacy review for DPDP, UIDAI and licence terms. Procurement for vendors.

Effort size. These are judgements, not measured figures.

| Route | Size | Reason |
|---|---|---|
| Cloud face API plus Rekognition image properties, own glare and in-frame checks | Small to medium | Face and sharpness scores ship ready. Work is the service, the contract, normalisation, glare and document-in-frame checks, and the opt-out and contracts. |
| Self-hosted OpenCV plus OFIQ | Medium | You write document checks and reason codes, integrate a C++ library, size CPU capacity, and tune everything. No vendor, no data leaves. |
| Self-hosted vendor (Regula or Microblink on-prem) | Small to medium for engineering, plus procurement | Checks and reason codes ship ready. Work is hosting the containers, licences and wiring. Cost unknown until quoted. |
| App changes for the retake response | Small | A new response state, a message table, a retry cap and fixed capture settings on two apps. |
| Hybrid client signals on Android | Small | ML Kit face detection and the CameraX focus result, both first-party. |

Costs from official pages only:

- OpenCV, scikit-image and OFIQ code: free.
- AWS Rekognition, ap-south-1: DetectFaces $0.00125 per image up to 1M a month, image properties $0.000938 per image up to 1M ([AWS price list, ap-south-1](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonRekognition/current/ap-south-1/index.json)). Higher tiers as in the table.
- Azure Face, Central India: $1.00 per 1,000 transactions up to 1M ([Azure Retail Prices API](https://prices.azure.com/api/retail/prices)).
- Google Vision: $1.50 per 1,000 units after the first 1,000 a month, per feature ([Vision pricing](https://cloud.google.com/vision/pricing)).
- Jumio Go: $37,500.00 per 12 months for 25,000 transactions ([AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-wyxkbmnbhansc)).
- Microblink, Regula, Signzy, IDfy, HyperVerge and Onfido: not public.
- Retries are billed as new calls on per-image cloud pricing. Each retake of a selfie scored with DetectFaces is another billed image ([Rekognition pricing](https://aws.amazon.com/rekognition/pricing/)).

## Risks and open questions

Risks:

- Slow uploads make every retake expensive in dealer time. Rural network speeds are unverified, so measure upload times in your own logs.
- Compression and resizing in the app or browser change the scores. Without fixed capture settings, the same scene scores differently by device and app version.
- Device variance. One global threshold will be wrong for some phones ([CVIA focus measure page](https://sites.google.com/view/cvia/focus-measure)).
- False rejects hurt sales. Start log-only and use a retry cap with a flagged override.
- Coverage gaps in cloud APIs. Rekognition, Azure and Google score faces well, but none checks glare or document framing. You still build those.
- Aadhaar exposure. A cloud API on an unmasked card receives the full number.
- Data leaving India with Google Vision, Microblink cloud, Onfido or Jumio.
- Azure Face Limited Access may be refused or slow.
- Provider outages block uploads unless the service fails open with a flag.
- Vendor lock-in. Keep your own reason codes and config so the scorer can be swapped.

Facts you need to supply:

- Backend language, hosting provider and region.
- Monthly capture volume and current retake and rejection rates, for cost and capacity.
- Current JPEG quality and image size each app sends, and whether either app resizes.
- Accepted ID types and whether both sides are captured.
- Whether images may leave India under your licence and contracts.

Open questions:

- Does the operator's current licence still bar transfer of user information outside India?
- Can an Azure Face resource be created in Central India for this account?
- What are upload times at your rural points of sale?
- What is the scoring time per image for OFIQ, OpenCV checks and each API on your hardware and region?
- Which regions do Signzy, IDfy and HyperVerge process images in, and do Signzy's "US APIs" support Indian IDs?
- Does DoT require keeping rejected capture attempts?
- How much does each app's compression move your sharpness scores? Test the same scene at several JPEG qualities.

## Sources

All accessed 2026-10-19.

Standards, specs and research

- NIST, FATE Quality. https://pages.nist.gov/frvt/html/frvt_quality.html
- WHATWG, HTML Standard, canvas element. https://html.spec.whatwg.org/multipage/canvas.html
- MDN, MediaDevices.getUserMedia(). https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia
- Pertuz, CVIA Focus Measure project page. https://sites.google.com/view/cvia/focus-measure

Open-source and first-party SDKs

- OpenCV, License. https://opencv.org/license/
- OpenCV, imgproc Image Filtering. https://docs.opencv.org/4.x/d4/d86/group__imgproc__filter.html
- scikit-image, skimage.measure (blur_effect). https://scikit-image.org/docs/stable/api/skimage.measure.html
- BSI OFIQ, README.md. https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/README.md
- BSI OFIQ, BUILD.md. https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/BUILD.md
- BSI OFIQ, LICENSE.md. https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/LICENSE.md
- BSI OFIQ, data/ofiq_config.jaxn. https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/data/ofiq_config.jaxn
- Google ML Kit, overview. https://developers.google.com/ml-kit/guides
- Google ML Kit, Terms and Privacy. https://developers.google.com/ml-kit/terms
- Google ML Kit, Face detection. https://developers.google.com/ml-kit/vision/face-detection
- Google ML Kit, Face class reference. https://developers.google.com/android/reference/com/google/mlkit/vision/face/Face
- Android Developers, CameraX configuration options. https://developer.android.com/media/camera/camerax/configuration
- Android Developers, ImageCapture.Builder. https://developer.android.com/reference/androidx/camera/core/ImageCapture.Builder
- Android Developers, FocusMeteringResult. https://developer.android.com/reference/androidx/camera/core/FocusMeteringResult
- Google AI Edge, MediaPipe Face Detector guide for Web. https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector/web_js

Cloud APIs

- AWS, Rekognition ImageQuality. https://docs.aws.amazon.com/rekognition/latest/APIReference/API_ImageQuality.html
- AWS, Rekognition DetectFaces. https://docs.aws.amazon.com/rekognition/latest/APIReference/API_DetectFaces.html
- AWS, Rekognition detecting labels and image properties. https://docs.aws.amazon.com/rekognition/latest/dg/labels-detect-labels-image.html
- AWS, Rekognition guidelines and quotas. https://docs.aws.amazon.com/rekognition/latest/dg/limits.html
- AWS, Rekognition pricing. https://aws.amazon.com/rekognition/pricing/
- AWS Price List API, AmazonRekognition ap-south-1. https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonRekognition/current/ap-south-1/index.json
- AWS Organizations, AI services opt-out policies. https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_ai-opt-out.html
- Microsoft, Face detection concepts. https://learn.microsoft.com/en-us/azure/ai-services/face/concept-face-detection
- Microsoft, Specify a face detection model. https://learn.microsoft.com/en-us/azure/ai-services/face/how-to/specify-detection-model
- Microsoft, Face Detect REST API v1.2. https://learn.microsoft.com/en-us/rest/api/face/face-detection-operations/detect?view=rest-face-v1.2
- Microsoft, Azure Retail Prices API (filter contains(productName,'Face'), Central India). https://prices.azure.com/api/retail/prices
- Microsoft, Face API pricing. https://azure.microsoft.com/en-us/pricing/details/cognitive-services/face-api/
- Google Cloud, Vision AnnotateImageResponse reference. https://cloud.google.com/vision/docs/reference/rest/v1/AnnotateImageResponse
- Google Cloud, Vision pricing. https://cloud.google.com/vision/pricing
- Google Cloud, Vision OCR and regional endpoints. https://cloud.google.com/vision/docs/ocr
- Google Cloud, Vision data usage. https://docs.cloud.google.com/vision/docs/data-usage

Commercial vendors

- Microblink, On-prem overview. https://docs.microblink.com/on-prem
- Microblink, On-prem image formats. https://docs.microblink.com/on-prem/image-formats
- Microblink, On-prem OpenAPI. https://docs.microblink.com/on-prem/api/latest/openapi.json
- Microblink, Verify image requirements. https://docs.microblink.com/verify/image-requirements
- Microblink, BlinkID Verify API regions. https://docs.microblink.com/verify/api
- Microblink, Verify API v3 cloud OpenAPI. https://docs.microblink.com/verify/openapi-v3-cloud.json
- Microblink, Plans. https://microblink.com/plans/
- Regula, Image Quality Assessment. https://docs.regulaforensics.com/develop/doc-reader-sdk/overview/image-quality/
- Regula, Service parameters of Document Reader Web Components. https://docs.regulaforensics.com/develop/doc-reader-sdk/web-components/configuration/parameters/
- Regula, Document Reader SDK Licensing Models. https://docs.regulaforensics.com/develop/doc-reader-sdk/overview/licensing/
- Regula, Face Image Quality Characteristics. https://docs.regulaforensics.com/develop/face-sdk/web-service/development/usage/face-detection/face-image-quality-check/
- Regula, Face SDK Web Service installation. https://docs.regulaforensics.com/develop/face-sdk/web-service/installation/
- Regula Help Center, Pricing: SDK Licensing Models. https://support.regulaforensics.com/hc/en-us/articles/360036943712-Pricing-SDK-Licensing-Models
- Signzy, ID Intelligence Advance. https://docs.signzy.com/us-apis/id-intelligence-advance
- Signzy, Image Quality Analysis. https://docs.signzy.com/us-apis/image-quality-analysis
- IDfy, Document Image Validation. https://www.idfy.com/document-validation/
- HyperVerge, ID Card OCR Quality Checks. https://hyperverge.co/in/integrations-marketplace/id-card-ocr-api/
- HyperVerge, Pricing. https://hyperverge.co/pricing/
- Entrust (Onfido), Document report. https://documentation.identity.entrust.com/guide/document-report/
- Entrust (Onfido), API reference v3.6. https://documentation.identity.entrust.com/api/latest/
- Jumio, Usability. https://documentation.jumio.ai/docs/references/capabilities/usability
- Jumio, Getting Started with the Web SDK. https://documentation.jumio.ai/docs/developer-resources/SDKs/web-sdk/introduction-web
- AWS Marketplace, Jumio Identity Verification listing. https://aws.amazon.com/marketplace/pp/prodview-wyxkbmnbhansc

Regulators and law (India)

- DoT, Instructions for Alternate Digital KYC Process, 03.04.2019. https://www.dot.gov.in/static/uploads/2026/02/94c5c5c3b29c65d994a1c0e6dce211fc.pdf
- DoT letter 800-09/2023-AS.II dated 31.08.2023, as enclosed in Railway Board letter dated 12.09.2024. https://telerb.railnet.gov.in/raw-attachment/wiki/UpLoads/CUGKYCInstruction.pdf
- DoT, Amendments in Unified Access Service Licence Agreement, 14.12.2005. https://www.dot.gov.in/static/uploads/2026/06/d6060d507355be6b98182baee7d8b890.pdf
- MeitY, Digital Personal Data Protection Act, 2023. https://www.meity.gov.in/static/uploads/2024/06/2bf1f0e9f04e6fb4f8fef35e82c42aa5.pdf
- MeitY, Digital Personal Data Protection Rules, 2025 (G.S.R. 846(E)). https://www.meity.gov.in/static/uploads/2025/11/53450e6e5dc0bfa85ebd78686cadad39.pdf
- MeitY, Commencement notification G.S.R. 843(E), 13.11.2025. https://www.meity.gov.in/static/uploads/2025/11/c56ceae6c383460ca69577428d36828b.pdf
- UIDAI, Circular on Dos and Don'ts for Requesting Entities, 31.10.2022. https://uidai.gov.in/images/DosandDonots_for_Requesting_Entities.pdf
- UIDAI, Aadhaar (Sharing of Information) Regulations, 2016. https://www.uidai.gov.in/images/The_Aadhaar_Sharing_of_information_regulation_2016.pdf
