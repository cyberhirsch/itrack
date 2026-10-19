# iTrack

Webcam eye tracking in the browser. Show people images and videos and record where they look,
with nothing but a webcam and nothing uploaded.

## How it works

1. **Setup.** Add your images and videos, turn on the camera, check the session settings.
2. **Start.** The screen goes fullscreen and a voice guides you (English or German):
   follow the dot with your eyes, follow it as it moves, then keep your eyes on it while you turn
   your head a little. A short accuracy check follows; if it is poor, the calibration repeats once.
3. **Tracking.** Your media play in order. Before each file a cross appears for a moment and
   corrects drift. Press **Esc** at any time to stop.
4. **Results.** Heatmaps and scanpaths per file (for videos around the playhead), statistics,
   and export as CSV (samples and fixations), session JSON (opens again later) and PNG.

## Run it

Open `index.html` over **https** or **http://localhost** (the camera needs a secure context):

```
npx serve .
```

or any static web server, or GitHub Pages. Chrome or Edge recommended (WebGL, speech voices).

## Under the hood

- **Face and iris landmarks:** MediaPipe Face Landmarker, 478 points including both irises.
- **Head crop:** the face is followed as a region of the full-resolution camera frame (up to 4K),
  cut out and enlarged before the landmarks are found.
- **Eye crops:** each eye is cut from the full frame, levelled and enlarged; the iris centre is
  refined from the darkest pixels inside the eye opening.
- **Head pose:** yaw, pitch, roll, position and distance from the landmarks, smoothed.
- **Gaze model:** ridge regression from the eye-region landmarks (the EyeTrax feature set) or the
  iris offset, plus the head pose, to the screen.
- **Head movement compensation:** learned from the head sweep in the calibration, where the target
  stays and the head moves.
- **Calibration:** fixation points (the dot stops, pulses and shrinks), a smooth-pursuit curve,
  the head sweep, then a 5-point accuracy check reported in pixels and degrees.
- **Smoothing:** One Euro filter or a Kalman filter tuned by the accuracy check.
- **Analysis:** I-DT fixations, duration-weighted heatmaps, scanpaths.

Expect about 1–3° accuracy (roughly 50–150 px) with a good webcam, even light and a still head.

## History

`history/` holds the four experiment versions this tool grew from (v001–v004) and their notes,
including the comparison with WebGazer, EyeTrax, GazeRecorder, RealEye and OpenTrack.

## Credits

EyeTrax (MIT) for the feature set and calibration ideas; MediaPipe (Apache-2.0) for the landmarks.
See [THIRD_PARTY.md](THIRD_PARTY.md).
