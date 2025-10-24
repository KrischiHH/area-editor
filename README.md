# Area Editor

This editor allows creating and publishing AR/WebXR scenes with support for multiple anchor modes.

## Changes in publish() and Routing

The `publish()` function has been enhanced to provide robust and resilient URL handling for publishing and viewer routing. These changes improve reliability when working with different deployment scenarios and server configurations.

### 1. Robust PUBLISH_ENDPOINT Parsing

The `PUBLISH_ENDPOINT` configuration now supports both absolute and relative URLs:

**Absolute URL example:**
```javascript
window.__AREA = {
  PUBLISH_ENDPOINT: 'https://example.com/api/publish'
};
```

**Relative URL example:**
```javascript
window.__AREA = {
  PUBLISH_ENDPOINT: '/api/publish'
};
```

When a relative URL is provided, it is resolved against `location.href`. If URL parsing fails for any reason, the system falls back to `location.origin` to ensure the publish operation doesn't fail completely.

### 2. Server Response URL Normalization

When the server returns a `viewerUrl` in the JSON response:

- If `viewerUrl` is an **absolute URL**, it is used as-is
- If `viewerUrl` is a **relative path**, it is normalized to an absolute URL using `VIEWER_BASE` as the base
- If `viewerUrl` is malformed or parsing fails, the system falls back to constructing a viewer URL using the configured `VIEWER_BASE` and mode-specific entry points

This ensures that viewer URLs work correctly regardless of how the server is configured to return them.

### 3. Base Query Parameter Validation

The generated viewer URL includes a `base` query parameter that tells the viewer where to load scene assets from. The validation ensures:

- The `base` parameter must be a valid origin (protocol + hostname + optional port)
- Invalid origins (empty, malformed, or unparseable) are replaced with the `workerOrigin`
- This prevents issues with relative paths or invalid URLs in the `base` parameter

Example valid base values:
- `https://example.com`
- `https://example.com:8080`
- `http://localhost:3000`

### 4. FormData Fields and Backward Compatibility

The current implementation uses multiple `file` entries in the FormData for backward compatibility:

```javascript
form.append('sceneId', sceneId);
form.append('file', glbFile);      // scene.glb
form.append('file', posterFile);   // poster.jpg (scene-viewer mode only)
form.append('file', audioFile);    // audio.mp3 (scene-viewer mode only)
form.append('file', sceneJsonFile); // scene.json
```

**Future Server Updates:**

If you update your backend server, consider using explicit field names for clearer semantics:
- `glb` - for the GLB model file
- `poster` - for the poster image (scene-viewer mode)
- `audio` - for audio files (scene-viewer mode)
- `sceneJson` - for the scene configuration JSON

This would make the API more self-documenting and easier to work with, but requires server-side changes to accept these named fields.

### 5. EDITOR_WORKER_BASE Configuration

For local development and preview scenarios, you can configure an editor-specific worker base:

```javascript
window.__AREA = {
  PUBLISH_ENDPOINT: '/api/publish',
  EDITOR_WORKER_BASE: 'http://localhost:8787',
  VIEWER_BASE: 'https://area-viewer.pages.dev'
};
```

When `anchorMode` is set to `'editor-local'` or `'local'`, and `EDITOR_WORKER_BASE` is configured, the system will use this base for generating asset URLs instead of the publish endpoint origin. This is useful for:

- Testing locally before deploying
- Previewing changes with a local worker
- Development workflows where the editor and worker run on different ports

### 6. Preview URL Markers

When a shared URL is generated with a local or editor-local base, the system automatically appends `&preview=1` to the URL. This helps recipients understand that:

- The URL may not be publicly accessible
- It's intended for preview/development purposes
- Assets may be served from a local development server

The preview marker is added when the `base` parameter matches:
- `location.origin` (current page's origin)
- The configured `EDITOR_WORKER_BASE`
- Any anchor mode that indicates local preview (`'editor-local'` or `'local'`)

## Server Contract and Recommended Response

Your publish endpoint should accept a `POST` request with:

**Request:**
- `Content-Type: multipart/form-data`
- Optional `X-AREA-Key` header for authentication
- FormData fields as described in section 4 above

**Recommended Response:**

```json
{
  "viewerUrl": "https://area-viewer.pages.dev/webxr.html?scene=my-scene&base=https://example.com",
  "sceneId": "my-scene",
  "status": "published"
}
```

The `viewerUrl` field is optional. If not provided, the editor will construct one using `VIEWER_BASE` and the appropriate entry point for the selected anchor mode.

## Configuration Options

All configuration is done via the `window.__AREA` object:

```javascript
window.__AREA = {
  // Required: Where to POST scene data
  PUBLISH_ENDPOINT: 'https://example.com/api/publish',  // or '/api/publish'
  
  // Optional: Authentication key for publish endpoint
  PUBLISH_KEY: 'your-secret-key',
  
  // Optional: Base URL for viewer (default: 'https://area-viewer.pages.dev')
  VIEWER_BASE: 'https://your-viewer.example.com',
  
  // Optional: Local worker base for preview links
  EDITOR_WORKER_BASE: 'http://localhost:8787'
};
```

## Migration Notes for Server Operators

If you're currently running a backend that receives published scenes:

### No Immediate Changes Required

The current changes maintain full backward compatibility. Your existing server will continue to work without modifications.

### Recommended Future Enhancements

1. **Return `viewerUrl` in response:** Gives you full control over the viewer URL format
2. **Consider explicit FormData field names:** Update your server to accept `glb`, `poster`, `audio`, `sceneJson` instead of multiple `file` entries
3. **Validate scene IDs:** Ensure scene IDs are properly sanitized and unique
4. **Support relative `viewerUrl`:** The editor will normalize them for you

### Testing Different Configurations

Test your server with:

1. **Absolute PUBLISH_ENDPOINT:** `https://example.com/api/publish`
   - Verify workerOrigin is extracted correctly
   - Check that scene URLs use the correct base

2. **Relative PUBLISH_ENDPOINT:** `/api/publish`
   - Verify it resolves correctly against current page location
   - Test from different paths (root, subfolders)

3. **Server returns absolute viewerUrl:** `https://viewer.example.com/webxr.html?scene=test`
   - Should be used as-is
   - Check that base parameter is validated

4. **Server returns relative viewerUrl:** `/webxr.html?scene=test`
   - Should be normalized using VIEWER_BASE
   - Final URL should be absolute

5. **Server returns no viewerUrl:**
   - Editor should construct URL using VIEWER_BASE + mode-specific entry point
   - Base parameter should be set to workerOrigin

6. **Test with local preview modes:**
   - Set anchorMode to 'editor-local' or 'local'
   - Configure EDITOR_WORKER_BASE
   - Verify preview marker is added to URLs
   - Check that workerOrigin uses EDITOR_WORKER_BASE

## Anchor Modes

The editor supports multiple anchor modes:

- **surface-webxr**: WebXR surface detection (default)
- **native**: Native AR with GLB and optional USDZ URLs
- **image**: Image target tracking
- **scene-viewer**: Google Scene Viewer mode
- **editor-local** or **local**: Local preview modes (use with EDITOR_WORKER_BASE)

Each mode uses a different viewer entry point:

| Mode | Viewer Entry Point |
|------|-------------------|
| surface-webxr | `/webxr.html` |
| native | `/index.html` |
| image | `/image-ar/viewer.html` |
| scene-viewer | `/scene-viewer/index.html` |

## Troubleshooting

**Q: My publish fails with "Publish ist nicht konfiguriert"**
- Ensure `window.__AREA.PUBLISH_ENDPOINT` is set before calling `publish()`

**Q: The base parameter in my viewer URL is wrong**
- Check that PUBLISH_ENDPOINT is correctly configured
- Verify your server isn't returning an invalid base in viewerUrl
- Check browser console for URL parsing errors

**Q: Preview URLs don't work**
- Ensure EDITOR_WORKER_BASE is set when using local preview modes
- Check that your local worker is running and accessible
- Verify CORS settings if calling across origins

**Q: Server returns 404 for scene assets**
- Verify the base parameter in the viewer URL matches where assets are stored
- Check that sceneId is properly URL-encoded
- Ensure your server serves assets at `/scenes/{sceneId}/` path structure

## License

[Your license here]
