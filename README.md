```markdown
# CHOICE TV — Single-file Live + VOD (index.html) + Relay Server

What I delivered
- index.html: a single-file client that:
  - plays VOD (HLS/MP4) and shows schedule-based "Now Playing" using Firestore.
  - allows admin sign-in (Google) and VOD uploads to Firebase Storage.
  - provides a webcam preview and "Go Live" feature that captures camera & mic in the browser and streams to a relay server via WebSocket using MediaRecorder.
  - shares a live link to social platforms (Twitter, Facebook, WhatsApp) — these links share the page URL; in production you should point them to a public player URL (HLS or viewer page).

- Relay server blueprint (see server files below) — a small Node.js WebSocket receiver that uses ffmpeg to consume binary chunks from the browser and publish:
  - to an RTMP destination (YouTube/Facebook/Twitch) if provided, and/or
  - generate an HLS file or push to a CDN for viewer playback; the server returns the HLS viewer URL to the client after publishing.

Why a relay server is required
- Browsers cannot directly speak RTMP. To stream to social platforms (YouTube/Twitch/Facebook), you need a relay/transcoding server that accepts browser media (MediaRecorder chunks or WebRTC) and feeds ffmpeg to push to RTMP endpoints or to create HLS segments for viewers.
- The client code provided uses MediaRecorder and WebSocket to stream WebM chunks to the relay server. The relay server uses ffmpeg to transcode WebM to an RTMP/HLS stream.

Files included (client + server blueprint)
- index.html (single-file web app)
- server/server.js (Node.js WebSocket relay)
- server/package.json
- server/Dockerfile

Server overview (server/server.js)
- Accepts WebSocket connections at path `/live`.
- First message from client should be a JSON: { action: 'start', rtmp: 'rtmp://...' } (rtmp optional).
- Binary messages are ffmpeg-friendly chunks (we send webm chunks).
- Server spawns ffmpeg that reads stdin and pushes to RTMP (and optionally writes HLS to a public folder).
- Server sends back JSON messages like { type: 'status', msg: '...' } and { type: 'hls', url: 'https://...' } when HLS becomes available.

Security & production notes
- Protect the relay endpoint with a token or authenticated session. The server prototype includes an optional query token check.
- Use HTTPS/WSS with a valid certificate (Let's Encrypt) for security and to allow camera/mic in modern browsers.
- Scale: For many concurrent broadcasters or viewers, use an autoscaled infrastructure, a CDN for HLS, and proper transcoding pipelines.
- Use existing managed services if you want simpler ops: Mux, Ant Media, Agora, Livepeer, Wowza, AWS IVS, or Cloudflare Stream.

How to deploy the relay server (quick)
1. Copy server files into a directory and adjust environment variables (RTMP forwarding defaults).
2. Have ffmpeg installed on the server.
3. Start server: `npm install && node server.js`
4. Expose `wss://your-relay.example.com/live` (use reverse proxy with TLS).
5. The client should connect to that WebSocket URL (fill the Relay WebSocket field in the index.html UI).
6. Viewer HLS URL (if server is configured to write HLS publicly) will be returned by the server once live; share that URL.

If you'd like I can:
- Provide the server files (server.js, Dockerfile, package.json) in this repo (they are prepared below).
- Produce a version that runs on Google Cloud Run (Cloud Run requires different approach to long-lived websockets; I can convert to use HTTP chunking or run on a VM).
- Add authentication to the WebSocket handshake (JWT or Firebase token verification).
- Create a complete deployment guide for a VPS (Ubuntu, Nginx reverse proxy, Let's Encrypt) or a Docker Compose setup.

What's next
- Deploy the relay server and configure a public viewer HLS URL, then enter its WebSocket URL into the index.html Go Live UI and test broadcasting from your browser.
- If you want I will also add a server endpoint that registers the current live stream in Firestore (creates an `assets` document of type `live` with the HLS viewer URL) so the main site can automatically show the live stream in "Now Playing".
