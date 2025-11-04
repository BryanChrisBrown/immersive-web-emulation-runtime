# Looking Glass WebXR Integration with IWER

This document describes how to integrate IWER (Immersive Web Emulation Runtime) with the Looking Glass WebXR library, replacing the original `@lookingglass/webxr-polyfill`.

## Summary

Successfully replaced the WebXR polyfill in Looking Glass' WebXR library with IWER. The integration required custom multi-view rendering support, as Looking Glass displays require 45-48 views (not the standard 2 stereo views). The library builds successfully and provides proper multi-view WebXR support.

## Multi-View Architecture Challenge

### The Problem

Standard WebXR (and IWER) is designed for stereo VR headsets with 2 views (left and right eye). Looking Glass holographic displays require **45-48 views** arranged in a quilt pattern to create the holographic effect.

- **IWER's XRFrame.getViewerPose()**: Returns 2 XRView objects (left/right)
- **Looking Glass needs**: 45-48 XRView objects (multi-perspective lightfield)

### The Solution

Created custom WebXR classes that bridge IWER's API with Looking Glass's multi-view requirements:

1. **LookingGlassXRFrame** - Overrides `getViewerPose()` to return 45-48 views
2. **LookingGlassXRSession** - Creates custom frames and manages animation loop
3. **LookingGlassXRSystem** - Creates custom sessions and coordinates with device

## Changes Made

### 1. Updated Dependencies (package.json)

Replaced the old polyfill dependency:
```json
{
  "dependencies": {
    "iwer": "file:../immersive-web-emulation-runtime",
    "gl-matrix": "^2.8.1",
    "holoplay-core": "^0.0.11"
  }
}
```

### 2. Updated LookingGlassWebXRPolyfill.ts

- Imported all XR classes from IWER instead of the old polyfill
- Created a minimal `WebXRPolyfill` base class for compatibility
- Created an `API` object containing all XR classes for the injection pattern
- Updated `XRSystem` constructor call to pass device directly instead of a Promise

### 3. Updated LookingGlassXRDevice.ts

- Imported `XRSpace` from IWER
- Created a minimal `XRDevice` base class that extends `EventTarget`
- Added property declarations for all class properties to fix TypeScript errors

### 4. Updated LookingGlassXRWebGLLayer.ts

- Imported `XRWebGLLayer` and `P_WEBGL_LAYER` from IWER
- Created compatibility alias: `XRWebGLLayer_PRIVATE = P_WEBGL_LAYER`
- Replaced access to parent class config with direct `layerInit` parameter handling

### 5. Updated index.d.ts

- Removed module declarations for the old polyfill packages
- Type declarations are now provided directly by IWER

## Key Compatibility Notes

### Private Symbol Changes

IWER uses `P_WEBGL_LAYER` instead of the old polyfill's `PRIVATE` symbol. We created an alias for compatibility.

### XRSystem Constructor

IWER's `XRSystem` constructor takes a device directly:
```typescript
this.xr = new XRSystem(this.device)
```

The old polyfill took a Promise:
```typescript
this.xr = new XRSystem(Promise.resolve(this.device))
```

### Config Handling

The old polyfill stored layer config in the private state. IWER doesn't expose it the same way, so we handle it directly from the `layerInit` parameter:
```typescript
const config = {
  depth: layerInit?.depth !== false,
  stencil: layerInit?.stencil || false,
  antialias: layerInit?.antialias !== false,
  alpha: layerInit?.alpha !== false
}
```

### 6. Created Custom Multi-View Classes

#### LookingGlassXRFrame.ts
Custom XRFrame that overrides `getViewerPose()` to create 45-48 XRView objects:

```typescript
getViewerPose(referenceSpace: XRReferenceSpace): XRViewerPose {
  const viewSpaces = device.getViewSpaces('immersive-vr'); // Gets 48 view spaces

  const views: XRView[] = [];
  for (let i = 0; i < viewSpaces.length; i++) {
    const projectionMatrix = device.getProjectionMatrix(null, i);
    const viewMatrix = device._getViewMatrixByIndex(i);
    const transform = this.createTransformFromMatrix(viewMatrix);
    views.push(new XRView('none', projectionMatrix, transform, session));
  }

  return new XRViewerPose(baseTransform, views, false);
}
```

#### LookingGlassXRSession.ts
Custom XRSession that:
- Creates LookingGlassXRFrame instances in requestAnimationFrame
- Manages Looking Glass device's internal session ID
- Calls device.onFrameStart() and device.onFrameEnd() for each frame

#### LookingGlassXRSystem.ts
Custom XRSystem that:
- Calls device.requestSession() to get internal session ID
- Creates LookingGlassXRSession wrapper
- Tracks active sessions

## Build Results

The library builds successfully with multi-view support:
- `dist/webxr.js` - ES module build (43.79 KiB)
- `dist/webxr.umd.cjs` - UMD build (34.67 KiB)

## How Multi-View Rendering Works

### Frame Flow

1. **Application calls** `session.requestAnimationFrame(callback)`
2. **LookingGlassXRSession** schedules frame via device's rAF
3. **Device calls** `onFrameStart()` to compute 48 view/projection matrices
4. **LookingGlassXRFrame** created with session and passed to callback
5. **Application calls** `frame.getViewerPose(refSpace)`
6. **LookingGlassXRFrame** returns XRViewerPose with 48 XRView objects
7. **Application renders** each view to corresponding viewport in quilt
8. **Device calls** `onFrameEnd()` to composite quilt to display

### View Matrix Calculation

Looking Glass computes view matrices for each of the 48 camera positions:

```javascript
// Each view is offset along a baseline (horizontal arc)
for (let i = 0; i < 48; i++) {
  const fraction = (i + 0.5) / 48 - 0.5;  // -0.5 to 0.5
  const angle = viewCone * fraction;
  const offset = focalDistance * Math.tan(angle);

  // Translate camera along baseline
  const viewMatrix = translate(basePose, [offset, 0, 0]);
  const projectionMatrix = computeAsymmetricFrustum(offset, ...);
}
```

## Testing

After making these changes, the Looking Glass WebXR library should work with IWER as the underlying WebXR polyfill. The API surface remains the same for users of the library.

## Location of Modified Code

The modified Looking Glass WebXR repository is located at:
```
/home/user/looking-glass-webxr
```

All changes are ready to be committed to the Looking Glass repository.
