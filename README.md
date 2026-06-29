# SSTV Communicator

Single-file HTML/JS/CSS application that converts images to SSTV audio for transmission over radio, and decodes SSTV audio back into images in real time. Optimized for the Baofeng UV-5R handheld transceiver.

## How It Works

### Protocol

The application uses a custom SSTV-like protocol designed for narrowband FM radios:

- **Leader tone**: 1900 Hz for 1 second (opens squelch on the receiving radio)
- **VIS (Vertical Interval Signaling)**: 1200 Hz sync, then a mode tone (1800 Hz = grayscale, 2000 Hz = color), then another sync
- **Image data**: Each line consists of a 1200 Hz sync pulse (15 ms), a 1500 Hz porch (3 ms), pixel tones (2 ms per pixel), and a 1500 Hz guard tone (2 ms)
- **Pixel mapping**: 1500 Hz = black, 2300 Hz = white, with linear interpolation for grayscale values
- **Frequency range**: 1200-2300 Hz, chosen to fit within the UV-5R's audio passband (~400-2500 Hz)

### Transmit Mode

1. Upload an image (tap the upload area or drag and drop)
2. Select the color mode (grayscale or color RGB) and resolution (64x48, 128x96, or 160x120)
3. Press Transmit. The application generates a raw audio buffer containing the encoded SSTV signal and plays it through the speaker.
4. Hold the Baofeng UV-5R's microphone near the speaker. The radio transmits the tones over the air.

### Receive Mode

1. Press Listen and grant microphone access.
2. Hold the receiving Baofeng UV-5R near the computer microphone.
3. The decoder captures audio in real time via the Web Audio API.
4. For every 2 ms chunk of audio, it measures the dominant frequency using zero-crossing detection.
5. A state machine processes the frequency stream:
   - **IDLE** -- listens for the 1900 Hz leader tone
   - **LEADER** -- leader detected; waits for 1200 Hz sync
   - **VIS_SYNC** -- first VIS sync detected
   - **VIS_PORCH** -- waits for the mode tone (1800 Hz or 2000 Hz)
   - **VIS_MODE** -- mode identified; waits for second sync
   - **LINE_WAIT** -- ready for image data; waits for line sync pulses
   - **LINE_SYNC** -- line sync detected; skips the 1500 Hz porch (2 chunks)
   - **LINE_DATA** -- reads pixels sequentially, mapping each chunk's frequency to a brightness value (0-255)
   - **LINE_DONE** -- stores the completed line and renders it to the canvas
   - **DONE** -- all lines received or 8-second timeout triggers completion
6. The decoded image appears line by line on the canvas as it is received.
7. Press Save to download the received image as PNG.

### Error Handling

- **Porch skip**: After each sync pulse, the decoder discards the first 2 chunks (4 ms) to avoid reading the 1500 Hz porch tone as pixel data.
- **Frequency smoothing**: The last 8 frequency measurements are averaged to reduce noise from zero-crossing jitter.
- **Confidence scoring**: Each pixel gets a confidence value based on how stable the frequency was during its 2 ms window. The overall confidence percentage is shown in the status panel.
- **Sync quality**: The timing between consecutive sync pulses is measured. Deviation from the expected line duration reduces the sync percentage.
- **Timeout**: If no sync pulse is detected for 8 seconds after at least one line has been decoded, the decoder assumes the transmission ended and marks the image as complete.

## Baofeng UV-5R Setup

1. Set Menu #5 (Bandwidth) to **Wide** (WN).
2. Set Menu #35 (Compander) to **OFF**.
3. Start with computer volume at ~50%. Increase until the radio shows modulation without distortion.
4. On the receiving end, keep the radio volume moderate to avoid clipping the microphone input.

## Serving the File

The application must be served over HTTP/HTTPS for microphone access to work. Run a simple server in the project directory:

```
npx serve .
python -m http.server 8080
```

Then open the URL in a browser. Opening the file directly via `file://` will block microphone access in most browsers.
