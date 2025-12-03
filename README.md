# Panopto Multi-Language Live Streaming Solution

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.17788084.svg)](https://doi.org/10.5281/zenodo.17788084)

This repository documents a client-side stream switching solution designed to support multiple simultaneous audio tracks (languages) for a single high-profile event.

As Panopto’s native architecture currently supports a single audio track per video stream, this solution wraps standard RTMP ingestion in a customised HTML interface. This centralises multiple language streams into a single URL, facilitating dynamic player switching without disrupting the viewer experience via page reloads.

## Architecture Overview

This solution bridges standard Panopto delivery with multi-track requirements via a lightweight web wrapper. Two distinct upstream topologies have been modelled for this implementation: a standard multi-encoder approach and an optimised enterprise approach using AWS Elemental Live.

### Topology A: Discrete Hardware Encoders (Standard Implementation)
In this configuration, separate physical encoding units are provisioned for each language. This is often utilised when legacy hardware is available or when language feeds originate from disparate physical locations.

<img width="627" height="824" alt="image" src="https://github.com/user-attachments/assets/8c769fdb-9675-4c5e-beb7-f2d293c537dd" />


* **Ingest:** Multiple encoders (e.g., Encoder 1 for English, Encoder 2 for Spanish) operate independently.
* **Routing:** Each encoder pushes to a unique Panopto Session GUID.
* **Synchronisation:** Alignment relies on the manual synchronisation of "Start" times. Minor latency variances (drift) are expected between languages.

### Topology B: AWS Elemental Live "Fan-Out" (Enterprise Implementation)
To ensure frame-accurate synchronisation and reduce hardware footprint, this topology utilises a single unified encoding appliance.

<img width="620" height="881" alt="image" src="https://github.com/user-attachments/assets/380d6001-90e6-401f-928b-b8efb60884d5" />


* **Ingest:** A single AWS Elemental Live appliance receives a master feed containing all audio tracks (e.g., SDI Embedded Audio Ch1-4).
* **Fan-Out:** The encoder processes the video signal once but generates unique RTMP streams for each language by mapping specific audio pairs to distinct Output Groups.
* **Synchronisation:** As all RTMP streams derive from a single system clock, the timecode is identical across all sessions. This minimises visual discontinuity when a user switches languages.

---

## The Client-Side Wrapper
Regardless of the upstream topology chosen, the client-side experience remains consistent. While Panopto provides an Embed API, the native interface lacks dynamic session switching. To address this:

1.  **Hosting:** A standalone HTML page hosts a Mapping Object (Language Name to Panopto Session GUID).
2.  **Interface:** A simplified dropdown UI facilitates language selection.
3.  **Initialisation:** The page targets a single iframe (`id="panopto-player-frame"`) to load the default language immediately.

### Stream Switching Logic
To prevent the full page refresh standard in browser behaviour, the wrapper employs the following logic:

* **Hot Swapping:** JavaScript intercepts the dropdown change event to retrieve the corresponding Session ID.
* **URL Construction:** The script dynamically constructs the new source URL using the specific session GUID.
* **Autoplay Enforcement:** The `autoplay=true` parameter is appended to ensure playback resumes immediately.

---

## Encoder Configuration Reference (AWS Elemental Live)

If utilising **Topology B**, the encoder must be configured to map specific source audio channels to the corresponding Panopto RTMP endpoints.

| Setting Category | Parameter | Value |
| :--- | :--- | :--- |
| **Input** | Audio Selector 1 | **Name:** `Audio_ENG`<br>**Source:** `Embedded`<br>**Track:** `1` (Pair 1) |
| **Input** | Audio Selector 2 | **Name:** `Audio_ESP`<br>**Source:** `Embedded`<br>**Track:** `2` (Pair 2) |
| **Output Group 1** | Destination | `rtmp://[Panopto-Ingest-URL]/[StreamKey-A]` |
| **Stream 1** | Audio Source | `Audio_ENG` |
| **Output Group 2** | Destination | `rtmp://[Panopto-Ingest-URL]/[StreamKey-B]` |
| **Stream 2** | Audio Source | `Audio_ESP` |

---

## User Journeys

### 1. AV Producer Journey (Setup)
* The AV team configures the upstream encoding environment (via Topology A or B).
* **English Feed:** Pushes to Panopto Session A.
* **Spanish Feed:** Pushes to Panopto Session B.
* **Outcome:** Two distinct, parallel broadcasts are active within the Panopto cloud environment.

### 2. Viewer Journey (Consumption)


https://github.com/user-attachments/assets/c36eb0dd-e939-4190-9e3c-4ab7871864aa


* The viewer navigates to the custom event URL.
* The page initialises; JavaScript injects the Session ID for the default language (English).
* The viewer selects "Spanish" via the dropdown menu.
* The script updates the iframe `src` attribute.
* **Outcome:** The player buffers briefly (approximately 1-2 seconds) and resumes playback with the Spanish audio track.

---

## Future Enhancements: Locale-Based Stream Initialisation

At present, the player wrapper defaults to a static primary language (e.g., English) upon loading. A proposed enhancement involves implementing logic to automatically detect the viewer's locale, reducing the need for manual selection by the end-user.

This feature functions by querying the browser's `navigator.language` property to ascertain the user's preferred system language. The script parses this string to identify the primary language ISO code (for instance, truncating `es-MX` to `es`).

This code is subsequently cross-referenced against the internal `streamMap` object:

* **Match found:** If the detected locale corresponds to an available stream, that specific Panopto Session ID is injected into the iframe source immediately.
* **No match (fallback):** If the locale is unsupported, the system reverts to the standard default language to ensure the broadcast remains accessible.

To deploy this functionality, the static initialisation script within the HTML wrapper would be augmented with conditional logic.

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

---

## Implementation Notes

The following considerations have been addressed following peer review:

* **Autoplay Policies:** While `autoplay=true` is included, browser autoplay policies may still require user interaction. A fallback message or play button overlay is recommended for production deployments.
* **Error Handling:** Validation logic should be implemented to handle invalid language keys or missing GUIDs.
* **User Experience:** Ideally, a loading indicator should be displayed during stream switches to mask the buffering period.
* **Accessibility:** `aria-label` attributes should be applied to the iframe and select elements.
* **Viewer Pane:** The parameter `offerviewer=false` should be utilised to conceal the viewer list, ensuring a cleaner interface.

These suggestions have been accommodated in the `stream-locale-switcher.HTML` file located in this repository.
