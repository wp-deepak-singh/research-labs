## Purpose

We want to add a blur detection pipeline to our eKYC application. It runs whenever a dealer does an eKYC, on both ID documents and selfies, so that unclear photos are caught and retaken before the application is submitted.

## Scope

In scope:

- Selfies and ID document photos in the eKYC web and Android apps.
- Capture guidance in the apps: keep the face or the document inside the frame.
- Face detection in the apps.
- Backend checks on each uploaded photo for blur and glare.
- A retake prompt with a clear reason when a photo fails.

Out of scope:

- Blur or glare scoring on the live camera feed.

## Summary

When a dealer takes a selfie or a photo of an ID card, the app helps them line it up: the face or card has to sit inside the frame. After the photo is uploaded, our server checks whether it is sharp and free of glare. If it is not, the dealer is asked to take it again and told why. This cuts rejections caused by bad photos.

## Tools

| Job                                              | Where it runs        | Tool                                                                                                                                  |
| ------------------------------------------------ | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Face detection                                   | Web and Android apps | [MediaPipe Face Detector](https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector)                                        |
| Selfie guidance (face in frame, size, eyes open) | Web and Android apps | MediaPipe ([Face Landmarker](https://ai.google.dev/edge/mediapipe/solutions/vision/face_landmarker))                                  |
| Document guidance (card in frame)                | Web and Android apps | A frame overlay in the app. This is to be drawn as an overlay on the screen.                                                          |
| Blur detection                                   | Backend              | [OpenCV](https://opencv.org/license/), variance of the [Laplacian](https://docs.opencv.org/4.x/d5/db5/tutorial_laplace_operator.html) |
| Glare detection                                  | Backend              | OpenCV, bright-pixel mask with [inRange](https://docs.opencv.org/4.x/da/d97/tutorial_threshold_inRange.html)                          |

Reference code:

- [BlurDetection2](https://github.com/WillBrennan/BlurDetection2/blob/master/blur_detection/detection.py), a Python blur check (MIT).
- [blurry-check](https://github.com/i-DENVA/blurry-check/blob/main/src/blur-detector.ts), blur and glare rules for ID cards and profile photos (MIT).

## Sources

All accessed 2026-10-20.

- [MediaPipe Face Detector](https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector)
- [MediaPipe Face Landmarker](https://ai.google.dev/edge/mediapipe/solutions/vision/face_landmarker)
- [MediaPipe solutions guide](https://ai.google.dev/edge/mediapipe/solutions/guide)
- [OpenCV Laplacian tutorial](https://docs.opencv.org/4.x/d5/db5/tutorial_laplace_operator.html)
- [OpenCV inRange tutorial](https://docs.opencv.org/4.x/da/d97/tutorial_threshold_inRange.html)
- [BlurDetection2](https://github.com/WillBrennan/BlurDetection2/blob/master/blur_detection/detection.py)
- [blurry-check](https://github.com/i-DENVA/blurry-check/blob/main/src/blur-detector.ts)
