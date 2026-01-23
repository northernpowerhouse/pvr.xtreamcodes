# Dispatcharr PVR Addon Implementation Plan

This plan details the steps to convert `pvr.xtreamcodes` into `pvr.dispatcharr` and implement recording management using the Dispatcharr API.

## 1. Renaming and Branding (User-Facing Only)
We will update the addon identity and folder structure, but keep internal source filenames intact to simplify migration.

- [ ] **Rename Addon ID & Name**: Update `pvr.xtreamcodes/addon.xml.in` to use `pvr.dispatcharr` as the ID and "Dispatcharr PVR Client" as the name.
- [ ] **Folder Structure**: Rename `pvr.xtreamcodes/` directory to `pvr.dispatcharr/`.
- [ ] **CMake Update**: Update `CMakeLists.txt` to point to the new folder location and set the new project ID.
- [ ] **Settings XML**: Update the section ID in `settings.xml` from `pvr.xtreamcodes` to `pvr.dispatcharr`.
- [ ] **Visible Strings**: scan `addon.cpp` and `xtream_client.cpp` for user-visible strings (logs, error messages) referencing "Xtream" and update them to "Dispatcharr".

## 2. Settings Management
We need a separate login for the Dispatcharr API, as it might differ from the Xtream Codes playback credentials, or at least be semantic.

- [ ] **Add Dispatcharr Settings**: Edit `resources/settings.xml` to add:
    - `dispatcharr_password`: New password field specifically for API operations (if different, or just reusing `password` but clarifying usage).
    - `api_request_timeout`: Optional separate timeout for management API.
- [ ] **Update Settings Struct**: Update `Settings` struct in `xtream_client.h` to hold the new password and API configuration.

## 3. Dispatcharr API Client
We need to extend the HTTP client to handle the JSON-based Dispatcharr API for recordings. Since this functionality is distinct from the Xtream Codes playback/EPG logic, we will create a dedicated client.

- [ ] **Create Client Files**: Create `src/dispatcharr_client.h` and `src/dispatcharr_client.cpp`.
- [ ] **Json Parser**: Use simple string formatting/parsing or `kodi::tools::Json` if available for the JSON-based API.
- [ ] **Implement DVR Methods** in `dispatcharr_client`:
    - `FetchRecordingRules()`: GET `/api/channels/series-rules/`
    - `AddSeriesRule(tvg_id, title, mode)`: POST `/api/channels/series-rules/`
    - `DeleteSeriesRule(tvg_id)`: DELETE `/api/channels/series-rules/{tvg_id}/`
    - `FetchRecordings()`: GET `/api/channels/recordings/`
    - `InternalDeleteRecording(id)`: DELETE `/api/channels/recordings/{id}/`
    - `FetchRecurringRules()`: GET `/api/channels/recurring-rules/`
    - `AddRecurringRule(...)`: POST `/api/channels/recurring-rules/`
- [ ] ** Integration**: Update `addon.cpp` to include `dispatcharr_client.h` and instantiate the client using the new settings.


## 4. PVR Recording Implementation
Map Kodi PVR callbacks to the client methods.

- [ ] **GetRecordings**: Implement `GetRecordings` in `addon.cpp`. Mapping:
    - Iterate response from `FetchRecordings()`.
    - Map fields: `title` -> `strTitle`, `start_time` -> `startTime`, `file_path` -> `strStreamURL`.
- [ ] **DeleteRecording**: Implement `DeleteRecording`.
    - Call `InternalDeleteRecording(recording.strRecordingId)`.

## 5. PVR Timer Implementation
Map detailed rules to Kodi Timers.

- [ ] **Timer Types**: Support two types of timers:
    1.  **Series Recording (Season Pass)**: Map to Kodi "Series Recording".
        -   Use `PVR_TIMER_TYPE_IS_REPEATING`? Or `PVR_TIMER_TYPE_FOR_SERIES`.
        -   When creating a timer for a show, use POST `/api/channels/series-rules/` with `tvg_id`.
    2.  **Recurring Manual Rules**: Kodi "Repeating Timer".
        -   Map to `/api/channels/recurring-rules/`.
    3.  **One-off Recording**:
        -   Map to `POST /api/channels/recordings/` (Manual schedule).

- [ ] **GetTimers**: Fetch both `series-rules` and `recurring-rules` and generic `recordings` (that are future scheduled) and merge them into the Kodi timer list.
    -   *Challenge*: Kodi distinguishes between "Timers" (rules) and "Recordings" (completed/in-progress files). Dispatcharr seems to treat "upcoming recordings" as individual `Recordings` too.
    -   *Strategy*: 
        -   `GetRecordings`: Only return items with `start_time < now`.
        -   `GetTimers`: Return items with `start_time > now` AND all "Rules".

- [ ] **AddTimer**: Switch logic based on `timer.type`.
    - If "Series": Call `AddSeriesRule`.
    - If "Repeating": Call `AddRecurringRule`.
    - Else (One-shot): Call `ScheduleRecording`.

- [ ] **DeleteTimer**:
    - Identify if it's a Rule (by prefix or ID format) or a Scheduled item.
    - Call appropriate Delete method.

## 6. Execution Plan
1.  **Preparation**: Rename files and folders.
2.  **Settings**: Update XML and C++ struct.
3.  **Client Plumbing**: Add JSON helper and networking for new API.
4.  **PVR Features**: Implement the Timer/Recording functions in `addon.cpp`.
5.  **Build & Verify**: Compile.

I will start by renaming the project structure.
