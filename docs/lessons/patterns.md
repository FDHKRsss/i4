# Patterns

_Reusable techniques that worked, so they are reused not rediscovered._

- Test a real `getUserMedia` React camera in jsdom by installing `navigator.mediaDevices` (via `Object.defineProperty`) whose `getUserMedia` resolves a fake `MediaStream` (`{ getTracks: () => [{ stop: vi.fn() }] }`) and spying `HTMLMediaElement.prototype.play` → `mockResolvedValue`. Then assert the exact constraint object (`{video:{facingMode:"environment"},audio:false}`), `video.srcObject`, that the shutter stays disabled while a never-resolving promise keeps it "starting", that shutter calls `compressToImages(video)`, and that `track.stop()` runs on unmount; `delete navigator.mediaDevices` in `afterEach`.
- Test the real canvas downscale/JPEG-encode path in jsdom by bypassing BOTH headless signals: (a) override `navigator.userAgent` to drop the "jsdom" marker, and (b) install a working `HTMLCanvasElement.prototype.toBlob` that records `{width,height,type,quality}` then resolves a JPEG blob, plus spy `getContext` → fake 2D context with `drawImage`. Assert bounded edges (≤1280 full @0.85, ≤360 thumb @0.72) and no upscale below the bound.
