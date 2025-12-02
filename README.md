# Panopto Multi-Language Live Streaming Solution

This repository documents a Client-Side Stream Switching solution designed to support multiple simultaneous audio tracks (languages) for a single high-profile event.

Current Panopto native webcasting architecture supports a single audio track per video stream. To circumvent this limitation, this solution utilises standard RTMP ingestion whilst wrapping the viewer experience in a customised HTML interface. This requires centralising multiple language streams into a single URL to facilitate dynamic player switching without requiring page reloads.

## Architecture Overview

This solution bridges the gap between Panopto's standard delivery mechanisms and the client's multi-track requirement via a lightweight web wrapper. 

<img width="627" height="824" alt="image" src="https://github.com/user-attachments/assets/7a25b33e-67b3-4364-93ff-e96dcfb72fe4" />



It consists of three core components:

### Part 1: Ingest and Session Provisioning
Panopto natively supports unlimited concurrent webcasts.

* **Provisioning:** Distinct "Webcast" sessions are created for each language (e.g., `Event_EN`, `Event_ES`).
* **Routing:** Specific RTMP URLs and Stream Keys are generated for each session.
* **Synchronisation:** Synchronisation relies on the encoders starting simultaneously. Slight variances in latency (milliseconds) are expected but are negligible for this use case.

### Part 2: The Custom Player Wrapper
Panopto provides an Embed API and standard iframe code (`Embed.aspx`).
The native player interface lacks a control mechanism to switch between Session IDs dynamically.

* Deployment of a standalone HTML page hosting a Mapping Object (Language Name to Panopto Session GUID).
* Implementation of a simplified dropdown UI for language selection.
* Targeting of a single iframe element (`id="panopto-player-frame"`) to load the default language upon page initialisation.

### Part 3: Stream Switching Logic
Standard browser behaviour triggers a full page refresh when changing sources, disrupting the viewing experience. The player hot-swaps the Session ID and enforces autoplay to minimize interruption.

* **Hot Swapping:** JavaScript logic intercepts the dropdown change event to retrieve the corresponding Session ID.
* **URL Construction:** The script dynamically constructs the new source URL: `https://{site}/Panopto/Pages/Embed.aspx?id={New_GUID}&autoplay=true`.
* **Autoplay Enforcement:** The `autoplay=true` parameter is appended to ensure playback resumes immediately after the swap.

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
