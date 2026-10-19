# № 27 — iTrack

Webcam eye tracking in the browser. The user loads images and videos, calibrates by looking at the
screen corners and clicking them, then the files play in sequence, fullscreen, while the gaze is
recorded. A review shows heatmaps and scanpaths per file; everything exports as CSV, JSON and PNG.
Nothing leaves the browser.

## Existing open source, and the choice

| Project | What it is | Licence | Status |
|---|---|---|---|
| [WebGazer.js](https://github.com/brownhci/WebGazer) (Brown HCI) | The classic browser eye tracker: face mesh, eye patches, ridge regression, self-calibrates from clicks | GPL-3.0 (LGPL for small companies) | 3.5.3; official maintenance has ended, community support only |
| [jsPsych WebGazer fork](https://github.com/jspsych/WebGazer) | WebGazer packaged for jsPsych experiments | GPL-3.0 | maintained with jsPsych |
| [MediaPipe Face Landmarker](https://ai.google.dev/edge/mediapipe/solutions/vision/face_landmarker) | 478 face landmarks including both irises, head pose matrix, 52 blendshapes (blinks, eye look) | Apache-2.0 | maintained by Google |
| HueVision ([Roboflow write-up](https://blog.roboflow.com/build-eye-tracking-in-browser/)) | MediaPipe FaceMesh plus a TensorFlow.js regressor from iris to screen | — | tutorial |
| [EyeTrax](https://github.com/ck-zhang/eyetrax) | Python, MediaPipe: 161 eye-region landmarks in a head frame plus yaw / pitch / roll, ridge (or SVR, MLP); 9 / 5-point, Lissajous, dense-grid and adaptive calibration; Kalman / KDE smoothing | MIT | active (2026) |
| [GazeRecorder](https://gazerecorder.com/download-gazerecorder/) | Desktop webcam eye tracking for studies (heatmaps, recordings); GazeCloud / GazeFlow API and SDK | closed, free for non-commercial use | product |
| RealEye ([video](https://www.youtube.com/watch?v=feVK53ONNok)) | Online webcam eye-tracking studies; the video calls a light version open source, no repository found | proprietary platform | product |
| [OpenTrack NeuralNet tracker](https://github.com/opentrack/neuralnet-tracker-traincode) | Webcam head tracking (6 DoF) for simulators: one ONNX net finds the face, a second estimates head pose; training code public | ISC (opentrack) | active |

**Chosen:** MediaPipe Face Landmarker for the landmarks, with our own feature, calibration and
analysis. Reasons: Apache-2.0 instead of GPL (matters if this is ever sold or embedded), active
maintenance, true iris landmarks instead of eye-patch pixels, and a head pose for free. WebGazer's
self-calibration from every click would also skew a viewing study (people look at what they click).

## v001

`iTrack_v001.html`, one file. `@mediapipe/tasks-vision@0.10.14` comes from jsDelivr and the float16
face landmarker model from Google's model storage (about 4 MB together, cached by the browser).

**Gaze feature.** Per eye: the iris centre (mean of the 5 iris landmarks) projected onto the axis
between the two eye corners and onto its normal, both divided by the eye width (ex, ey); the two eyes
are averaged. Width-normalised, so it does not change with the distance to the camera.

**Head compensation (slider, default 0).** ex += k · 0.4 · yaw, ey += k · 0.4 · pitch. The eyeball
radius is about 0.4 eye widths, so turning the head moves the iris by about 0.4 eye widths per radian
in the opposite direction. The sign depends on the camera, so k is signed; tune it in Test gaze by
fixating a cross and turning the head, then recalibrate.

**Calibration.** 4 corner targets (inset 40 px) or 9 (corners, edge midpoints, centre), 1–3 rounds
(the second runs backwards). Look at the dot and click it; the frames from 0.4 s before to 0.4 s after
the click are samples, blinks removed (blink blendshape > 0.45), and per target samples further than
3 MAD from the median are dropped. The map is a ridge least squares (λ = 10⁻³ N, standardised terms)
from (1, ex, ey, ex·ey), an exact bilinear map for 4 targets, plus ex² and ey² for 9, to the
normalised screen position. Synthetic check (eye signal with curvature and noise): 3–5 px validation
error on 1920 × 1080, with a planted outlier removed.

**Validation.** 5 points (centre and the four 25 / 75 % points), 1.7 s each, shown automatically. The
median gaze from 0.65 s on gives the accuracy (mean error in px, and degrees from the head distance in
the pose matrix at 96 CSS px per inch) and the precision (RMS spread). Webcam trackers reach roughly
60–150 px.

**Run.** Fullscreen. Before each file an optional drift check: look at the cross, press Space; the
offset to the centre is corrected additively, and a warning shows when the head has moved more than
4 cm from the calibration position. Images show for a set time, videos play to their end (muted by
default; if autoplay with sound is refused, it retries muted). → or N skips, Esc stops and keeps what
was recorded. Every camera frame is a sample: time, video time, valid / blink, smoothed and raw screen
px, and u / v inside the displayed media (0..1).

**Smoothing.** One Euro filter on the normalised gaze (minimum cutoff 0.05–4 Hz from the Smoothing
slider, β 0.6), reset after gaps over 0.3 s.

**Review.** Per file: valid samples, gaze on the media, fixation count and mean duration, sample rate.
Heatmap: gaussian splats on a grid of at most 320 px, weighted by sample duration, colour ramp blue ›
green › yellow › red. Fixations: I-DT, dispersion (x range + y range) 70 px and at least 100 ms by
default, both adjustable. Scanpath: numbered circles sized by duration. Videos: heatmap and scanpath
over a window before the playhead, plus a ring at the gaze at the playhead.

**Export.** CSV of every sample plus a fixation table; a session JSON (options, calibration quality,
per-file samples and fixations) that loads back for review (re-add the same files by name to see them
underneath); PNG of the current view at the media resolution.

**Limits.** Head movement is the main error source: the calibration holds for the head position it
was made in (the drift check and head compensation help). Vertical gaze is weaker than horizontal (the
iris moves less and the lids follow it). Glasses with reflections and low light cost accuracy. The
camera needs a secure context: `file://` works in Chrome; other browsers may need `http://localhost`.

## v002 — head rotation tracked and learned, head and eyes cropped and enlarged

`iTrack_v002.html`, after EyeTrax and the request that head rotation be tracked for compensation and
the head be cropped and blown up.

**Head crop.** The camera is asked for 1920 × 1080. The face is followed as a square region (1.6 ×
the landmark extent, smoothed) in the full frame; that region is cut out and enlarged to 384 / 512 /
768 px and a second landmarker instance runs on it, so the face fills the network input wherever it
sits; the landmarks are mapped back exactly. When the face is lost, one full-frame pass finds it again.
Honest note: MediaPipe already crops the face internally to a fixed input, so the head crop mostly
helps detection stability at a distance and at the frame edges; the real resolution gain is in the
eye crops.

**Eye crops.** Each eye is cut from the full-resolution frame, rotated level and enlarged (120 × 60);
the iris centre is refined as the centroid of the darkest quarter of the pixels inside the eye opening
(an ellipse between the corners and lids) and mixed with the landmark iris (Iris refine, default 0.5).
Both crops are shown, with the opening and the refined centre.

**Head pose from the landmarks** (EyeTrax's frame): x along the outer eye corners, y towards the
forehead (orthogonalised), z = x × y; yaw, pitch, roll, face centre and inter-eye distance, smoothed
with a One Euro filter. Distance ≈ focal length × 6.3 cm / inter-eye px, at a 60° field of view.
Shown as gauges in the setup view and recorded with every sample.

**Learned head compensation.** The head pose (6 values) goes into the regression. That only works if
the calibration contains head movement with a known target, so a **head sweep** follows the clicks:
look at the centre dot for 10 s while turning the head left, right, up, down and tilting it. Synthetic
check (eye signal = gaze angle − head angle): without the sweep, a 17° head turn gives 1134 px error;
with it, 13 px.

**Smooth pursuit** (EyeTrax's Lissajous 3 : 2, 0.42 of the screen, slow at the ends), 8 s, the target
taken 100 ms earlier for the eye's lag: many distinct targets quickly.

**Models.** Eye region (EyeTrax features: 161 landmarks in the head frame divided by the inter-eye
distance, 483 values) or iris offset (bilinear / quadratic), both plus the head pose; ridge on
standardised features (eye region λ = 10^slider, default 1 like EyeTrax). The eye-region model falls
back to the iris model without the pursuit or 9 targets (too few distinct targets for 489 features).

**Also from EyeTrax:** blinks as an eye aspect ratio below 0.8 × its running mean (or the blendshape);
an automatic capture mode (1 s pulse, 1 s capture with a closing ring, no clicking); a Kalman filter
(constant velocity) whose measurement noise comes from the validation precision; the calibration is
saved in the browser and can be reused.

## v003 — full camera resolution

`iTrack_v003.html`. v002 asked for 1920 × 1080 (`ideal`), so a 4K camera delivered FHD. Now the
resolution is an option, Maximum by default: the camera is opened with a very large ideal size, then
the track's capabilities are read and the largest width and height are applied. The delivered size,
frame rate and the camera's maximum are logged and shown under Tracking. Changing the option reopens
the camera. Some cameras give 4K only at low frame rates or over MJPEG; step down if the fps drops.

## v004 — why it stayed 1080

`iTrack_v004.html`. v003 still delivered 1920 × 1080 on a 4K camera. Cause: the size request also said
`frameRate: { ideal: 30 }`. With ideal constraints the browser picks the mode with the smallest summed
fitness distance, so a camera whose 4K mode runs below 30 fps loses to 1080 at 30. Now no frame rate
is requested unless *Prefer 30 fps* is on. When the delivered size is still below the reported
maximum, the exact maximum is forced and a refusal is logged (error name and constraint). A camera
picker lists every video input (virtual cameras, e.g. OBS or vendor apps, often sit in front at
1080), and *Probe all cameras* opens each one briefly and reports its maximum size, frame-rate range
and the size it opened at. If the reported maximum itself is 1920 × 1080, the browser does not see the
camera's 4K mode (a virtual camera, the vendor app holding it, or the driver).

## Ideas

- OpenTrack's NeuralNet head-pose ONNX models in onnxruntime-web, if MediaPipe's landmark pose is too noisy
- EyeTrax's adaptive calibration (60 blue-noise points with retraining every 10)
- a per-file mini recalibration (2–3 points) instead of the single drift point
- areas of interest (boxes, dwell time, time to first fixation) and multi-viewer heatmaps from several
  session JSONs
- the eye-look blendshapes (eyeLookIn / Out / Up / Down) as extra regressors for vertical gaze
