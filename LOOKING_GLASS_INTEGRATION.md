# Looking Glass WebXR Integration with IWER

This document describes how to integrate IWER (Immersive Web Emulation Runtime) with the Looking Glass WebXR library, replacing the original `@lookingglass/webxr-polyfill`.

## Summary

Successfully replaced the WebXR polyfill in Looking Glass' WebXR library with IWER. The integration required minimal code changes and the library builds successfully.

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

## Build Results

The library builds successfully:
- `dist/webxr.js` - ES module build (39.54 KiB)
- `dist/webxr.umd.cjs` - UMD build (31.50 KiB)

## Testing

After making these changes, the Looking Glass WebXR library should work with IWER as the underlying WebXR polyfill. The API surface remains the same for users of the library.

## Location of Modified Code

The modified Looking Glass WebXR repository is located at:
```
/home/user/looking-glass-webxr
```

All changes are ready to be committed to the Looking Glass repository.
