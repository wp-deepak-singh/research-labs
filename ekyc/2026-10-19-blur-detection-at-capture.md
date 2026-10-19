# Detecting blurry and unusable selfies and ID images at capture time

## Short answer

No standard or regulator document I found sets a numeric blur threshold, and published focus measures give relative, device-dependent scores. On each preview frame, compute a cheap sharpness score such as the variance of the Laplacian on the face box or document quad, combine it with camera focus state, face size, pose, eye and document-in-frame checks, then re-score the captured still and prompt a retake if any check fails. Calibrate every threshold on your own devices and log the raw scores, because neither ML Kit face detection nor CameraX returns a blur score.

## Findings

### 1. What blur is

A camera image can be modelled as the true scene passed through a blur filter, plus noise. OpenCV writes this in the frequency domain as `S = H·U + N`, where `H` is the frequency response of the point spread function (PSF), the pattern a single point of light turns into ([OpenCV, Out-of-focus Deblur Filter](https://docs.opencv.org/4.x/de/d3c/tutorial_out_of_focus_deblur_filter.html)).

Two kinds of blur matter at capture time.

- Defocus blur. The lens is focused at the wrong distance. A circular PSF is a good approximation of out-of-focus distortion, specified by one parameter, its radius ([OpenCV, Out-of-focus Deblur Filter](https://docs.opencv.org/4.x/de/d3c/tutorial_out_of_focus_deblur_filter.html)). Defocus acts as a low-pass filter, so it removes fine detail first ([Pertuz, CVIA focus measure page](https://sites.google.com/view/cvia/focus-measure)).
- Motion blur. The subject or the phone moves during the exposure. The PSF of linear motion blur is a line segment with a length and an angle ([OpenCV, Motion Deblur Filter](https://docs.opencv.org/4.x/d1/dfd/tutorial_motion_deblur_filter.html)). Motion blur therefore smears detail in one direction more than others.

Liu, Li and Jia built a classifier that detects partially blurred images and labels the blur type using colour, gradient and spectrum features, without estimating the blur kernel ([Liu et al., CVPR 2008](https://doi.org/10.1109/CVPR.2008.4587465), abstract only).

### 2. How blur is measured

A focus measure (also called a sharpness measure) is a number that goes up when an image has more fine detail. Pertuz, Puig and Garcia reviewed and compared many of them ([Pertuz et al., Pattern Recognition 46(5), 2013](https://doi.org/10.1016/j.patcog.2012.11.011)). Their project page groups the operators into six families: gradient-based, Laplacian-based, wavelet-based, statistics-based, DCT-based and miscellaneous ([CVIA focus measure page](https://sites.google.com/view/cvia/focus-measure)). The authors publish MATLAB code for all of them as `fmeasure` ([MATLAB File Exchange 27314](https://www.mathworks.com/matlabcentral/fileexchange/27314-focus-measure)).

Variance of the Laplacian. The Laplacian is the sum of the second derivatives in x and y. It responds strongly at edges. OpenCV's `cv::Laplacian` computes it with Sobel derivatives when `ksize > 1`, and with the 3x3 kernel `[0 1 0; 1 -4 1; 0 1 0]` when `ksize == 1` ([OpenCV imgproc filter docs](https://docs.opencv.org/4.x/d4/d86/group__imgproc__filter.html)). The focus measure is the variance of that output. In Pertuz's `fmeasure`, operator `LAPV` filters the image with a Laplacian kernel and returns the squared standard deviation, and attributes the method to Pech-Pacheco et al. 2000 ([fmeasure source, File Exchange 27314](https://www.mathworks.com/matlabcentral/fileexchange/27314-focus-measure)). That paper studied autofocus for diatom microscopy and proposed a modified Tenengrad and a modified Laplacian focus measure ([Pech-Pacheco et al., ICPR 2000](https://doi.org/10.1109/ICPR.2000.903548), abstract only). In OpenCV you get the variance by squaring the standard deviation returned by `cv::meanStdDev`, which also accepts a mask ([OpenCV core docs](https://docs.opencv.org/4.x/d2/de8/group__core__array.html)).

Tenengrad. This measure uses first derivatives. In `fmeasure`, `TENG` applies Sobel filters in x and y and returns the mean of `Gx² + Gy²`. `TENV` returns the variance of the same quantity. The code attributes `TENG` to Krotkov 1986 and `TENV` to Pech-Pacheco 2000 ([fmeasure source](https://www.mathworks.com/matlabcentral/fileexchange/27314-focus-measure)).

Frequency-domain measures. Blur removes high frequencies, so the share of energy in high-frequency coefficients works as a sharpness measure. Pertuz's review includes DCT-based and wavelet-based operators, and notes that DCT-based ones have mostly been used for autofocus because image formats already compute 8x8 DCT blocks ([CVIA focus measure page](https://sites.google.com/view/cvia/focus-measure)). I did not find a first-party library function that returns an FFT blur score. An FFT high-pass energy ratio is a reasonable construction, but I have no primary source for a specific recipe (unverified).

A no-reference perceptual metric. scikit-image ships `skimage.measure.blur_effect`, based on Crete et al. 2007. It returns 0 for no blur and 1 for maximal blur. By default it returns the maximum of per-axis blur values, and `reduce_func=None` returns the per-axis list. The docs say `h_size` must stay the same to compare results between images ([scikit-image `skimage.measure` docs](https://scikit-image.org/docs/stable/api/skimage.measure.html); [Crete et al., SPIE 6492, 2007](https://doi.org/10.1117/12.702790)).

Learned measures. OFIQ's face `Sharpness` measure computes the mean and standard deviation of Laplacian responses at kernel sizes 1, 3, 5, 7 and 9, plus mean-difference and Sobel features, over the face region. It feeds these features to an OpenCV random-forest model, `face_sharpness_rtree.xml.gz` ([OFIQ `Sharpness.cpp`](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/OFIQlib/modules/measures/src/Sharpness.cpp); [OFIQ `ofiq_config.jaxn`](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/data/ofiq_config.jaxn)).

### 3. How thresholds are chosen, and why there is no universal number

- Pertuz's experiments found that the relative performance of focus measures depends on the capturing device, the imaging conditions and the scene. The authors say an absolute ranking is not feasible ([CVIA focus measure page](https://sites.google.com/view/cvia/focus-measure)).
- Apple documents its face capture quality score for comparing captures of the same face against each other. It is not documented as an absolute pass mark ([Apple, `VNFaceObservation.faceCaptureQuality`](https://developer.apple.com/documentation/vision/vnfaceobservation/facecapturequality-bjg5)).
- NIST evaluates face quality algorithms by how well they predict recognition failure. It sets a threshold so that false non-match rate (FNMR) is 1%, then measures how much FNMR drops after discarding the lowest-quality 1% or 5% of images ([NIST FATE Quality](https://pages.nist.gov/frvt/html/frvt_quality.html)). The same method can be used to tune your own threshold against your own face-match outcomes.
- None of the regulator documents in section 7 give a numeric sharpness threshold.

I found no primary source for a variance-of-Laplacian cutoff for selfies or ID cards. Any number you see quoted, such as "100", comes from blogs and has no basis here (unverified).

### 4. Known failure modes of focus measures

- Low texture. Gradient-based operators work well only when the scene has a lot of texture, and this is a common restriction for most focus measures ([CVIA focus measure page](https://sites.google.com/view/cvia/focus-measure)). A plain wall, a smooth cheek or a blank card area gives a low score even when sharp.
- Noise. Laplacian-based operators are the most sensitive to image noise. Statistics-based operators are the most robust ([CVIA focus measure page](https://sites.google.com/view/cvia/focus-measure)). Low light raises sensor noise, which can push a Laplacian score up on a blurry image (unverified for phone pipelines specifically).
- Window size and resolution. Gradient and Laplacian operators are the most sensitive to a smaller evaluation window. Wavelet-based ones are the most robust ([CVIA focus measure page](https://sites.google.com/view/cvia/focus-measure)). A score computed on a 640x480 preview frame is not comparable with one computed on a 12 MP still unless both are resized to the same ROI size (unverified as a fixed rule, but implied by the window-size result and by scikit-image's same-`h_size` requirement).
- Contrast and lighting. In OpenCV's Laplacian tutorial, high-contrast regions show stronger responses ([OpenCV Laplace Operator tutorial](https://docs.opencv.org/4.x/d5/db5/tutorial_laplace_operator.html)). Pertuz tested contrast and saturation as separate factors ([Pertuz et al. 2013](https://doi.org/10.1016/j.patcog.2012.11.011)).
- Glare. A specular highlight on a laminated card creates hard bright edges that can raise a sharpness score while hiding text (unverified). Glare needs its own check.
- Background. A sharp background behind a blurred face or card can pass a whole-frame check. Score only the face box or the document quad. OFIQ restricts its sharpness features to the face region ([OFIQ `Sharpness.cpp`](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/OFIQlib/modules/measures/src/Sharpness.cpp)).

### 5. Other capture-quality checks that matter for eKYC

Face image.

- Face size. ML Kit says each face should be at least 100x100 pixels for detection, and 200x200 for contour detection. Input images should be at least 480x360 ([ML Kit face detection, Android](https://developers.google.com/ml-kit/vision/face-detection/android)).
- Pose. ML Kit returns head Euler angles X, Y and Z in degrees ([ML Kit `Face` reference](https://developers.google.com/android/reference/com/google/mlkit/vision/face/Face)). The Indian e-Governance face standard requires head rotation under ±5 degrees in roll, pitch and yaw ([eGov Face Image Data Standard v1.0, 5.2.4](https://egovstandards.gov.in/sites/default/files/Face_Image_Data_Standard_Ver1.0.pdf)).
- Eyes open. ML Kit returns left and right eye-open probabilities from 0.0 to 1.0 when classification is enabled ([ML Kit `Face` reference](https://developers.google.com/android/reference/com/google/mlkit/vision/face/Face)).
- Inter-eye distance. The same Indian standard asks for at least 120 pixels between the eyes for a head width of 240 pixels ([eGov Face Image Data Standard v1.0, 5.2.3](https://egovstandards.gov.in/sites/default/files/Face_Image_Data_Standard_Ver1.0.pdf)).
- Exposure, shadows and hot spots. The Indian standard's checklist asks for no motion blur, no over or under exposure, no shadows over the face or in the eye sockets, and no hot spots ([eGov Face Image Data Standard v1.0, 6.2](https://egovstandards.gov.in/sites/default/files/Face_Image_Data_Standard_Ver1.0.pdf)). OFIQ has separate measures for under-exposure, over-exposure, illumination uniformity, dynamic range and luminance ([OFIQ `ofiq_config.jaxn`](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/data/ofiq_config.jaxn)).
- One person in frame. OFIQ has a `SingleFacePresent` measure ([OFIQ config](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/data/ofiq_config.jaxn)). India's DoT rules say no other person may come into the frame ([DoT Digital KYC instructions, 03.04.2019, para f](https://www.dot.gov.in/static/uploads/2026/02/94c5c5c3b29c65d994a1c0e6dce211fc.pdf)).

ID document image.

- Whole document in frame, flat and square to the camera. Apple's `VNDetectDocumentSegmentationRequest` returns the four corner points of a document quadrilateral and a saliency mask ([Apple docs](https://developer.apple.com/documentation/vision/vndetectdocumentsegmentationrequest)). ML Kit's document scanner does automatic capture with document detection, edge detection and rotation detection ([ML Kit document scanner](https://developers.google.com/ml-kit/vision/doc-scanner)). India's DoT rules say the document is placed horizontally, captured vertically from above, with no skew or tilt ([DoT Digital KYC instructions 2019, para g](https://www.dot.gov.in/static/uploads/2026/02/94c5c5c3b29c65d994a1c0e6dce211fc.pdf)).
- Lighting and legibility. DoT says the customer photo and documents must be captured in proper light so that they are clearly readable and identifiable ([DoT 2019, para h](https://www.dot.gov.in/static/uploads/2026/02/94c5c5c3b29c65d994a1c0e6dce211fc.pdf)). RBI's KYC Master Direction has the same sentence for Digital KYC ([RBI Master Direction on KYC, 2016, updated 14 Aug 2025](https://www.rbi.org.in/scripts/BS_ViewMasDirections.aspx?id=11566)).
- Glare. I found no first-party mobile API that reports glare on a document (unverified). A simple check counts the share of near-saturated pixels inside the document quad (recommendation, no primary source).
- Resolution and DPI. A phone photo has no physical DPI unless you know the card's size in the frame (unverified). What matters is how many pixels span the card. The Indian face standard specifies 300 ppi for camera and scanner input ([eGov Face Image Data Standard v1.0, 6.2](https://egovstandards.gov.in/sites/default/files/Face_Image_Data_Standard_Ver1.0.pdf)). I found no primary source that sets a minimum pixel width for ID card OCR on phones (unverified).

### 6. Face image quality standards

ISO/IEC 29794-5:2025, Biometric sample quality, Part 5: Face image data. Published April 2025, 63 pages. It sets requirements for software that measures how well one face image conforms to canonical face images, for example those in ISO/IEC 39794-5:2019 Clause D.1. It covers three use cases: reference samples for ID documents, system enrolment, and probes for instantaneous response. It does not cover pairs or sequences of images, 3D, encoding of quality data, or evaluating quality algorithms. ISO lists it at stage 90.92, "to be revised", with a new CD under development ([ISO 81005](https://www.iso.org/standard/81005.html)). I did not open the full text, which is paywalled. Its individual measure definitions and any thresholds are unverified here. BSI links a draft FDIS from its OFIQ page, which I did not open ([BSI OFIQ](https://bsi.bund.de/dok/OFIQ-e)).

ISO/IEC 19794-5, Face image data interchange format. The 2005 edition specifies scene, photographic, digitization and format requirements for face images used for human and automated recognition, in JPEG or JPEG 2000, with best-practice annexes for travel-document photos ([ISO 38749](https://www.iso.org/standard/38749.html)). ISO says the 2005 edition was revised by a 2011 edition and "exceptionally remains valid until January 2040" ([ISO 38749](https://www.iso.org/standard/38749.html)). Its successor format is ISO/IEC 39794-5:2019, which also includes acquisition requirements and best practices ([ISO 72156](https://www.iso.org/standard/72156.html)). I did not open either full text, so the numeric photo requirements are unverified. The Indian e-Governance standard cites 19794-5 Annex A.3.1 for its inter-eye distance and image size values ([eGov Face Image Data Standard v1.0](https://egovstandards.gov.in/sites/default/files/Face_Image_Data_Standard_Ver1.0.pdf)).

ICAO Doc 9303 and the ICAO Portrait Quality technical report. ICAO's site blocked automated download with a Cloudflare challenge, so I could not open Doc 9303 Part 3 or the "Portrait Quality (Reference Facial Images for MRTD)" technical report, v1.0, April 2018 (unverified).

BSI OFIQ. OFIQ (Open Source Face Image Quality) is a C/C++ library from Germany's BSI. Its README says it is the reference implementation for ISO/IEC 29794-5 ([OFIQ README](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/README.md); [BSI OFIQ](https://bsi.bund.de/dok/OFIQ-e)). The current version file reads 1.2.1 ([OFIQ `Version.txt`](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/Version.txt)). It has build instructions and demo apps for Android and iOS ([OFIQ `mobile/BUILD.md`](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/mobile/BUILD.md)). The default configuration computes these measures: HeadPose, InterEyeDistance, NoHeadCoverings, BackgroundUniformity, DynamicRange, ExpressionNeutrality, EyesVisible, EyesOpen, FaceOcclusionPrevention, CropOfTheFaceImage, IlluminationUniformity, MouthOcclusionPrevention, MouthClosed, OverExposurePrevention, Sharpness, SingleFacePresent, UnderExposurePrevention, UnifiedQualityScore, NaturalColour, CompressionArtifacts, Luminance and HeadSize ([OFIQ `ofiq_config.jaxn`](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/data/ofiq_config.jaxn)). Each raw value goes through a configurable sigmoid mapping ([OFIQ `Measure.cpp`](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/OFIQlib/modules/measures/src/Measure.cpp)). I did not measure OFIQ's run time on a phone (unverified).

NIST FATE Quality. NIST runs two tracks. Specific Image Defect Detection (SIDD) tests individual quality measures. Quality Summarization tests a single quality score, now assessed through SIDD's "UnifiedQualityScore" component. NIST separates a quality scalar, one number that can trigger a recapture, from a quality vector, which reports properties such as focus, illumination, pose and expression so the user can be told what to fix ([NIST FATE Quality](https://pages.nist.gov/frvt/html/frvt_quality.html)). NIST updated its SIDD API document in April 2024 to match the ISO/IEC 29794-5 DIS. It published NISTIR 8485, covering 20 quality measures ([NIST FATE Quality](https://pages.nist.gov/frvt/html/frvt_quality.html)).

### 7. On-device options with first-party docs

Google ML Kit face detection (Android and iOS).

- It gives a bounding box, head Euler angles, landmarks, contours, smiling and eye-open probabilities, and tracking IDs ([ML Kit face detection](https://developers.google.com/ml-kit/vision/face-detection); [`Face` reference](https://developers.google.com/android/reference/com/google/mlkit/vision/face/Face)).
- It gives no blur or sharpness value. The `Face` class has no such method ([`Face` reference](https://developers.google.com/android/reference/com/google/mlkit/vision/face/Face)). The guide only says poor focus can hurt accuracy and that you should ask the user to recapture ([ML Kit face detection, Android](https://developers.google.com/ml-kit/vision/face-detection/android)).
- It detects faces and does not recognize people, so it does no face matching ([ML Kit face detection](https://developers.google.com/ml-kit/vision/face-detection)).
- Settings that matter. `PERFORMANCE_MODE_FAST` is the default, `setMinFaceSize` defaults to 0.1 of image width, and classification must be turned on to get eye-open values. For real time, use contours or classification plus landmarks, but not both. With CameraX, keep `STRATEGY_KEEP_ONLY_LATEST` so frames are dropped rather than queued. The model is about 6.9 MB bundled or about 800 KB through Google Play services ([ML Kit face detection, Android](https://developers.google.com/ml-kit/vision/face-detection/android)).

Google ML Kit document scanner (Android only).

- It provides a full scanning UI with automatic capture, edge detection, cropping and rotation. It runs on-device and needs no camera permission in your app ([ML Kit document scanner](https://developers.google.com/ml-kit/vision/doc-scanner)).
- It returns JPEG and/or PDF. It needs API level 21+ and at least 1.7 GB of total RAM, otherwise it throws `UNSUPPORTED` ([ML Kit document scanner, Android](https://developers.google.com/ml-kit/vision/doc-scanner/android)).
- `SCANNER_MODE_FULL`, the default, adds ML image cleaning that can erase stains and fingers. `SCANNER_MODE_BASE` limits editing to crop, rotate and reorder ([ML Kit document scanner](https://developers.google.com/ml-kit/vision/doc-scanner)). The docs describe no quality or blur score in the result.
- Its models and UI come through Google Play services ([ML Kit document scanner, Android](https://developers.google.com/ml-kit/vision/doc-scanner/android)), so it needs a device with Play services.

Android CameraX and Camera2.

- `CameraControl.startFocusAndMetering()` sets AF, AE and AWB regions from a `MeteringPoint`, which is "tap to focus". The default point size is 0.15, and the action auto-cancels after 5 seconds unless you set another duration. CameraX applies the action "with best effort" because not every device supports every mode ([CameraX configuration](https://developer.android.com/media/camera/camerax/configuration)).
- `FocusMeteringResult.isFocusSuccessful()` returns true when the camera does not support AF at all, so fixed-focus front cameras always report success ([CameraX `FocusMeteringResult`](https://developer.android.com/reference/androidx/camera/core/FocusMeteringResult)).
- Camera2 reports `CONTROL_AF_STATE` per frame, with values such as `PASSIVE_FOCUSED`, `FOCUSED_LOCKED` and `NOT_FOCUSED_LOCKED` ([Camera2 `CaptureResult`](https://developer.android.com/reference/android/hardware/camera2/CaptureResult)).
- `ImageAnalysis` defaults to 640x480. Exposure compensation is available through `setExposureCompensationIndex()`. CameraX may pick a different resolution from the one you ask for, so check output sizes ([CameraX configuration](https://developer.android.com/media/camera/camerax/configuration)).
- CameraX gives no blur score. Focus state means the lens believes it is focused. It says nothing about motion blur.

Apple Vision and AVFoundation (iOS).

- `VNDetectFaceCaptureQualityRequest` (iOS 13+) and the newer `DetectFaceCaptureQualityRequest` (iOS 18+) return a 0 to 1 score. Faces near 1 are "better lit, sharper, and more centrally positioned" ([Apple, VNDetectFaceCaptureQualityRequest](https://developer.apple.com/documentation/vision/vndetectfacecapturequalityrequest); [Apple, DetectFaceCaptureQualityRequest](https://developer.apple.com/documentation/vision/detectfacecapturequalityrequest)). Apple says the score is for comparing captures of the same face ([Apple, faceCaptureQuality](https://developer.apple.com/documentation/vision/vnfaceobservation/facecapturequality-bjg5)). It mixes lighting, blur and position into one number, so it cannot tell the user which one to fix.
- `AVCaptureDevice.focusMode` supports `autoFocus`, `continuousAutoFocus` and `locked`. Changing it requires `lockForConfiguration()` ([Apple, FocusMode](https://developer.apple.com/documentation/avfoundation/avcapturedevice/focusmode-swift.enum); [Apple, focusMode](https://developer.apple.com/documentation/avfoundation/avcapturedevice/focusmode-swift.property)). `isAdjustingFocus` is key-value observable, so you can hold the shutter while the lens is moving ([Apple, isAdjustingFocus](https://developer.apple.com/documentation/avfoundation/avcapturedevice/isadjustingfocus)).
- `VNDetectDocumentSegmentationRequest` (iOS 15+) returns document corners and a mask ([Apple docs](https://developer.apple.com/documentation/vision/vndetectdocumentsegmentationrequest)). `VNDocumentCameraViewController` (iOS 13+) is a ready-made scanning UI that returns page images ([Apple docs](https://developer.apple.com/documentation/visionkit/vndocumentcameraviewcontroller)). Neither documents a blur score.

Cross-platform libraries. OpenCV gives you `Laplacian`, `Sobel` and `meanStdDev` to build the focus measures above ([OpenCV imgproc filter docs](https://docs.opencv.org/4.x/d4/d86/group__imgproc__filter.html); [OpenCV core docs](https://docs.opencv.org/4.x/d2/de8/group__core__array.html)). OFIQ builds for Android and iOS ([OFIQ `mobile/BUILD.md`](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/mobile/BUILD.md)).

### 8. Indian regulator documents

These apply only if the operator is licensed in India. Check the operator's country before relying on them.

- DoT, Alternate Digital KYC for new mobile connections, 03.04.2019. It requires a live photo taken at the PoS, not a printed photo, with a white background and no other person in frame. Documents are captured from above with no skew or tilt. Photo and documents are captured "in proper light so that they are clearly readable and identifiable". A readable watermark carries CAF number, GPS coordinates, PoS name and code, date and time ([DoT Digital KYC instructions, 03.04.2019](https://www.dot.gov.in/static/uploads/2026/02/94c5c5c3b29c65d994a1c0e6dce211fc.pdf)).
- DoT, amendment to subscriber verification instructions, 31.08.2023 (letter 800-09/2023-AS.II). No connection may be activated if the auto-populated photo is "non-recognizable" or does not match the live photo. Face authentication through UIDAI is allowed for Aadhaar e-KYC. The live photo gets the same watermark ([copy enclosed in Railway Board letter dated 12.09.2024](https://telerb.railnet.gov.in/raw-attachment/wiki/UpLoads/CUGKYCInstruction.pdf)). I could not find this letter on dot.gov.in itself.
- RBI, Master Direction on KYC, 2016, updated 14 Aug 2025. Digital KYC uses the same "proper light ... clearly readable and identifiable" wording. For V-CIP, live video quality must allow identifying the customer "beyond doubt", and the application must have liveness or spoof detection and face matching ([RBI Master Direction](https://www.rbi.org.in/scripts/BS_ViewMasDirections.aspx?id=11566)). The direction is addressed to RBI-regulated entities (REs), not telecom licensees.
- UIDAI, Aadhaar Authentication API 2.5. Face data ("FID") is an ISO 19794-5 packaged image, possibly JPEG 2000. Face alone is not accepted and must go with another modality or OTP ([UIDAI Auth API 2.5, Jan 2022](https://uidai.gov.in/images/resource/Aadhaar_Authentication_API-2.5_Revision-1_of_January_2022.pdf)). I found no face image quality threshold in this document.
- Government of India e-Governance Face Image Data Standard v1.0, November 2010. It was written for human visual verification in e-Governance. It follows ISO 19794-5:2005 and sets 120 px minimum inter-eye distance, pose within ±5 degrees, eyes open, 300 ppi, and a checklist that includes "no motion blur" ([eGov Face Image Data Standard v1.0](https://egovstandards.gov.in/sites/default/files/Face_Image_Data_Standard_Ver1.0.pdf)). I found no DoT document that makes this standard mandatory for SIM KYC.

None of these documents gives a numeric sharpness, blur or glare threshold.

### 9. Recommended pipeline

Everything in this section is the note's recommendation. Each step rests on the facts cited above. Numbers marked "tune" are placeholders to calibrate, not values from any source.

1. Live checks on preview frames, before the shutter is enabled.
   - Run analysis on the latest frame only, with `STRATEGY_KEEP_ONLY_LATEST` on Android ([ML Kit Android guide](https://developers.google.com/ml-kit/vision/face-detection/android)).
   - Selfie. Run ML Kit face detection with classification on. Require exactly one face, a face box of at least 100x100 px in the analysis frame ([ML Kit](https://developers.google.com/ml-kit/vision/face-detection/android)), Euler angles within a tuned limit (±5 degrees if you follow the Indian e-Gov standard, [5.2.4](https://egovstandards.gov.in/sites/default/files/Face_Image_Data_Standard_Ver1.0.pdf)), and both eye-open probabilities above a tuned value. On iOS, add `VNDetectFaceCaptureQualityRequest` and use it to pick the best of several frames, not as a pass mark ([Apple](https://developer.apple.com/documentation/vision/vnfaceobservation/facecapturequality-bjg5)).
   - Document. Detect the card quad, with `VNDetectDocumentSegmentationRequest` on iOS or your own detector on Android. Require all four corners inside the frame with a margin, and a near-rectangular shape for the "no skew or tilt" rule ([DoT 2019](https://www.dot.gov.in/static/uploads/2026/02/94c5c5c3b29c65d994a1c0e6dce211fc.pdf)).
   - Focus. Hold the shutter while `isAdjustingFocus` is true on iOS, or while `CONTROL_AF_STATE` is scanning on Android. Use `startFocusAndMetering()` on the face or card centre ([CameraX](https://developer.android.com/media/camera/camerax/configuration); [Apple](https://developer.apple.com/documentation/avfoundation/avcapturedevice/isadjustingfocus)).
   - Sharpness. Crop to the face box or the card quad, resize to a fixed size, convert to grayscale, and compute the variance of the Laplacian. Require it above a tuned threshold for several consecutive frames. A fixed ROI size keeps scores comparable ([CVIA window-size finding](https://sites.google.com/view/cvia/focus-measure)).
   - Motion. Compute the gradient energy in x and y separately on the ROI. A large imbalance suggests directional smear ([OpenCV motion PSF](https://docs.opencv.org/4.x/d1/dfd/tutorial_motion_deblur_filter.html); per-axis idea as in [`blur_effect`](https://scikit-image.org/docs/stable/api/skimage.measure.html)).
   - Exposure and glare. Use the ROI histogram for the share of clipped bright pixels and the mean brightness. Adjust with exposure compensation before asking the user to move ([CameraX](https://developer.android.com/media/camera/camerax/configuration)).
2. Auto-capture or enable the shutter only when every check passes on N consecutive frames (tune N).
3. Post-capture check on the full-resolution still. Re-run face detection or quad detection and the sharpness score at the same fixed ROI size. The still can differ from the preview frame because of motion at the moment of capture and a different resolution ([CameraX resolution notes](https://developer.android.com/media/camera/camerax/configuration)). If you use ML Kit's document scanner, consider `SCANNER_MODE_BASE`. KYC evidence should not have stains or fingers erased by ML ([ML Kit document scanner](https://developers.google.com/ml-kit/vision/doc-scanner)).
4. Retake prompt. Report the specific failed check, as NIST's quality-vector idea suggests ([NIST FATE Quality](https://pages.nist.gov/frvt/html/frvt_quality.html)). Examples: "Hold still", "Move closer", "Tilt the card to remove glare", "Open your eyes", "Face the camera". After a set number of failures, allow capture with a flag rather than blocking the dealer forever (tune the count).
5. Logging for tuning. For every attempt, store device model, camera ID, ROI size, raw sharpness score, per-axis gradient energy, clipped-pixel share, face box size, Euler angles, eye probabilities, AF state, pass or fail per check, and whether the user retook. Join these logs with later outcomes: OCR success, face-match score, manual review result. Then set thresholds by the NIST method of measuring how much the error rate drops when the lowest-scoring images are rejected ([NIST FATE Quality](https://pages.nist.gov/frvt/html/frvt_quality.html)). Do this per device class, because focus measure behaviour is device-dependent ([CVIA](https://sites.google.com/view/cvia/focus-measure)).
6. Optional server-side or offline second opinion for the selfie. Run OFIQ on the captured face to get per-component quality values aligned with ISO/IEC 29794-5 ([OFIQ README](https://github.com/BSI-OFIQ/OFIQ-Project/blob/main/README.md)).

## Open questions

- Which country is the operator licensed in? Section 8 applies only to India. If the operator is in India, does it require the e-Gov face standard or another internal photo spec?
- What is the device range? ML Kit's document scanner needs Google Play services and at least 1.7 GB RAM. Low-end dealer phones may fall outside that.
- Do the front cameras on target devices have autofocus? Fixed-focus cameras make `isFocusSuccessful()` always true, so the sharpness score is the only focus signal.
- What ID documents are in scope (Aadhaar, PAN, voter ID, passport, others)? Card size, lamination and print quality change glare and texture, and so change thresholds.
- What are the actual threshold values? No primary source gives them. They need a labelled sample from real dealer captures.
- What do ISO/IEC 29794-5, 19794-5 and 39794-5 actually specify in detail? I only read ISO's abstracts. The full texts are paywalled.
- What do ICAO Doc 9303 Part 3 and the ICAO Portrait Quality TR say? I could not open them because the ICAO site blocked automated access.
- How fast is OFIQ on a mid-range Android phone? I have not measured it.
- How well does FFT or DCT high-frequency energy work as a blur gate on phone images? I found no primary source with a recipe.
- How do capture checks interact with the offline-first workstream? If checks run on-device, the thresholds need to ship with the app and be updatable without a release.

## Sources

All accessed 2026-10-19.

- OpenCV, Laplace Operator tutorial. https://docs.opencv.org/4.x/d5/db5/tutorial_laplace_operator.html
- OpenCV, imgproc Image Filtering (cv::Laplacian). https://docs.opencv.org/4.x/d4/d86/group__imgproc__filter.html
- OpenCV, core Operations on arrays (cv::meanStdDev). https://docs.opencv.org/4.x/d2/de8/group__core__array.html
- OpenCV, Out-of-focus Deblur Filter tutorial. https://docs.opencv.org/4.x/de/d3c/tutorial_out_of_focus_deblur_filter.html
- OpenCV, Motion Deblur Filter tutorial. https://docs.opencv.org/4.x/d1/dfd/tutorial_motion_deblur_filter.html
- scikit-image 0.26, skimage.measure (blur_effect). https://scikit-image.org/docs/stable/api/skimage.measure.html
- Crete, Dolmiere, Ladret, Nicolas. The blur effect: perception and estimation with a new no-reference perceptual blur metric. SPIE 6492, 2007. https://doi.org/10.1117/12.702790 (cited via scikit-image docs; paper not read)
- Pertuz, Puig, Garcia. Analysis of focus measure operators for shape-from-focus. Pattern Recognition 46(5):1415-1432, 2013. https://doi.org/10.1016/j.patcog.2012.11.011 (abstract and highlights read; full text not read)
- Pertuz, CVIA Focus Measure project page. https://sites.google.com/view/cvia/focus-measure
- Pertuz, Focus Measure (fmeasure.m, operators.txt), MATLAB File Exchange 27314, v2.2.0.0. https://www.mathworks.com/matlabcentral/fileexchange/27314-focus-measure
- Pech-Pacheco, Cristobal, Chamorro-Martinez, Fernandez-Valdivia. Diatom autofocusing in brightfield microscopy: a comparative study. ICPR 2000, pp. 314-317. https://doi.org/10.1109/ICPR.2000.903548 (abstract only)
- Liu, Li, Jia. Image partial blur detection and classification. CVPR 2008. https://doi.org/10.1109/CVPR.2008.4587465 (abstract only)
- ISO/IEC 29794-5:2025 catalogue page. https://www.iso.org/standard/81005.html
- ISO/IEC 19794-5:2005 catalogue page. https://www.iso.org/standard/38749.html
- ISO/IEC 39794-5:2019 catalogue page. https://www.iso.org/standard/72156.html
- ICAO, Portrait Quality (Reference Facial Images for MRTD) TR v1.0. https://www.icao.int/sites/default/files/TRIP/Publications/TR-Portrait-Quality-v1.0.pdf (could not open, Cloudflare challenge)
- BSI, Open Source Face Image Quality (OFIQ). https://bsi.bund.de/dok/OFIQ-e
- BSI-OFIQ/OFIQ-Project on GitHub: README.md, Version.txt, mobile/BUILD.md, data/ofiq_config.jaxn, OFIQlib/modules/measures/src/Sharpness.cpp, OFIQlib/modules/measures/src/Measure.cpp. https://github.com/BSI-OFIQ/OFIQ-Project
- NIST, Face Analysis Technology Evaluation (FATE) Quality. https://pages.nist.gov/frvt/html/frvt_quality.html
- Google ML Kit, Face detection overview. https://developers.google.com/ml-kit/vision/face-detection
- Google ML Kit, Detect faces on Android. https://developers.google.com/ml-kit/vision/face-detection/android
- Google ML Kit, Face class reference. https://developers.google.com/android/reference/com/google/mlkit/vision/face/Face
- Google ML Kit, Document scanner overview. https://developers.google.com/ml-kit/vision/doc-scanner
- Google ML Kit, Document scanner on Android. https://developers.google.com/ml-kit/vision/doc-scanner/android
- Android Developers, CameraX configuration options. https://developer.android.com/media/camera/camerax/configuration
- Android Developers, FocusMeteringResult. https://developer.android.com/reference/androidx/camera/core/FocusMeteringResult
- Android Developers, Camera2 CaptureResult. https://developer.android.com/reference/android/hardware/camera2/CaptureResult
- Apple, VNDetectFaceCaptureQualityRequest. https://developer.apple.com/documentation/vision/vndetectfacecapturequalityrequest
- Apple, DetectFaceCaptureQualityRequest. https://developer.apple.com/documentation/vision/detectfacecapturequalityrequest
- Apple, VNFaceObservation.faceCaptureQuality. https://developer.apple.com/documentation/vision/vnfaceobservation/facecapturequality-bjg5
- Apple, AVCaptureDevice.FocusMode. https://developer.apple.com/documentation/avfoundation/avcapturedevice/focusmode-swift.enum
- Apple, AVCaptureDevice.focusMode. https://developer.apple.com/documentation/avfoundation/avcapturedevice/focusmode-swift.property
- Apple, AVCaptureDevice.isAdjustingFocus. https://developer.apple.com/documentation/avfoundation/avcapturedevice/isadjustingfocus
- Apple, VNDetectDocumentSegmentationRequest. https://developer.apple.com/documentation/vision/vndetectdocumentsegmentationrequest
- Apple, VNDocumentCameraViewController. https://developer.apple.com/documentation/visionkit/vndocumentcameraviewcontroller
- DoT, Instructions for Alternate Digital KYC Process, 03.04.2019. https://www.dot.gov.in/static/uploads/2026/02/94c5c5c3b29c65d994a1c0e6dce211fc.pdf
- DoT letter 800-09/2023-AS.II dated 31.08.2023, as enclosed in Railway Board letter 2021/Tele/11(2)/1 dated 12.09.2024. https://telerb.railnet.gov.in/raw-attachment/wiki/UpLoads/CUGKYCInstruction.pdf
- RBI, Master Direction - Know Your Customer (KYC) Direction, 2016, updated 14 Aug 2025. https://www.rbi.org.in/scripts/BS_ViewMasDirections.aspx?id=11566
- UIDAI, Aadhaar Authentication API 2.5, Revision 1, January 2022. https://uidai.gov.in/images/resource/Aadhaar_Authentication_API-2.5_Revision-1_of_January_2022.pdf
- Government of India, Face Image Data Standard v1.0 (eGov.BIDS:01), November 2010. https://egovstandards.gov.in/sites/default/files/Face_Image_Data_Standard_Ver1.0.pdf
