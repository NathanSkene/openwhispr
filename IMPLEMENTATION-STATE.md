# Recording Overlay Implementation State

## Branch: `feat/recording-overlay`

## Completed
1. **`src/components/RecordingOverlay.tsx`** - Created. Full React component with canvas waveform, rolling buffer, IPC listener, cancel button.
2. **`src/components/RecordingOverlay.css`** - Created. Pill shape, dark/light mode, animations, spinner.
3. **`src/helpers/windowConfig.js`** - Modified. Added `RECORDING_OVERLAY_CONFIG` and `WindowPositionUtil.getRecordingOverlayPosition()`.
4. **`src/helpers/windowManager.js`** - Modified. Added import, constructor properties, and all methods: `createRecordingOverlay()`, `showRecordingOverlay()`, `hideRecordingOverlay()`, `sendRecordingOverlayUpdate()`, `setRecordingOverlayEnabled()`. Added `showRecordingOverlay()` calls in `sendStartDictation()` and `sendToggleDictation()`.
5. **`main.js`** - Modified. Added IPC handlers (`recording-overlay-update`, `recording-overlay-cancel`, `recording-overlay-enabled-changed`). Added `createRecordingOverlay()` call at app startup.
6. **`preload.js`** - Modified. Added `sendRecordingOverlayUpdate`, `onRecordingOverlayUpdate`, `cancelRecordingFromOverlay`, `notifyRecordingOverlayEnabledChanged` to electronAPI.
7. **`src/main.jsx`** - Modified. Added import of RecordingOverlay and route for `?recording-overlay=true`.
8. **`src/hooks/useAudioRecording.js`** - Modified. Added `sendRecordingOverlayUpdate` in `onStateChange` callback and `onAudioLevel` callback in `setCallbacks`.
9. **`src/helpers/audioManager.js`** - Modified. Added `onAudioLevel` to constructor, `setCallbacks()`, cleanup, and silence detection interval.

## Remaining (in order)

### 10. `src/stores/settingsStore.ts`
- Add `"showRecordingOverlay"` to SYNCED_SETTINGS array (after `"floatingIconAutoHide"`)
- Add `showRecordingOverlay: boolean;` to Settings interface (after `floatingIconAutoHide`)
- Add default: `showRecordingOverlay: readBoolean("showRecordingOverlay", true),` (after `floatingIconAutoHide` default)
- Add setter (after `setFloatingIconAutoHide`):
```ts
  setShowRecordingOverlay: (enabled: boolean) => {
    if (get().showRecordingOverlay === enabled) return;
    if (isBrowser) localStorage.setItem("showRecordingOverlay", String(enabled));
    set({ showRecordingOverlay: enabled });
    if (isBrowser) {
      window.electronAPI?.notifyRecordingOverlayEnabledChanged?.(enabled);
    }
  },
```

### 11. `src/components/SettingsPage.tsx`
Find the floatingIconAutoHide toggle and add after it:
```tsx
<SettingsPanelRow>
  <SettingsRow
    label="Show recording overlay"
    description="Show a floating overlay with audio waveform while recording"
  >
    <Toggle checked={showRecordingOverlay} onChange={setShowRecordingOverlay} />
  </SettingsRow>
</SettingsPanelRow>
```
Extract `showRecordingOverlay` and `setShowRecordingOverlay` from the settings store.

## Key Architecture Decisions
- Separate BrowserWindow (like MeetingNotificationOverlay)
- Audio levels forwarded: renderer → IPC → main → IPC → overlay
- RMS from existing silence detection (~100ms interval) in audioManager.js
- `focusable: false` prevents focus stealing
- Canvas waveform with 40-value rolling buffer
- `onAudioLevel` callback added to AudioManager for real-time level forwarding
