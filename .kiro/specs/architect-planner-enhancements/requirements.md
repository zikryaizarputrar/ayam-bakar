# Requirements Document

## Introduction

This document covers two new features for the 3D Architecture Planner web application (`index.html`). The app is a browser-based tool built with Three.js r128 and OrbitControls. Currently it allows users to add walls and rooms to a 3D scene and navigate the camera, but provides no way to reposition placed objects or save work between sessions.

The two features addressed here are:

1. **Movable Walls** — Users can click and drag any wall or room object on the 3D floor grid to reposition it.
2. **Google Login & User Data** — Users can sign in with a Google account to save designs to the cloud, reload saved designs in a future session, and view their activity history.

---

## Glossary

- **Planner**: The 3D Architecture Planner single-page web application.
- **Scene**: The Three.js 3D scene containing all architecture objects, the floor grid, and lighting.
- **Wall**: A Three.js `Mesh` created with `BoxGeometry(5, 2.5, 0.2)` that represents a wall in the Scene.
- **Room**: A Three.js `Mesh` created with `BoxGeometry(4, 3, 4)` and wireframe material that represents a room outline in the Scene.
- **Object**: A Wall or Room mesh in the Scene, stored in the `objects` array.
- **OrbitControls**: The Three.js orbit camera controller attached to the renderer's DOM element.
- **DragController**: The new component responsible for intercepting pointer events to move Objects on the floor plane while temporarily suspending OrbitControls.
- **Floor Plane**: The invisible `y = 0` horizontal plane used as the projection surface for drag calculations.
- **Raycaster**: A Three.js `Raycaster` used to compute intersections between the pointer ray and Scene objects or the Floor Plane.
- **Firebase**: The Google Firebase platform used to provide authentication and cloud data storage.
- **AuthService**: The component responsible for managing Google Sign-In and Sign-Out via Firebase Authentication.
- **DataService**: The component responsible for reading and writing user design data to Firebase Firestore.
- **Design**: A saved snapshot of the Scene's `objects` array, including each Object's type (Wall or Room), position, and rotation, stored as a Firestore document.
- **ActivityLog**: A Firestore sub-collection recording timestamped events (sign-in, design saved, design loaded, object added, scene cleared) for a given user.
- **User**: An authenticated person identified by a Google account UID.

---

## Requirements

### Requirement 1: Select an Object for Dragging

**User Story:** As a user, I want to click on a wall or room in the 3D scene so that I can indicate which object I intend to move.

#### Acceptance Criteria

1. WHEN the user presses the primary mouse button over a Wall or Room mesh, THE DragController SHALL cast a ray from the camera through the cursor position, intersect it against the `objects` array, and mark the nearest hit Object as the active drag target.
2. WHEN the user presses the primary mouse button over empty space (no Object is hit by the Raycaster), THE DragController SHALL NOT mark any drag target, and OrbitControls SHALL remain enabled and respond to the event normally.
3. WHEN an Object is marked as the active drag target, THE DragController SHALL store the Object's original material color, then change the material color to `0xffaa00`; this highlight SHALL persist until the drag ends.
4. WHEN an Object is marked as the active drag target, THE DragController SHALL set `OrbitControls.enabled = false` before the first `mousemove` event is processed.
5. WHEN the user presses the primary mouse button and a drag is already active, THE DragController SHALL ignore the new press without clearing or replacing the existing active drag target.

---

### Requirement 2: Drag an Object Across the Floor Grid

**User Story:** As a user, I want to drag a selected wall or room across the floor so that I can position it anywhere on the grid.

#### Acceptance Criteria

1. WHEN the drag operation begins (mouse button pressed on an Object), THE DragController SHALL record the offset between the Object's current `position.x`/`position.z` and the Floor Plane intersection point so that the Object does not jump to the cursor on the first move event.
2. WHILE an Object is the active drag target and the user moves the mouse, THE DragController SHALL project the pointer ray onto the Floor Plane (y = 0), apply the recorded offset, snap to the nearest 0.5-unit grid increment, clamp to [-14, 14] on both axes, and update the Object's `position.x` and `position.z` accordingly.
3. WHILE an Object is the active drag target, THE DragController SHALL keep the Object's `position.y` unchanged so that it remains resting on the floor (Walls at y = 1.25, Rooms at y = 1.5).
4. WHILE an Object is the active drag target, THE DragController SHALL snap the Object's `position.x` and `position.z` independently to the nearest multiple of 0.5 scene units using `Math.round(value / 0.5) * 0.5`.
5. WHILE an Object is the active drag target, THE DragController SHALL clamp the snapped `position.x` and `position.z` to the inclusive range [-14, 14] after snapping to keep the Object within the visible 30 × 30 floor grid.
6. IF the pointer ray does not intersect the Floor Plane during a `mousemove` event (e.g., ray is parallel to the plane), THEN THE DragController SHALL keep the Object at its last valid position and SHALL NOT update `position.x` or `position.z`.
7. WHEN the drag operation ends (mouse button released), THE DragController SHALL discard the stored offset so it does not affect the next drag operation.

---

### Requirement 3: Release a Dragged Object

**User Story:** As a user, I want to release the mouse button to drop a wall or room at its new position so that the drag operation ends cleanly.

#### Acceptance Criteria

1. WHEN the user releases the primary mouse button AND an Object is the active drag target, THE DragController SHALL clear the active drag target and finalize the Object's position at its current snapped, clamped location.
2. WHEN the active drag target is cleared, THE DragController SHALL restore the Object's material color to the value stored when the drag began (the original pre-drag color), not a hardcoded constant.
3. WHEN the active drag target is cleared, THE DragController SHALL set `OrbitControls.enabled = true` to restore camera navigation.
4. IF the pointer leaves the renderer's canvas element while an Object is the active drag target, THEN THE DragController SHALL execute the same teardown as criteria 1–3: clear the drag target, restore the material color, and re-enable OrbitControls.
5. WHEN the user releases the primary mouse button and no drag is currently active, THE DragController SHALL take no action.

---

### Requirement 4: Touch-Device Drag Support

**User Story:** As a user on a touch device, I want to drag walls and rooms with a single finger so that I can use the planner on a tablet or phone.

#### Acceptance Criteria

1. WHEN a single touch begins over a Wall or Room mesh, THE DragController SHALL normalize the touch coordinates from `touches[0]` to Normalized Device Coordinates (NDC) in the range [-1, 1] using the renderer canvas dimensions, then apply the same Raycaster hit-test logic as pointer-based selection to mark the Object as the active drag target.
2. WHILE an Object is being dragged with a single touch, THE DragController SHALL apply the same 0.5-unit snapping and [-14, 14] boundary clamping defined in Requirement 2 when updating the Object's `position.x` and `position.z` for each `touchmove` event.
3. WHEN two or more simultaneous touches are detected and an Object is the active drag target, THE DragController SHALL clear the active drag target and restore the Object's material color to its pre-drag value.
4. WHEN two or more simultaneous touches are detected, THE DragController SHALL set `OrbitControls.enabled = true` so that pinch-to-zoom and two-finger pan continue to function.
5. WHEN a `touchend` event fires and an Object is the active drag target, THE DragController SHALL apply the same teardown as pointer-based release: clear the drag target, restore material color, and re-enable OrbitControls.
6. WHEN a `touchend` event fires and no drag is currently active, THE DragController SHALL take no action.

---

### Requirement 5: Google Sign-In

**User Story:** As a user, I want to sign in with my Google account so that my designs are linked to my identity and can be saved and retrieved.

#### Acceptance Criteria

1. WHILE no User is authenticated, THE Planner SHALL display a "Sign in with Google" button in the UI panel.
2. WHEN the user clicks "Sign in with Google", THE AuthService SHALL initiate a Google OAuth popup flow using Firebase Authentication (`signInWithPopup` with `GoogleAuthProvider`).
3. WHEN the OAuth flow completes successfully, THE AuthService SHALL store the authenticated User's unique identifier, display name, and profile photo URL in application state.
4. WHEN the authenticated User's data is stored, THE Planner SHALL hide the "Sign in with Google" button and SHALL display the User's display name and profile photo in the UI panel; IF the profile photo URL is null or empty, THE Planner SHALL display a generic placeholder avatar instead.
5. IF the OAuth popup is closed or cancelled by the user without completing authentication, THEN THE Planner SHALL display a dismissible message "Sign-in cancelled" in the UI panel and SHALL remain in the unauthenticated state.
6. IF the OAuth flow fails due to a network or provider error, THEN THE Planner SHALL display a dismissible error message in the UI panel describing the failure and SHALL remain in the unauthenticated state.
7. WHEN the Planner initializes or the page is refreshed, THE AuthService SHALL restore the previously authenticated User's session from Firebase's browser-persisted local storage without requiring the user to sign in again; IF no persisted session exists, THE Planner SHALL display the "Sign in with Google" button.

---

### Requirement 6: Google Sign-Out

**User Story:** As a user, I want to sign out of my Google account so that my session is ended and another person can use the device.

#### Acceptance Criteria

1. WHILE a User is authenticated, THE Planner SHALL display a "Sign Out" button in the UI panel.
2. WHEN the user clicks "Sign Out", THE AuthService SHALL call the Firebase sign-out method to end the current authentication session.
3. WHEN sign-out completes successfully, THE Planner SHALL hide the User's display name, profile photo, and "Sign Out" button, and SHALL show the "Sign in with Google" button.
4. IF sign-out fails, THEN THE Planner SHALL display a dismissible error message in the UI panel and SHALL leave the authenticated User's session and profile display unchanged.

---

### Requirement 7: Save Design to Cloud

**User Story:** As a user, I want to save my current 3D scene to the cloud so that I can reload it in a future session.

#### Acceptance Criteria

1. IF a User is authenticated, THE Planner SHALL display a "Save Design" button in the UI panel; WHILE no User is authenticated, THE Planner SHALL hide the "Save Design" button.
2. IF the `objects` array is empty when the user clicks "Save Design", THEN THE Planner SHALL display an informational message "Nothing to save — add walls or rooms first" and SHALL NOT write to Firestore.
3. WHEN the user clicks "Save Design" and the Scene contains at least one Object, THE DataService SHALL serialize each Object's type (wall or room), world-space position (x, y, z), and rotation (x, y, z) into a structured array.
4. WHEN the serialized design is produced, THE DataService SHALL persist it to cloud storage under the authenticated User's account with an auto-generated unique identifier and an ISO-8601 timestamp recording when it was saved.
5. WHEN the cloud write succeeds, THE Planner SHALL display a non-blocking success notification "Design saved" that auto-dismisses after 3 seconds.
6. IF the cloud write fails, THEN THE Planner SHALL display a dismissible error message in the UI panel indicating the save failed; the local Scene SHALL remain unchanged.
7. WHEN a design is saved successfully, THE DataService SHALL record a "design saved" event in the User's ActivityLog; IF the ActivityLog write fails, THE DataService SHALL silently ignore that failure without surfacing an error to the user.

---

### Requirement 8: Load a Saved Design

**User Story:** As a user, I want to load a previously saved design so that I can continue working on it.

#### Acceptance Criteria

1. IF a User is authenticated, THE Planner SHALL display a "Load Design" button in the UI panel; WHILE no User is authenticated, THE Planner SHALL hide the "Load Design" button.
2. WHEN the user clicks "Load Design", THE DataService SHALL query the authenticated User's saved designs ordered by save time, newest first, retrieving at most 50 results.
3. WHEN the query returns one or more designs, THE Planner SHALL display a list showing each design's formatted save timestamp so the user can select one to load.
4. IF the query returns no results, THE Planner SHALL display the message "No saved designs found" and SHALL NOT proceed to a selection list.
5. WHEN the user selects a design from the list, THE DataService SHALL fetch the full design data from cloud storage.
6. WHEN the design data is fetched, THE Planner SHALL replace the current Scene contents with the Objects described in the design, preserving their saved positions and rotations; the resulting Scene SHALL contain exactly the objects present when that design was saved.
7. WHEN a design is loaded successfully, THE Planner SHALL display a non-blocking success notification "Design loaded" that auto-dismisses after 3 seconds.
8. IF the query or fetch fails, THEN THE Planner SHALL display a dismissible error message indicating the load failed; the current Scene SHALL remain unmodified.
9. WHEN a design is loaded successfully, THE DataService SHALL record a "design loaded" event in the User's ActivityLog; IF the ActivityLog write fails, THE DataService SHALL silently ignore that failure.

---

### Requirement 9: Activity History Display

**User Story:** As a user, I want to view my activity history so that I can see what designs I've saved or loaded in past sessions.

#### Acceptance Criteria

1. IF a User is authenticated, THE Planner SHALL display a "My Activity" button in the UI panel; WHILE no User is authenticated, THE Planner SHALL hide the "My Activity" button.
2. WHEN the user clicks "My Activity", THE DataService SHALL query the User's ActivityLog ordered by timestamp descending, retrieving at most 50 entries.
3. WHEN the query returns one or more entries, THE Planner SHALL display an overlay panel listing each entry with a human-readable event label and a locale-formatted date-time string (e.g., "Jul 8, 2026, 10:30 AM") for each entry's timestamp.
4. IF the query returns no entries, THE Planner SHALL display the message "No activity recorded yet" inside the overlay panel.
5. WHEN the activity overlay is open, THE Planner SHALL display a "Close" button; WHEN the user clicks "Close", THE Planner SHALL dismiss the overlay and restore the main UI.
6. IF the ActivityLog query fails, THEN THE Planner SHALL dismiss any partially-open overlay and display a dismissible error message in the UI panel describing the failure.

---

### Requirement 10: Automatic Activity Logging

**User Story:** As a user, I want key actions to be automatically recorded so that my activity history is complete without manual effort.

#### Acceptance Criteria

1. WHEN a User signs in successfully, THE AuthService SHALL write a "signed in" event with the current ISO-8601 timestamp to the User's ActivityLog.
2. WHEN an authenticated User adds a Wall to the Scene, THE DataService SHALL write an "object added — wall" event with the current ISO-8601 timestamp to the User's ActivityLog.
3. WHEN an authenticated User adds a Room to the Scene, THE DataService SHALL write an "object added — room" event with the current ISO-8601 timestamp to the User's ActivityLog.
4. WHEN an authenticated User clears the Scene, THE DataService SHALL write a "scene cleared" event with the current ISO-8601 timestamp to the User's ActivityLog.
5. THE DataService SHALL write each ActivityLog entry as a separate document with an auto-generated unique identifier so that concurrent writes do not overwrite each other.
6. IF an ActivityLog write fails for any event in criteria 1–4, THEN THE DataService SHALL silently ignore the failure without displaying an error message or blocking the user action that triggered the log.

---

### Requirement 11: Unauthenticated State Preservation

**User Story:** As a user, I want to use the planner without signing in so that I can explore the tool before committing to an account.

#### Acceptance Criteria

1. WHILE a User is not authenticated, THE Planner SHALL allow users to add Walls, add Rooms, drag Objects, and clear the Scene without restriction.
2. WHILE a User is not authenticated, THE Planner SHALL NOT display the "Save Design", "Load Design", or "My Activity" buttons.
3. WHILE a User is not authenticated, IF the DataService is triggered by any action, THEN THE DataService SHALL silently skip any Firestore read or write operation without displaying an error to the user.
