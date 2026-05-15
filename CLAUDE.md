# CLAUDE.md — wysiwyg-editor-node-sdk

Node.js backend SDK for [Froala WYSIWYG Editor](https://froala.com/wysiwyg-editor/). Handles server-side file/image/video upload, deletion, listing, AWS S3 signature generation, and WebSocket-based collaborative editing.

**Primary consumer:** `../real-app`

---

## Backend Services Connected

| Service | Protocol | Purpose |
|---------|----------|---------|
| AWS S3 | HTTPS (Signature v4) | Cloud file/image/video storage |
| Local filesystem | Node.js fs | On-disk file storage with optional ImageMagick resize |
| WebSocket relay | `ws://` | Real-time collaborative editing (room-based) |

---

## Folder Structure

```
wysiwyg-editor-node-sdk/
├── lib/
│   ├── froalaEditor.js        # Main entry point — merges and re-exports all modules
│   ├── image.js               # Image upload / delete / list
│   ├── file.js                # File upload / delete
│   ├── video.js               # Video upload / delete
│   ├── s3.js                  # AWS S3 Signature v4 hash generation
│   ├── collaborative.js       # WebSocket relay for collaborative editing
│   └── utils/
│       ├── disk.management.js # Busboy streaming, file I/O, ImageMagick resize
│       └── utils.js           # Extension extraction, MIME/ext validation
├── examples/
│   ├── server.js              # Express server demoing every endpoint
│   └── index.html             # Browser client with Froala Editor wired up
└── package.json               # main: "lib/froalaEditor.js", version: 5.0.1
```

---

## Main Entry Point

```js
const FroalaEditor = require('wysiwyg-editor-node-sdk');
// exposes: FroalaEditor.Image, .File, .Video, .S3, .Collaborative
```

---

## Module API Reference

### `FroalaEditor.Image`

```js
Image.upload(req, fileRoute, [options], callback)
// callback(err, { link: '/uploads/abc123.jpg' })

Image.delete(src, callback)
// callback(err)  — src is the relative path returned from upload

Image.list(folderPath, [thumbPath], callback)
// callback(err, [{ url, thumb, tag }])
```

Default allowed extensions: `gif jpeg jpg png svg blob`
Default MIME types: `image/gif image/jpeg image/pjpeg image/x-png image/png image/svg+xml`

---

### `FroalaEditor.File`

```js
File.upload(req, fileRoute, [options], callback)
// callback(err, { link: '/uploads/abc123.pdf' })

File.delete(src, callback)
```

Default allowed extensions: `txt pdf doc`
Default MIME types: `text/plain application/msword application/pdf`

---

### `FroalaEditor.Video`

```js
Video.upload(req, fileRoute, [options], callback)
// callback(err, { link: '/uploads/abc123.mp4' })

Video.delete(src, callback)
```

Default allowed extensions: `mp4 webm ogg`
Default MIME types: `video/mp4 video/webm video/ogg`

---

### Upload `options` Object

```js
{
  fieldname: 'file',             // multipart field name (default: 'file')
  validation: {                  // OR pass a function (see below)
    allowedExts: ['jpg', 'png'],
    allowedMimeTypes: ['image/jpeg', 'image/png']
  },
  resize: [width, height]        // images/videos only — requires ImageMagick
}
```

**Custom validation function:**
```js
validation: function(filePath, mimetype, callback) {
  // callback(err, isValid: boolean)
}
```

---

### `FroalaEditor.S3`

```js
const hash = S3.getHash({
  bucket:    'my-bucket',
  region:    'us-east-1',        // or 's3' for us-east-1 legacy path
  keyStart:  'editor/',
  acl:       'public-read',
  accessKey: process.env.AWS_ACCESS_KEY,
  secretKey: process.env.AWS_SECRET_ACCESS_KEY
});
// Returns { bucket, region, keyStart, params: { acl, policy, x-amz-* } }
// Policy expires 5 minutes after generation — call this per-request
```

---

### `FroalaEditor.Collaborative`

```js
// Option A — standalone WebSocket server
const wss = Collaborative.createServer({ port: 1234 });

// Option B — attach to existing HTTP/Express server (shares port)
const server = http.createServer(app);
Collaborative.attachToServer(server);
server.listen(3000);

// Room name comes from the WebSocket URL path: ws://host:port/<roomName>

Collaborative.getStats();
// Returns { rooms: number, clients: number }
```

Heartbeat: ping every 30 s, drops client after 10 s without pong.

---

## Standard Express Endpoint Pattern

See [examples/server.js](examples/server.js) for the full reference implementation. Typical endpoint shape:

```js
const FroalaEditor = require('wysiwyg-editor-node-sdk');

app.post('/upload_image', (req, res) => {
  FroalaEditor.Image.upload(req, '/public/uploads/', (err, data) => {
    if (err) return res.status(400).json(err);
    res.json(data);                            // { link: '/public/uploads/...' }
  });
});

app.post('/delete_image', (req, res) => {
  FroalaEditor.Image.delete(req.body.src, (err) => {
    if (err) return res.status(400).json({ error: err });
    res.json({ deleted: true });
  });
});

app.get('/load_images', (req, res) => {
  FroalaEditor.Image.list('/public/uploads/', (err, data) => {
    if (err) return res.status(400).json(err);
    res.json(data);
  });
});

app.get('/get_amazon', (req, res) => {
  res.json(FroalaEditor.S3.getHash({
    bucket:    process.env.AWS_BUCKET,
    region:    process.env.AWS_REGION,
    keyStart:  process.env.AWS_KEY_START,
    acl:       process.env.AWS_ACL,
    accessKey: process.env.AWS_ACCESS_KEY,
    secretKey: process.env.AWS_SECRET_ACCESS_KEY
  }));
});
```

---

## Environment Variables

All are optional at SDK level; required at runtime only when using S3:

| Variable | Required for | Example |
|----------|-------------|---------|
| `AWS_BUCKET` | S3 uploads | `my-froala-bucket` |
| `AWS_REGION` | S3 uploads | `us-east-1` |
| `AWS_KEY_START` | S3 uploads | `editor/` |
| `AWS_ACL` | S3 uploads | `public-read` |
| `AWS_ACCESS_KEY` | S3 uploads | AWS Access Key ID |
| `AWS_SECRET_ACCESS_KEY` | S3 uploads | AWS Secret Access Key |

---

## Prerequisites

ImageMagick must be installed on the server for resize operations:

```bash
# Ubuntu/Debian
apt-get install imagemagick

# macOS
brew install imagemagick
```

---

## Local Linking to ../real-app

### Option A — `npm link` (symlink)

```bash
# In this repo
cd d:/Projects/wysiwyg-editor-node-sdk
npm link

# In the consuming app
cd d:/Projects/real-app
npm link wysiwyg-editor-node-sdk
```

### Option B — `file:` path in package.json (preferred for CI)

In `d:/Projects/real-app/package.json`:

```json
{
  "dependencies": {
    "wysiwyg-editor-node-sdk": "file:../wysiwyg-editor-node-sdk"
  }
}
```

Then run `npm install` in `real-app`. Changes to this SDK are picked up on the next `require()` without re-linking.

---

## Related Repos

- **`../real-app`** — primary consumer; wires these endpoints into its Express routes and Froala Editor frontend configuration.
