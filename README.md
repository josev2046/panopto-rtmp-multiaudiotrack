# Panopto Multi-Language Live Streaming Solution

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.17788084.svg)](https://doi.org/10.5281/zenodo.17788084)

This repository documents a client-side stream switching solution designed to support multiple simultaneous audio tracks (languages) for a single high-profile event.

As Panopto’s native architecture currently supports a single audio track per video stream, this solution wraps standard RTMP ingestion in a customised HTML interface. This centralises multiple language streams into a single URL, facilitating dynamic player switching without disrupting the viewer experience via page reloads.

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



https://github.com/user-attachments/assets/a06daff0-dc8f-4081-aa22-b9bf7ecdec2a


* The viewer navigates to the custom event URL.
* The page initialises; JavaScript injects the Session ID for the default language (English).
* The viewer selects "Spanish" via the dropdown menu.
* The script updates the iframe `src` attribute.
* **Outcome:** The player buffers briefly (approximately 1-2 seconds) and resumes playback with the Spanish audio track.

## Future enhancements: locale-based stream initialisation

At present, the player wrapper defaults to a static primary language (e.g., English) upon loading. A proposed enhancement involves implementing logic to automatically detect the viewer's locale, reducing the need for manual selection by the end-user.

This feature would function by querying the browser's `navigator.language` property to ascertain the user's preferred system language. The script would then parse this string to identify the primary language ISO code (for instance, truncating `es-MX` to `es`).

This code is subsequently cross-referenced against the internal `streamMap` object:

* **Match found:** If the detected locale corresponds to an available stream, that specific Panopto Session ID is injected into the iframe source immediately.
* **No match (fallback):** If the locale is unsupported, the system reverts to the standard default language to ensure the broadcast remains accessible.


To deploy this functionality, the static initialisation script within the HTML wrapper would be augmented with conditional logic:

```javascript
// 1. Define the default fallback (e.g., English)
const defaultLang = 'en';

// 2. Detect and normalize browser language (e.g., 'es-MX' becomes 'es')
const userLocale = (navigator.language || navigator.userLanguage).split('-')[0].toLowerCase();

// 3. Determine the startup ID
// If the detected language exists in our map, use it. Otherwise, use default.
const initialId = streamMap.hasOwnProperty(userLocale) 
    ? streamMap[userLocale] 
    : streamMap[defaultLang];

// 4. Initialize Player
document.getElementById('panopto-player-frame').src = 
    `https://{site}/Panopto/Pages/Embed.aspx?id=${initialId}&autoplay=true`;
```
## Other notes (as per peer review)

* While `autoplay=true` is included, browser autoplay policies may still require user interaction. Consider adding a fallback message or play button overlay.
  
* Maybe worth adding a validation for invalid language keys or missing GUIDs:

```javascript
function getStreamURL(languageKey) {
   const guid = streamMap[languageKey];
   if (!guid) {
       console.error(`No GUID found for language: ${languageKey}`);   
       return null;
   }
   return `${baseURL}?id=${guid}&autoplay=true${baseParams}`;
}
```

* Consider adding a loading indicator during stream switches to improve UX.
  
* Add aria-label attributes to the iframe and select element for accessibility if needed.

  
* The `offerviewer=true` parameter seems counterintuitive - typically you'd want `offerviewer=false` to hide the viewer pane for a cleaner interface.

I have reflected all those changes in the `stream-locale-switcher` HTML in this repository.
