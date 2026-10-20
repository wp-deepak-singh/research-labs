# Build brief: MediaPipe tasks and free tools for sharp, clear selfie and ID capture

Earlier notes: [Blur detection at capture](2026-10-19-blur-detection-at-capture.md), [Blur detection build brief for web and Android](2026-10-19-blur-detection-web-android-build-brief.md) and [Backend-only image quality scoring](2026-10-19-backend-only-image-quality-scoring.md).

## Short answer

No MediaPipe task outputs a blur, sharpness or glare score, and none detects ID documents. MediaPipe covers geometry. Face Landmarker gives face presence, size, position, eyes open and head pose, and the selfie segmenter and hand landmarker add background and finger signals with limits.
Pixel quality comes from free OpenCV checks (variance of the Laplacian for blur, a bright-pixel mask for glare) that run in the browser as OpenCV.js, on Android, and on the backend. scikit-image, BRISQUE and OFIQ fit the backend.
This note recommends MediaPipe for face geometry plus OpenCV for blur and glare on both apps, then the backend re-check from the backend-only note.

## How it works

### What the sample repo contains

The [mediapipe-samples-web](https://github.com/google-ai-edge/mediapipe-samples-web) repo is Apache-2.0 and describes itself as "A collection of examples for the MediaPipe Task APIs that can run fully inside your browser" ([repo metadata via GitHub API](https://api.github.com/repos/google-ai-edge/mediapipe-samples-web)). The StackBlitz link opens the same repo ([StackBlitz](https://stackblitz.com/github/google-ai-edge/mediapipe-samples-web)).

Vision demos in `src/tasks/`, as listed in the [README](https://github.com/google-ai-edge/mediapipe-samples-web/blob/main/README.md) and the repo tree:

- Face Detector, Face Landmarker, Gesture Recognizer, Hand Landmarker, Holistic Landmarker, Pose Landmarker.
- Image Classifier, Image Embedder, Image Segmenter, Interactive Segmenter, Object Detector.
- Non-vision demos: audio classifier, decision maker, semantic retriever, universal embedder and text tasks. None of them is relevant here.

How the samples run, from the source:

- They depend on `@mediapipe/tasks-vision` 1.1.x ([package.json](https://github.com/google-ai-edge/mediapipe-samples-web/blob/main/package.json)).
- A build script copies the WASM files into the app and serves them from the app's own origin ([copy-wasm.js](https://github.com/google-ai-edge/mediapipe-samples-web/blob/main/copy-wasm.js); [base-worker.ts](https://github.com/google-ai-edge/mediapipe-samples-web/blob/main/src/workers/base-worker.ts)).
- Each task runs in a web worker ([src/workers](https://github.com/google-ai-edge/mediapipe-samples-web/tree/main/src/workers)). The Face Detector demo defaults to the GPU delegate ([face-detector.ts](https://github.com/google-ai-edge/mediapipe-samples-web/blob/main/src/tasks/face-detector.ts)).
- Models download from `storage.googleapis.com/mediapipe-models/` and are kept with the browser Cache API, so each model downloads once ([model-cache.ts](https://github.com/google-ai-edge/mediapipe-samples-web/blob/main/src/components/model-cache.ts)).

The MediaPipe catalogue lists face detection, face landmark detection, gesture recognition, hand landmark detection, holistic landmark detection, image classification, image embedding, image segmentation, interactive segmentation, object detection and pose landmark detection as vision solutions on Android, Web, Python and iOS. Of these, only gesture recognition, image classification and object detection can be customised with Model Maker ([MediaPipe Solutions guide](https://ai.google.dev/edge/mediapipe/solutions/guide)).

### Where each check runs

The capture flow is the one in the [earlier build brief](2026-10-19-blur-detection-web-android-build-brief.md): live preview check, check on the captured still, backend re-check, then a retake prompt with a reason. This note fills in which free tool does which check.

| Check | Selfie | ID document | Tool |
|---|---|---|---|
| Face present, one face, size, centred | Yes | Portrait on the card, optional | MediaPipe Face Detector or Face Landmarker |
| Eyes open, head pose | Yes | No | MediaPipe Face Landmarker |
| Second person in frame | Yes | No | Face Landmarker with `num_faces` above 1, or the selfie segmenter |
| Document in frame | No | Yes | No MediaPipe model. OpenCV contours, ML Kit document scanner on Android, or a custom model |
| Fingers over the document | No | Yes, weak | MediaPipe Hand Landmarker, with the limits below |
| Blur | Yes | Yes | OpenCV variance of the Laplacian, on the face or document crop |
| Glare and over-exposure | Yes | Yes | OpenCV bright-pixel mask |
| Backend re-check | Yes | Yes | OpenCV, scikit-image, BRISQUE, OFIQ, MediaPipe Python |

### MediaPipe tasks and what each contributes

No MediaPipe task outputs blur, sharpness or glare. The task pages list the outputs, and none of them is a pixel-quality score (task pages cited below). The `@mediapipe/tasks-vision` 1.1.1 type definitions contain no blur, sharpness or glare field. The only `qualityScores` field is the confidence of a segmentation mask, not image quality ([vision.d.ts, tasks-vision 1.1.1](https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@1.1.1/vision.d.ts)).

Face Detector

- Gives a bounding box, a confidence score and 6 key points per face: both eyes, nose tip, mouth and both eye tragions ([Face Detector overview](https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector)). From these you get face count, face size, centring and a rough roll angle from the eye line. The roll calculation is my derivation, not documented.
- Does not give eyes open, yaw or pitch, occlusion or quality.
- Models are BlazeFace short-range, for front-camera selfies, and full-range, for back-camera images. Both take 128x128 input ([Face Detector overview](https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector)).
- Model card: trained on "consented images of people using a mobile AR application captured with smartphone cameras". Out of scope are faces looking away or significantly inclined, people further than 2 metres, and "any form of surveillance or identity recognition". It is "not intended for human life-critical decisions". The card is Apache 2.0 ([BlazeFace short-range model card](https://storage.googleapis.com/mediapipe-assets/MediaPipe%20BlazeFace%20Model%20Card%20%28Short%20Range%29.pdf); [full-range card](https://storage.googleapis.com/mediapipe-assets/MediaPipe%20BlazeFace%20Model%20Card%20%28Full%20Range%29.pdf)).
- Benchmark on a Pixel 6: 2.94 ms on CPU, 7.41 ms on GPU for short-range ([Face Detector overview](https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector)).

Face Landmarker

- Gives 478 3D landmarks, 52 blendshape scores, and a facial transformation matrix that maps a canonical face model onto the detected face ([Face Landmarker overview](https://ai.google.dev/edge/mediapipe/solutions/vision/face_landmarker)). Blendshapes and the matrix are off by default and switched on with `output_face_blendshapes` and `output_facial_transformation_matrixes`. The web sample turns both on ([face-landmarker.ts](https://github.com/google-ai-edge/mediapipe-samples-web/blob/main/src/tasks/face-landmarker.ts)).
- Eyes open. The blendshape list includes `eyeBlinkLeft` and `eyeBlinkRight`, each scored from 0 to 1 ([Blendshape V2 model card](https://storage.googleapis.com/mediapipe-assets/Model%20Card%20Blendshape%20V2.pdf)). Left and right refer to the subject, and a mirrored image swaps them ([Face Landmarker overview](https://ai.google.dev/edge/mediapipe/solutions/vision/face_landmarker)).
- Head pose. Yaw, pitch and roll can be extracted from the transformation matrix. MediaPipe documents the matrix but not the angle extraction, so that step is your own code and its accuracy is unverified.
- Occlusion, partly. The Face Mesh card says the model is not suitable for faces looking away more than 80° or less than 50% visible ([Face Mesh V2 model card](https://storage.googleapis.com/mediapipe-assets/Model%20Card%20MediaPipe%20Face%20Mesh%20V2.pdf)). A low face-presence score can stand in for heavy occlusion. This use is my inference.
- Does not give blur, exposure or glare. The bundle contains BlazeFace short-range, Face Mesh V2 and Blendshape V2 ([Face Landmarker overview](https://ai.google.dev/edge/mediapipe/solutions/vision/face_landmarker)). The cards say the models are for AR entertainment, do not provide recognition or identification, and are Apache 2.0 ([Face Mesh V2 card](https://storage.googleapis.com/mediapipe-assets/Model%20Card%20MediaPipe%20Face%20Mesh%20V2.pdf); [Blendshape V2 card](https://storage.googleapis.com/mediapipe-assets/Model%20Card%20Blendshape%20V2.pdf)).
- I found no published Face Landmarker benchmark on the overview page (unverified).

Image Classifier

- Default models are EfficientNet-Lite0 and Lite2, trained on ImageNet's 1,000 classes ([Image Classifier overview](https://ai.google.dev/edge/mediapipe/solutions/vision/image_classifier)). The label list has no ID card, passport or blur class. Its nearest labels are things like "envelope", "menu" and "book jacket" ([labels.txt](https://storage.googleapis.com/mediapipe-tasks/image_classifier/labels.txt)).
- A custom "sharp, blurred, glare" classifier could be trained with Model Maker ([Solutions guide](https://ai.google.dev/edge/mediapipe/solutions/guide)). How accurate it would be is unverified. Model Maker is "no longer actively maintained" ([Model Maker](https://ai.google.dev/edge/mediapipe/solutions/model_maker)).
- I found no model card or licence for these model files on the overview page (unverified).

Object Detector

- Default models are EfficientDet-Lite0, EfficientDet-Lite2 and SSD MobileNetV2, trained on COCO with 80 labels ([Object Detector overview](https://ai.google.dev/edge/mediapipe/solutions/vision/object_detector)). The label map has "person", "cell phone" and "book" but no ID card or document ([labelmap.txt](https://storage.googleapis.com/mediapipe-tasks/object_detector/labelmap.txt)).
- A document detector would need a custom model. Model Maker can retrain the object detector on MobileNet-V2 or MobileNet-MultiHW-AVG with your own labelled images ([object detection customisation](https://ai.google.dev/edge/mediapipe/solutions/customization/object_detector)). Model Maker is deprecated ([Model Maker](https://ai.google.dev/edge/mediapipe/solutions/model_maker)), and a bounding box shows the card is present but not whether all four corners are in frame.
- No model card or licence is listed for these model files (unverified).

Image Segmenter

- The selfie segmenter splits person from background. The multiclass model labels background, hair, body skin, face skin, clothes and "others (accessories)". DeepLab-v3 covers general classes such as person, cat and dog ([Image Segmenter overview](https://ai.google.dev/edge/mediapipe/solutions/vision/image_segmenter)).
- It helps with background checks, with confining the blur score to the person or face region, and with spotting a second person. The accessories class may flag sunglasses or masks, which is my inference and unverified.
- There is no document class.
- The selfie segmentation card lists multiple people at different scales and people further than 4 metres as out of scope. It is Apache 2.0 ([Selfie segmentation model card](https://storage.googleapis.com/mediapipe-assets/Model%20Card%20MediaPipe%20Selfie%20Segmentation.pdf)). The multiclass card is also Apache 2.0 ([Multiclass segmentation model card](https://storage.googleapis.com/mediapipe-assets/Model%20Card%20Multiclass%20Segmentation.pdf)).
- Pixel 6 benchmarks: selfie segmenter 33.46 ms CPU, multiclass 217.76 ms CPU and 71.24 ms GPU ([Image Segmenter overview](https://ai.google.dev/edge/mediapipe/solutions/vision/image_segmenter)).

Interactive Segmenter

- Segments the object at a point or stroke you give it ([Interactive Segmenter overview](https://ai.google.dev/edge/mediapipe/solutions/vision/interactive_segmenter)). With the frame centre as the point, it might outline a card. That use is unverified.
- The MagicTouch card targets photo editing and object annotation and is Apache 2.0 ([MagicTouch model card](https://storage.googleapis.com/mediapipe-assets/Model%20Card%20MagicTouch.pdf)). The sample's model is about 30.5 MB, the largest here (size under Resources).

Image Embedder

- Produces a feature vector for comparing images by cosine similarity. Models are MobileNetV3 small and large, trained on ImageNet ([Image Embedder overview](https://ai.google.dev/edge/mediapipe/solutions/vision/image_embedder)).
- It can flag a resubmitted photo, or a selfie that matches the ID photo too closely, such as a photo of a photo. It says nothing about blur or glare. Its use for fraud checks is my suggestion and unverified.

Hand Landmarker and Gesture Recognizer

- Hand Landmarker returns 21 landmarks per hand and handedness ([Hand Landmarker overview](https://ai.google.dev/edge/mediapipe/solutions/vision/hand_landmarker)). Fingertip points inside the document box would mean fingers over the card.
- The model card lists "Predicting hand landmarks with gloves or occlusions. For example when the hand is holding objects" as out of scope. It is Apache 2.0 ([Hand tracking model card](https://storage.googleapis.com/mediapipe-assets/Model%20Card%20Hand%20Tracking%20%28Lite_Full%29%20with%20Fairness%20Oct%202021.pdf)). A hand holding an ID card is that case, so treat the signal as weak.
- Gesture Recognizer's default gestures are closed fist, open palm, pointing up, thumbs down, thumbs up, victory and "I love you" ([Gesture Recognizer overview](https://ai.google.dev/edge/mediapipe/solutions/vision/gesture_recognizer)). None of them is relevant to quality. I found no licence line in its model card text (unverified) ([gesture model card](https://storage.googleapis.com/mediapipe-assets/gesture_recognizer/model_card_hand_gesture_classification_with_faireness_2022.pdf)).

Holistic and Pose Landmarker add body pose. They add nothing for a head-and-shoulders selfie or a card image.

### Platform fit and data flow

- Web. Install `@mediapipe/tasks-vision` from npm or a CDN. The WASM fileset loads with `FilesetResolver.forVisionTasks`, and the guide's example points at jsDelivr ([Face Detector web guide](https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector/web_js)). Supported browsers are Chrome and Safari ([web setup](https://ai.google.dev/edge/mediapipe/solutions/setup_web)). Tasks accept a CPU or GPU delegate ([vision.d.ts](https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@1.1.1/vision.d.ts)). The npm package is Apache-2.0, version 1.1.1 ([npm registry](https://registry.npmjs.org/@mediapipe/tasks-vision/latest)).
- Android. The `com.google.mediapipe:tasks-vision` Gradle dependency, Android SDK 24 or higher. An emulator may not work for all tasks ([Android setup](https://ai.google.dev/edge/mediapipe/solutions/setup_android); [Face Detector Android guide](https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector/android)). GPU is enabled through `BaseOptions` ([Android setup](https://ai.google.dev/edge/mediapipe/solutions/setup_android)). The same `.tflite` and `.task` models work on web and Android.
- Backend. `pip install mediapipe`, Python 3.9 or later on Windows, Mac and Linux ([Python setup](https://ai.google.dev/edge/mediapipe/solutions/setup_python)). PyPI shows version 1.1.1 under Apache 2.0 ([PyPI mediapipe](https://pypi.org/pypi/mediapipe/json)). Use it to re-run the face checks on the uploaded image.
- On-device processing. MediaPipe's privacy notice says processing of input images "takes place on device" and that MediaPipe "does not send that input data to Google servers". It also says the Tasks APIs "send metrics about the performance and utilization of the APIs in your app to Google", and that "You are responsible for obtaining informed consent from your app users" about this ([MediaPipe README, Privacy Notice, last modified June 5, 2026](https://github.com/google-ai-edge/mediapipe/blob/master/README.md)). The MediaPipe repo is Apache-2.0 ([repo metadata](https://api.github.com/repos/google-ai-edge/mediapipe)).
- Asset hosting. Pages that load WASM from jsDelivr and models from `storage.googleapis.com` make the dealer's browser contact those hosts. The images stay on the device. Serve the WASM and models from your own origin, as the samples do for WASM ([copy-wasm.js](https://github.com/google-ai-edge/mediapipe-samples-web/blob/main/copy-wasm.js)). That recommendation is mine.

What this means for DPDP and Aadhaar. Scoring on the device sends no image to Google or to another vendor, so client checks add no new processor for the image. Two duties remain. The metrics flow needs a notice, under MediaPipe's own terms above. And the upload to your backend still carries the image, so the DPDP, UIDAI masking and retention points in the [backend-only note](2026-10-19-backend-only-image-quality-scoring.md) apply unchanged.

## Options

### MediaPipe tasks, ranked for this use

| Task | Use for capture quality | Not covered | Fit |
|---|---|---|---|
| Face Landmarker | Face presence, count, size, centring, eyes open via `eyeBlink*`, head pose via the matrix | Blur, exposure, glare, documents | Main selfie geometry check, web and Android |
| Face Detector | Face presence, count, size, centring, rough roll | Eyes, yaw and pitch, quality | Lighter option if you skip the eye and pose checks |
| Image Segmenter, selfie | Person mask, background, second person | Quality, documents | Optional for the selfie |
| Hand Landmarker | Fingertips over the card | Reliable when hands hold objects, per its card | Weak, optional |
| Object Detector, custom | Card presence and box after retraining | Corners, quality. Model Maker is deprecated | Only if contours fail |
| Image Classifier, custom | Possible sharp, blurred or glare classes after retraining | Accuracy unknown | Experimental |
| Interactive Segmenter | Card outline from a centre point, unverified | Size, at about 30.5 MB | Not recommended |
| Image Embedder | Duplicate or photo-of-photo checks | Quality | Separate fraud topic |
| Gesture, Pose, Holistic | Nothing for quality | | Skip |

### Free non-MediaPipe tools

| Tool | What it gives | Where it runs | Licence and source |
|---|---|---|---|
| OpenCV `Laplacian` plus `meanStdDev` | Variance of the Laplacian as a blur score, on the face or card crop | Both. OpenCV.js in the browser, the `org.opencv:opencv` AAR on Android, Python or C++ on the backend | Apache 2.0 ([OpenCV licence](https://opencv.org/license/); [Maven org.opencv:opencv 5.0.0 POM](https://repo1.maven.org/maven2/org/opencv/opencv/5.0.0/opencv-5.0.0.pom)). The OpenCV.js build list includes `Laplacian`, `meanStdDev`, `inRange`, `countNonZero`, `minMaxLoc`, `connectedComponentsWithStats` and `inpaint` ([opencv_js.config.py](https://github.com/opencv/opencv/blob/4.x/platforms/js/opencv_js.config.py)). Laplacian samples exist in Python, Java and JS ([Laplace tutorial](https://docs.opencv.org/4.x/d5/db5/tutorial_laplace_operator.html); [OpenCV.js gradients](https://docs.opencv.org/4.x/da/d85/tutorial_js_gradients.html)). |
| OpenCV HSV bright-pixel mask | Share of pixels with high V and low S, or near-white grey, as a glare score. Connected components give the spot size | Both | `inRange` on HSV ([inRange tutorial](https://docs.opencv.org/4.x/da/d97/tutorial_threshold_inRange.html); [OpenCV.js colour spaces](https://docs.opencv.org/4.x/db/d64/tutorial_js_colorspaces.html)). For 8-bit images, H runs 0 to 180, S and V 0 to 255 ([colour conversions](https://docs.opencv.org/4.x/de/d25/imgproc_color_conversions.html)). No built-in glare detector exists. |
| OpenCV contours | Document-in-frame by finding a four-corner contour with `approxPolyDP` | Both | [Contour features](https://docs.opencv.org/4.x/dd/d49/tutorial_py_contour_features.html). Robustness on dealer photos is unverified. |
| scikit-image `blur_effect` | No-reference blur score, 0 for none to 1 for maximal. Comparable only at the same `h_size` | Backend, Python | [skimage.measure](https://scikit-image.org/docs/stable/api/skimage.measure.html). Version 0.26.0 on PyPI. |
| OpenCV contrib BRISQUE | General no-reference quality score. Ships `brisque_model_live.yml` and `brisque_range_live.yml` | Backend, Python. The module is built with `WRAP python` and has no Java binding ([quality CMakeLists](https://github.com/opencv/opencv_contrib/blob/4.x/modules/quality/CMakeLists.txt); [quality README](https://github.com/opencv/opencv_contrib/blob/4.x/modules/quality/README.md)) | Apache 2.0. `cv2.quality.QualityBRISQUE_compute` is present in the `opencv-contrib-python-headless` 5.0.0 wheel, which I tested on the box. |
| BSI OFIQ | ISO/IEC 29794-5 face quality vector including Sharpness, EyesOpen, HeadPose, exposure and CompressionArtifacts | Backend. C++ with a ctypes Python wrapper built from source ([OFIQ python README](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/python/README.md)). Also has mobile builds ([OFIQ mobile/BUILD.md](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/mobile/BUILD.md)) | Code MIT, models under their own licences ([OFIQ LICENSE.md](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/LICENSE.md)). Version 1.2.1. |
| ML Kit face detection | Face box, Euler angles, eye-open probabilities. No blur ([ML Kit face detection](https://developers.google.com/ml-kit/vision/face-detection)) | Android only, API 23 or higher. Images at least 480x360, faces at least 100x100 px. App size about 800 KB unbundled or 6.9 MB bundled ([ML Kit face detection, Android](https://developers.google.com/ml-kit/vision/face-detection/android)) | Free, Google APIs terms, sends usage metrics ([ML Kit terms](https://developers.google.com/ml-kit/terms)) |
| ML Kit document scanner | Scan UI with edge detection, auto-capture and crop. No quality score | Android only. Needs Play services and 1.7 GB RAM ([ML Kit document scanner, Android](https://developers.google.com/ml-kit/vision/doc-scanner/android)) | Free, Google APIs terms |
| Browser Image Capture API | `takePhoto()` for a full-resolution still, and focus constraints where supported. Its `sharpness` is a camera setting, not a measurement ([W3C Image Capture, Editor's Draft](https://w3c.github.io/mediacapture-image/)) | Web. Uneven support, see the [earlier build brief](2026-10-19-blur-detection-web-android-build-brief.md) | Web standard draft |

### Recommended combination

This is the note's recommendation, not a sourced finding.

- Selfie, web and Android. Use MediaPipe Face Landmarker with `num_faces` set to 2, so a second face fails the check. Read face box size and centre, `eyeBlinkLeft` and `eyeBlinkRight`, and yaw, pitch and roll from the matrix. Then score the face crop with OpenCV variance of the Laplacian for blur and a bright-pixel share for glare and over-exposure. One MediaPipe model and one OpenCV build serve both apps. On Android, ML Kit face detection is a lighter alternative to Face Landmarker if the web and Android checks do not need to match.
- ID document, web and Android. Use OpenCV to find the card as a four-corner contour and check that it fills enough of the frame. On Android you can use the ML Kit document scanner in `SCANNER_MODE_BASE` instead. Score the card crop with the same OpenCV blur and glare checks. Hand Landmarker is optional and weak. Train a custom Object Detector only if contours fail on real captures.
- Backend. Re-run the same OpenCV measures on the uploaded image, plus OFIQ or MediaPipe Python for the selfie. This is the service from the [backend-only note](2026-10-19-backend-only-image-quality-scoring.md). The client checks cut wasted uploads, and the backend check still decides.

How it relates to the backend-only note. That note's service, API contract, thresholds, logging and retention stay as written. This combination adds the client layer that the backend-only design lacks, which means live guidance and fewer bad uploads. The added cost is model downloads and per-device tuning. Keep one threshold config, with the client values set equal to or slightly stricter than the backend values, so that a client pass is rarely a backend fail. That rule is my recommendation.

## Development requirements

### Web app

- Add `@mediapipe/tasks-vision` and serve its WASM and models from your own origin ([copy-wasm.js](https://github.com/google-ai-edge/mediapipe-samples-web/blob/main/copy-wasm.js)). Cache models with the Cache API, as the sample does ([model-cache.ts](https://github.com/google-ai-edge/mediapipe-samples-web/blob/main/src/components/model-cache.ts)).
- Run MediaPipe and OpenCV.js in a web worker, as the samples do for MediaPipe ([src/workers](https://github.com/google-ai-edge/mediapipe-samples-web/tree/main/src/workers)). Use the GPU delegate with a CPU fallback.
- Use a reduced OpenCV.js build if the full 11 MB file is too heavy. The build list in `opencv_js.config.py` controls which functions are included ([opencv_js.config.py](https://github.com/opencv/opencv/blob/4.x/platforms/js/opencv_js.config.py)). The size of a reduced build is unverified.
- Score preview frames at a fixed crop size, then re-score the captured still. Export the still at a fixed, high JPEG quality ([HTML Standard, canvas](https://html.spec.whatwg.org/multipage/canvas.html)).
- Show retake reasons, cap retries, and allow a flagged submission.

### Android app

- Add `com.google.mediapipe:tasks-vision`, with min SDK 24 ([Android setup](https://ai.google.dev/edge/mediapipe/solutions/setup_android)), and the OpenCV AAR ([Maven POM](https://repo1.maven.org/maven2/org/opencv/opencv/5.0.0/opencv-5.0.0.pom)). Or use ML Kit, with min SDK 23.
- Feed CameraX analysis frames to the face and quality checks. Keep only the latest frame. Re-score the still before upload ([CameraX configuration](https://developer.android.com/media/camera/camerax/configuration)).
- Set JPEG quality explicitly with `setJpegQuality` ([CameraX ImageCapture.Builder](https://developer.android.com/reference/androidx/camera/core/ImageCapture.Builder)).
- Ship models in the APK, or download them once. Use the same model files as the web app.

### Backend

- The scoring service from the [backend-only note](2026-10-19-backend-only-image-quality-scoring.md), using OpenCV, scikit-image or BRISQUE in Python, OFIQ for selfies, and MediaPipe Python for face geometry if you want parity with the client.
- Serve one versioned threshold config to both apps and the backend.

### Integration points

- Client to backend. Send the image, the client scores, the reason codes, model and threshold versions, and capture metadata. Log the client scores next to the backend scores to find drift between them.
- Consent and notice text covering the MediaPipe metrics flow ([MediaPipe Privacy Notice](https://github.com/google-ai-edge/mediapipe/blob/master/README.md)), and the ML Kit metrics flow if you use ML Kit ([ML Kit terms](https://developers.google.com/ml-kit/terms)).

## Resources needed

Everything listed is free. No accounts or keys are needed for MediaPipe, OpenCV, scikit-image or OFIQ code. OFIQ model files come from the ISO portal under their own licences ([OFIQ LICENSE.md](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/LICENSE.md)).

Download weight. Model sizes are the `content-length` that Google's model host and the OpenCV docs host return. Compressed transfer sizes are unverified.

| Asset | Size |
|---|---|
| `vision_wasm_internal.wasm`, tasks-vision 1.1.1 | 13,519,922 bytes, about 13.5 MB. A no-SIMD build is about 12.6 MB ([npm tarball 1.1.1](https://registry.npmjs.org/@mediapipe/tasks-vision/-/tasks-vision-1.1.1.tgz)) |
| BlazeFace short-range | 229,746 bytes, about 0.23 MB ([model file](https://storage.googleapis.com/mediapipe-models/face_detector/blaze_face_short_range/float16/1/blaze_face_short_range.tflite)) |
| BlazeFace full-range | 1,083,786 bytes, about 1.1 MB ([model file](https://storage.googleapis.com/mediapipe-models/face_detector/blaze_face_full_range/float16/1/blaze_face_full_range.tflite)) |
| Face Landmarker bundle | 3,758,596 bytes, about 3.8 MB ([model file](https://storage.googleapis.com/mediapipe-models/face_landmarker/face_landmarker/float16/1/face_landmarker.task)) |
| Selfie segmenter | 249,537 bytes, about 0.25 MB ([model file](https://storage.googleapis.com/mediapipe-models/image_segmenter/selfie_segmenter/float16/1/selfie_segmenter.tflite)) |
| Multiclass selfie segmenter | 16,371,837 bytes, about 16.4 MB ([model file](https://storage.googleapis.com/mediapipe-models/image_segmenter/selfie_multiclass_256x256/float32/latest/selfie_multiclass_256x256.tflite)) |
| Hand Landmarker bundle | 7,819,105 bytes, about 7.8 MB ([model file](https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/1/hand_landmarker.task)) |
| EfficientDet-Lite0 int8 | 4,602,795 bytes, about 4.6 MB ([model file](https://storage.googleapis.com/mediapipe-models/object_detector/efficientdet_lite0/int8/1/efficientdet_lite0.tflite)) |
| EfficientNet-Lite0 float32 | 18,582,189 bytes, about 18.6 MB ([model file](https://storage.googleapis.com/mediapipe-models/image_classifier/efficientnet_lite0/float32/1/efficientnet_lite0.tflite)) |
| Interactive segmenter | 30,525,312 bytes, about 30.5 MB ([model file](https://storage.googleapis.com/mediapipe-models/interactive_segmenter_v2/magic_touch/int8/1/interactive_segmentation.task)) |
| MobileNetV3 small embedder | 4,117,670 bytes, about 4.1 MB ([model file](https://storage.googleapis.com/mediapipe-models/image_embedder/mobilenet_v3_small/float32/1/mobilenet_v3_small.tflite)) |
| `opencv.js` 4.13.0 | 10,964,323 bytes, about 11.0 MB ([docs.opencv.org/4.13.0/opencv.js](https://docs.opencv.org/4.13.0/opencv.js)) |
| `opencv.js` 5.0 | 16,210,809 bytes, about 16.2 MB ([docs.opencv.org/5.0/opencv.js](https://docs.opencv.org/5.0/opencv.js)) |
| ML Kit face detection, Android | About 800 KB unbundled, 6.9 MB bundled ([ML Kit face detection, Android](https://developers.google.com/ml-kit/vision/face-detection/android)) |

For the recommended web path, the first visit downloads about 13.5 MB of MediaPipe WASM, 3.8 MB of Face Landmarker and 11 MB of OpenCV.js. That totals about 28 MB before compression, by my arithmetic. Later visits read from cache. The size of the MediaPipe Android library is unverified.

Compute. Client checks run on the dealer's phone. Backend checks run on CPU. GPU needs for OFIQ are unverified, as in the backend-only note.

Devices and data:

- Low-end dealer Android phones and their browsers, including Chrome, Samsung Internet and WebView, plus iPhones if used. The web setup page names Chrome and Safari only ([web setup](https://ai.google.dev/edge/mediapipe/solutions/setup_web)).
- A labelled set of real selfies and ID images, sharp, blurred and with glare, per device class, to set thresholds.
- For a custom document detector, labelled card images for Model Maker ([object detection customisation](https://ai.google.dev/edge/mediapipe/solutions/customization/object_detector)).

Roles needed are a web developer, an Android developer, a backend developer, an engineer with image-processing experience to tune blur, glare and contour checks, QA with a device lab, and privacy review for the metrics notice.

Effort size. These are judgements, not measured figures.

| Piece | Size | Reason |
|---|---|---|
| Recommended combination, client side | Medium | Two platforms, one MediaPipe model and OpenCV on each, a document contour check, retake UI, and per-device tuning. No vendor work. |
| Backend re-check | Small to medium | As in the backend-only note. |
| Custom MediaPipe document detector | Medium, extra | Labelling, training with deprecated Model Maker, and export to both apps. Only if contours fail. |
| Adding Face Detector only, as in the samples | Small | It covers presence and size only. Eyes, pose, blur and glare are still missing. |

## Risks and open questions

Risks:

- Expecting a quality score from MediaPipe. None exists. Blur and glare must come from OpenCV or another pixel measure.
- Model-card scope. The face models' cards exclude "surveillance or identity recognition" and "human life-critical decisions" ([BlazeFace card](https://storage.googleapis.com/mediapipe-assets/MediaPipe%20BlazeFace%20Model%20Card%20%28Short%20Range%29.pdf)). Using them as a capture aid is not recognition. Have legal confirm that eKYC capture fits the stated intended uses.
- Hands holding objects are out of scope for the hand model. Finger-over-card detection may be unreliable ([Hand tracking card](https://storage.googleapis.com/mediapipe-assets/Model%20Card%20Hand%20Tracking%20%28Lite_Full%29%20with%20Fairness%20Oct%202021.pdf)).
- Model Maker is no longer maintained ([Model Maker](https://ai.google.dev/edge/mediapipe/solutions/model_maker)). A custom document detector relies on a deprecated tool.
- Download weight on slow networks. About 28 MB on the first visit for the recommended web path.
- Speed on low-end phones. Google's benchmarks are for a Pixel 6. Dealer devices are untested (unverified).
- Browser coverage. Web setup names Chrome and Safari. Samsung Internet, WebView and Firefox behaviour with the WASM and GPU delegate is unverified.
- Metrics to Google. Both MediaPipe and ML Kit send usage metrics and require a user notice.
- Threshold drift between client and backend if they use different crop sizes or builds.

Facts you need to supply:

- Dealer device and browser mix, and network speeds at points of sale.
- Whether the web app runs in a WebView or an iframe.
- The Android minSdk. MediaPipe needs 24 and ML Kit 23.
- Accepted ID types and whether both sides are captured.

Open questions:

- How fast are Face Landmarker and OpenCV.js on your lowest-end phones, and what frame rate is needed for live guidance?
- Does an OpenCV contour check find cards reliably on real dealer captures, or is a custom detector needed?
- What licences apply to the EfficientNet, EfficientDet, MobileNet and DeepLab model files, which have no model card on the task pages?
- What exactly do MediaPipe's web metrics contain, and can they be turned off?
- Is a reduced OpenCV.js build small enough, and how large is it?

## Sources

All accessed 2026-10-20.

MediaPipe repos and packages

- google-ai-edge/mediapipe-samples-web, README.md. https://github.com/google-ai-edge/mediapipe-samples-web/blob/main/README.md
- google-ai-edge/mediapipe-samples-web, repo metadata. https://api.github.com/repos/google-ai-edge/mediapipe-samples-web
- google-ai-edge/mediapipe-samples-web, package.json. https://github.com/google-ai-edge/mediapipe-samples-web/blob/main/package.json
- google-ai-edge/mediapipe-samples-web, copy-wasm.js. https://github.com/google-ai-edge/mediapipe-samples-web/blob/main/copy-wasm.js
- google-ai-edge/mediapipe-samples-web, src/workers/base-worker.ts. https://github.com/google-ai-edge/mediapipe-samples-web/blob/main/src/workers/base-worker.ts
- google-ai-edge/mediapipe-samples-web, src/workers. https://github.com/google-ai-edge/mediapipe-samples-web/tree/main/src/workers
- google-ai-edge/mediapipe-samples-web, src/tasks/face-detector.ts. https://github.com/google-ai-edge/mediapipe-samples-web/blob/main/src/tasks/face-detector.ts
- google-ai-edge/mediapipe-samples-web, src/tasks/face-landmarker.ts. https://github.com/google-ai-edge/mediapipe-samples-web/blob/main/src/tasks/face-landmarker.ts
- google-ai-edge/mediapipe-samples-web, src/components/model-cache.ts. https://github.com/google-ai-edge/mediapipe-samples-web/blob/main/src/components/model-cache.ts
- StackBlitz, mediapipe-samples-web. https://stackblitz.com/github/google-ai-edge/mediapipe-samples-web
- google-ai-edge/mediapipe, README.md, Privacy Notice. https://github.com/google-ai-edge/mediapipe/blob/master/README.md
- google-ai-edge/mediapipe, repo metadata. https://api.github.com/repos/google-ai-edge/mediapipe
- npm, @mediapipe/tasks-vision latest. https://registry.npmjs.org/@mediapipe/tasks-vision/latest
- npm, @mediapipe/tasks-vision 1.1.1 tarball. https://registry.npmjs.org/@mediapipe/tasks-vision/-/tasks-vision-1.1.1.tgz
- jsDelivr, @mediapipe/tasks-vision 1.1.1 vision.d.ts. https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@1.1.1/vision.d.ts
- PyPI, mediapipe. https://pypi.org/pypi/mediapipe/json

MediaPipe documentation

- MediaPipe Solutions guide. https://ai.google.dev/edge/mediapipe/solutions/guide
- MediaPipe Model Maker. https://ai.google.dev/edge/mediapipe/solutions/model_maker
- Object detection model customisation. https://ai.google.dev/edge/mediapipe/solutions/customization/object_detector
- Setup guide for web. https://ai.google.dev/edge/mediapipe/solutions/setup_web
- Setup guide for Android. https://ai.google.dev/edge/mediapipe/solutions/setup_android
- Setup guide for Python. https://ai.google.dev/edge/mediapipe/solutions/setup_python
- Face Detector overview. https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector
- Face Detector web guide. https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector/web_js
- Face Detector Android guide. https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector/android
- Face Landmarker overview. https://ai.google.dev/edge/mediapipe/solutions/vision/face_landmarker
- Image Classifier overview. https://ai.google.dev/edge/mediapipe/solutions/vision/image_classifier
- Object Detector overview. https://ai.google.dev/edge/mediapipe/solutions/vision/object_detector
- Image Segmenter overview. https://ai.google.dev/edge/mediapipe/solutions/vision/image_segmenter
- Interactive Segmenter overview. https://ai.google.dev/edge/mediapipe/solutions/vision/interactive_segmenter
- Image Embedder overview. https://ai.google.dev/edge/mediapipe/solutions/vision/image_embedder
- Hand Landmarker overview. https://ai.google.dev/edge/mediapipe/solutions/vision/hand_landmarker
- Gesture Recognizer overview. https://ai.google.dev/edge/mediapipe/solutions/vision/gesture_recognizer
- Image Classifier labels. https://storage.googleapis.com/mediapipe-tasks/image_classifier/labels.txt
- Object Detector label map. https://storage.googleapis.com/mediapipe-tasks/object_detector/labelmap.txt

MediaPipe model cards

- BlazeFace short-range. https://storage.googleapis.com/mediapipe-assets/MediaPipe%20BlazeFace%20Model%20Card%20%28Short%20Range%29.pdf
- BlazeFace full-range. https://storage.googleapis.com/mediapipe-assets/MediaPipe%20BlazeFace%20Model%20Card%20%28Full%20Range%29.pdf
- Face Mesh V2. https://storage.googleapis.com/mediapipe-assets/Model%20Card%20MediaPipe%20Face%20Mesh%20V2.pdf
- Blendshape V2. https://storage.googleapis.com/mediapipe-assets/Model%20Card%20Blendshape%20V2.pdf
- Selfie Segmentation. https://storage.googleapis.com/mediapipe-assets/Model%20Card%20MediaPipe%20Selfie%20Segmentation.pdf
- Multiclass Segmentation. https://storage.googleapis.com/mediapipe-assets/Model%20Card%20Multiclass%20Segmentation.pdf
- MagicTouch. https://storage.googleapis.com/mediapipe-assets/Model%20Card%20MagicTouch.pdf
- Hand Tracking (Lite/Full). https://storage.googleapis.com/mediapipe-assets/Model%20Card%20Hand%20Tracking%20%28Lite_Full%29%20with%20Fairness%20Oct%202021.pdf
- Hand gesture classification. https://storage.googleapis.com/mediapipe-assets/gesture_recognizer/model_card_hand_gesture_classification_with_faireness_2022.pdf

MediaPipe model files (sizes)

- https://storage.googleapis.com/mediapipe-models/face_detector/blaze_face_short_range/float16/1/blaze_face_short_range.tflite
- https://storage.googleapis.com/mediapipe-models/face_detector/blaze_face_full_range/float16/1/blaze_face_full_range.tflite
- https://storage.googleapis.com/mediapipe-models/face_landmarker/face_landmarker/float16/1/face_landmarker.task
- https://storage.googleapis.com/mediapipe-models/image_segmenter/selfie_segmenter/float16/1/selfie_segmenter.tflite
- https://storage.googleapis.com/mediapipe-models/image_segmenter/selfie_multiclass_256x256/float32/latest/selfie_multiclass_256x256.tflite
- https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/1/hand_landmarker.task
- https://storage.googleapis.com/mediapipe-models/object_detector/efficientdet_lite0/int8/1/efficientdet_lite0.tflite
- https://storage.googleapis.com/mediapipe-models/image_classifier/efficientnet_lite0/float32/1/efficientnet_lite0.tflite
- https://storage.googleapis.com/mediapipe-models/interactive_segmenter_v2/magic_touch/int8/1/interactive_segmentation.task
- https://storage.googleapis.com/mediapipe-models/image_embedder/mobilenet_v3_small/float32/1/mobilenet_v3_small.tflite

OpenCV and other free tools

- OpenCV, License. https://opencv.org/license/
- Maven Central, org.opencv:opencv 5.0.0 POM. https://repo1.maven.org/maven2/org/opencv/opencv/5.0.0/opencv-5.0.0.pom
- OpenCV, opencv_js.config.py. https://github.com/opencv/opencv/blob/4.x/platforms/js/opencv_js.config.py
- OpenCV, Laplace Operator tutorial. https://docs.opencv.org/4.x/d5/db5/tutorial_laplace_operator.html
- OpenCV.js, Image Gradients tutorial. https://docs.opencv.org/4.x/da/d85/tutorial_js_gradients.html
- OpenCV, Thresholding Operations using inRange. https://docs.opencv.org/4.x/da/d97/tutorial_threshold_inRange.html
- OpenCV.js, Changing Colorspaces. https://docs.opencv.org/4.x/db/d64/tutorial_js_colorspaces.html
- OpenCV, Color conversions. https://docs.opencv.org/4.x/de/d25/imgproc_color_conversions.html
- OpenCV, Contour Features. https://docs.opencv.org/4.x/dd/d49/tutorial_py_contour_features.html
- OpenCV, opencv.js 4.13.0. https://docs.opencv.org/4.13.0/opencv.js
- OpenCV, opencv.js 5.0. https://docs.opencv.org/5.0/opencv.js
- OpenCV contrib, quality module README. https://github.com/opencv/opencv_contrib/blob/4.x/modules/quality/README.md
- OpenCV contrib, quality module CMakeLists.txt. https://github.com/opencv/opencv_contrib/blob/4.x/modules/quality/CMakeLists.txt
- scikit-image, skimage.measure. https://scikit-image.org/docs/stable/api/skimage.measure.html
- BSI OFIQ, python/README.md. https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/python/README.md
- BSI OFIQ, mobile/BUILD.md. https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/mobile/BUILD.md
- BSI OFIQ, LICENSE.md. https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/LICENSE.md
- Google ML Kit, Face detection. https://developers.google.com/ml-kit/vision/face-detection
- Google ML Kit, Detect faces on Android. https://developers.google.com/ml-kit/vision/face-detection/android
- Google ML Kit, Document scanner on Android. https://developers.google.com/ml-kit/vision/doc-scanner/android
- Google ML Kit, Terms and Privacy. https://developers.google.com/ml-kit/terms
- Android Developers, CameraX configuration options. https://developer.android.com/media/camera/camerax/configuration
- Android Developers, ImageCapture.Builder. https://developer.android.com/reference/androidx/camera/core/ImageCapture.Builder
- W3C, MediaStream Image Capture, Editor's Draft. https://w3c.github.io/mediacapture-image/
- WHATWG, HTML Standard, canvas element. https://html.spec.whatwg.org/multipage/canvas.html
