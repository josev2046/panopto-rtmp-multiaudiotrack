# Panopto Multi-Language Live Streaming Solution

This repository documents a client-side stream switching solution designed to support multiple simultaneous audio tracks (languages) for a single high-profile event.

As Panopto’s native architecture supports a single audio track per video stream, this solution wraps standard RTMP ingestion in a customised HTML interface. This centralises multiple language streams into a single URL, facilitating dynamic player switching without disrupting the viewer experience via page reloads.

## Architecture Overview

This solution bridges standard Panopto delivery with multi-track requirements via a lightweight web wrapper.

<img width="627" height="824" alt="Architecture Diagram" src="https://github.com/user-attachments/assets/7a25b33e-67b3-4364-93ff-e96dcfb72fe4" />

The architecture comprises three core components:

### 1. Ingest and Session Provisioning
Panopto natively supports unlimited concurrent webcasts. This solution leverages that capability as follows:

* **Provisioning:** Distinct "Webcast" sessions are created for each language (e.g., `Event_EN`, `Event_ES`).
* **Routing:** Unique RTMP URLs and Stream Keys are generated for each session.
* **Synchronisation:** Alignment relies on simultaneous encoder start times. Minor latency variances (milliseconds) are expected but negligible for this use case.

### 2. The Custom Player Wrapper
While Panopto provides an Embed API, the native interface lacks dynamic session switching. To address this:

* **Hosting:** A standalone HTML page hosts a Mapping Object (Language Name to Panopto Session GUID).
* **Interface:** A simplified dropdown UI facilitates language selection.
* **Initialisation:** The page targets a single iframe (`id="panopto-player-frame"`) to load the default language immediately.

### 3. Stream Switching Logic
To prevent the full page refresh standard in browser behaviour, the wrapper employs the following logic:

* **Hot Swapping:** JavaScript intercepts the dropdown change event to retrieve the corresponding Session ID.
* **URL Construction:** The script dynamically constructs the new source URL: `https://{site}/Panopto/Pages/Embed.aspx?id={New_GUID}&autoplay=true`.
* **Autoplay Enforcement:** The `autoplay=true` parameter is appended to ensure playback resumes immediately.

## User Journeys

### 1. AV Producer Journey (Setup)
* The AV team configures the upstream encoders.
* **Encoder A (English)** pushes to Panopto Session A.
* **Encoder B (Spanish)** pushes to Panopto Session B.
* **Outcome:** Two distinct, parallel broadcasts are active within the Panopto cloud environment.

### 2. Viewer Journey (Consumption)
* The viewer navigates to the custom event URL.
* The page initialises; JavaScript injects the Session ID for the default language (English).
* The viewer selects "Spanish" via the dropdown menu.
* The script updates the iframe `src` attribute.
* **Outcome:** The player buffers briefly (approximately 1-2 seconds) and resumes playback with the Spanish audio track.
