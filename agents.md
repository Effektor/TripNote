I have a single-file static web app (index.html) that I host on GitHub Pages. It reads a Google Maps export JSON file, shows a list of trips with date filters and checkboxes, and can export selected trips to CSV for Excel (körjournal).

The UI and behavior are already correct. The ONLY thing that is wrong is that the JavaScript expects the OLD Google Location History format with `timelineObjects` and `activitySegment`. My actual export is the NEW “Semantic Location History” format, which looks like the attached small.json.

Your task: update the JavaScript in index.html so that it correctly parses the new JSON structure, while keeping the rest of the app (HTML layout, filters, selection logic, CSV export) as unchanged as possible. Ideally it should also be backwards compatible with the old format, but the new format is the priority.

Details about the current code (approximate):
- There is a function `loadTripsFromData(data)` that currently does something like:
  - `const timelineObjects = data.timelineObjects || data.TimelineObjects || [];`
  - Loops over `timelineObjects`
  - For each object, reads `obj.activitySegment.duration.startTimestamp`, `duration.endTimestamp`, `activity.distance`, `startLocation`, `endLocation`, etc.
  - Builds an internal `trip` object with:
    - `id`
    - `startTimestamp`
    - `endTimestamp`
    - `dateOnly`
    - `startTime`
    - `endTime`
    - `distanceKm`
    - `startLat`, `startLng`, `endLat`, `endLng`
    - `startAddress`, `endAddress`
    - `selected`
  - Stores all trips in `allTrips` and then calls `applyFilter()`
- The filter, table rendering, and CSV export all operate on this `allTrips` array, assuming those fields exist. Those parts should remain the same.

NEW JSON FORMAT (Semantic Location History):
The root object has (among other things):

- `semanticSegments`: array
- `rawSignals`: array
- `userLocationProfile`: object

An example of the structure is in the attached small.json. The important parts for trips are in `semanticSegments`.

Typical `semanticSegments` elements look like these two patterns:

1) MOVEMENT / TRIP (has `activity`):
{
  "startTime": "2024-07-30T18:37:06.000+02:00",
  "endTime": "2024-07-30T18:47:06.000+02:00",
  "startTimeTimezoneUtcOffsetMinutes": 120,
  "endTimeTimezoneUtcOffsetMinutes": 120,
  "activity": {
    "start": {
      "latLng": "57.7178942°, 12.0144616°"
    },
    "end": {
      "latLng": "57.7178942°, 12.0144616°"
    },
    "distanceMeters": 0.0,
    "probability": ...,
    "topCandidate": {
      "type": "WALKING",
      "probability": ...
    }
  }
}

2) VISIT / STAY (has `visit`, no `activity`):
{
  "startTime": "2024-07-30T18:47:06.000+02:00",
  "endTime": "2024-07-31T23:30:43.000+02:00",
  "visit": {
    "hierarchyLevel": 0,
    "probability": ...,
    "topCandidate": {
      "placeId": "...",
      "semanticType": "HOME",  // may be HOME, WORK, etc.
      "placeLocation": {
        "latLng": "57.7179386°, 12.0143944°"
      }
    }
  }
}

There are also segments that only contain `timelinePath` with points:
{
  "startTime": "...",
  "endTime": "...",
  "timelinePath": [
    {
      "point": "57.7178942°, 12.0144616°",
      "time": "2024-07-30T18:47:00.000+02:00"
    },
    ...
  ]
}

For the körjournal we only care about segments that represent trips (movement). That is:
- `semanticSegments` elements that have an `activity` block.
- Ignore segments that only have `visit` or only `timelinePath` if there is no `activity`.

What I want the code to do:

1. Modify `loadTripsFromData(data)`:
   - It should first try the new format:
     - If `data.semanticSegments` exists and is an array, use that as the primary source.
     - For each segment in `semanticSegments`:
       - If `segment.activity` exists, treat that as a trip.
       - Extract:
         - `startTime` and `endTime` from `segment.startTime` / `segment.endTime` (these are ISO strings with timezone offset).
         - Distance in km from `segment.activity.distanceMeters` (if present), i.e. `distanceKm = distanceMeters / 1000`, formatted to 3 decimal places.
         - Start and end coordinates from:
           - `segment.activity.start.latLng`
           - `segment.activity.end.latLng`
           These are strings like `"57.7178942°, 12.0144616°"`.
           You can either:
             a) keep them as strings and put them directly into `startLat` / `startLng` and `endLat` / `endLng` fields, OR
             b) parse them into numeric latitude and longitude:
                - split on comma
                - strip whitespace and trailing "°"
         - For `startAddress` and `endAddress`:
           - For now, just leave them as empty strings (`""`), since this export doesn’t provide direct addresses in the segment.
       - Compute:
         - `dateOnly` from `startTime` using local time (YYYY-MM-DD).
         - `startTime` (time-of-day HH:MM) from `segment.startTime`.
         - `endTime` (time-of-day HH:MM) from `segment.endTime`.
       - Create a `trip` object with the same shape the rest of the code expects:
         {
           id: <incrementing integer>,
           startTimestamp: <original segment.startTime>,
           endTimestamp: <original segment.endTime>,
           dateOnly: <YYYY-MM-DD derived from startTime>,
           startTime: <HH:MM from startTime>,
           endTime: <HH:MM from endTime>,
           distanceKm: <string with 3 decimals, or "" if not available>,
           startLat: <parsed or raw lat value>,
           startLng: <parsed or raw lng value>,
           endLat: <parsed or raw lat value>,
           endLng: <parsed or raw lng value>,
           startAddress: "",
           endAddress: "",
           selected: true
         }
   - If `data.semanticSegments` does not exist, fall back to the current old logic using `timelineObjects` and `activitySegment`, so the page still works with old exports.

2. Date filtering:
   - The existing code calculates `dateOnly`, `startTime`, `endTime` using `parseDateOnlyFromIso()` and `parseTimeOnlyFromIso()` on `startTimestamp` and `endTimestamp`.
   - Keep those helpers, but now they should work with the new `startTimestamp`/`endTimestamp` fields (which come directly from `segment.startTime` / `segment.endTime`).
   - Make sure Date parsing works correctly with the ISO strings that include a timezone offset like `"2024-07-30T18:37:06.000+02:00"`.

3. CSV export:
   - Keep the existing headers and logic, but the data source is now the `trip` objects derived from `semanticSegments`.
   - The headers are currently something like:
     ["StartTime", "EndTime", "Distance_km", "StartLat", "StartLng", "EndLat", "EndLng", "StartAddress", "EndAddress"]
   - For new-format data, fill them from the `trip` object fields as described above.
   - It is OK if `StartAddress` and `EndAddress` are empty for now.

4. Ignore:
   - `rawSignals`
   - `userLocationProfile.frequentPlaces`, `frequentTrips`, `persona`, etc.
   These are not needed for the körjournal functionality.

5. Keep the UI exactly as it is:
   - File upload
   - From/To date filter
   - Table of trips with checkboxes
   - “Markera synliga” / “Avmarkera synliga”
   - “Exportera valda till CSV”
   - Summary bar with counts and total distance

6. Once done, the app should:
   - Successfully load the attached small.json (Semantic Location History format).
   - Show one row per `semanticSegments` entry that has an `activity` object.
   - Allow filtering by date and selecting trips.
   - Export the selected trips to CSV with correct StartTime, EndTime, and distance in km.

Please modify the existing JavaScript in index.html to meet the above requirements, with minimal changes to the HTML and existing UI logic.
