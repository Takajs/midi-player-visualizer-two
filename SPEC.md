# MIDI Player Visualizer - Complete Specification

**Project Purpose**: A web-based MIDI player with real-time piano-roll style animations showing notes falling toward keys as they play.

**Version**: 1.0  
**Status**: Specification Complete  
**Target Browsers**: Modern standard web browsers (Chrome, Firefox, Safari, Edge)

---

## 1. Core Requirements

### 1.1 Audio Playback
- **Input**: User uploads .midi/.mid files (max 2MB assumed)
- **Synthesis**: Use Web Audio API with Tone.js or similar MIDI synthesizer library to generate audio in real-time
- **Playback**: Audio plays synchronized with visualization
- **Justification**: Pure MIDI synthesis ensures perfect audio-animation synchronization and enables seeking without complex audio file management

### 1.2 Visualization Style
- **Primary Mode**: Piano-roll style falling notes (no additional visualization modes)
- **Note Representation**: Animated rectangles falling vertically
- **Duration Representation**: Rectangle height is proportional to note duration
- **Color Scheme**: 
  - Each MIDI channel gets a base color
  - Within a channel, note brightness varies by pitch: lower notes darker, higher notes lighter
  - Create perceptual brightness gradient within the same hue
  - Example: Channel 1 = blue, C3 = dim blue, C7 = bright light blue

### 1.3 Keyboard Display
- **Range**: Full MIDI keyboard (128 notes, C-1 to G9)
- **Layout**: Horizontal white/black piano keys (standard piano layout)
- **Width**: Stretch to fill full screen width (full responsive)
- **Keyboard Behavior**: 
  - Shows the actual keyboard at the bottom of the screen
  - Each falling note rectangle aligns with its corresponding key
  - Note plays when rectangle hits the key
  - Note "shines" (highlight/glow effect) when actively playing

### 1.4 Note Animation & Timing
- **Falling Distance**: 3-5 seconds of lookahead visible vertically (user can configure or default to 4 seconds)
- **Falling Speed**: Calculated so notes reach the keyboard exactly at their scheduled play time
- **Visual Persistence**: Note rectangle persists and highlights while the sound is playing
- **Fade Out**: 100ms smooth fade (transparency transition) when note end event occurs
- **Performance Target**: 60 FPS animation, acceptable to drop frames during very heavy polyphony (20+ simultaneous notes)

---

## 2. User Interface

### 2.1 Layout
```
┌─────────────────────────────────────────┐
│  [File Upload]  [Play] [Pause]          │
│  Time: MM:SS / MM:SS [Seek Slider]      │
├─────────────────────────────────────────┤
│                                         │
│        FALLING NOTES VISUALIZATION      │
│        (3-5 seconds lookahead)          │
│                                         │
├─────────────────────────────────────────┤
│  [C-1] [#] [D] [#] [E] ... [G9]         │
│   (Full 128-key piano keyboard)         │
└─────────────────────────────────────────┘
```

### 2.2 Controls
- **File Upload Button**: Select .mid/.midi file from system
- **Play Button**: Start/resume playback
- **Pause Button**: Pause playback (maintains position)
- **Seek Slider**: Drag to jump to any position in the song
- **Time Display**: Current playback time and total duration (MM:SS format)
- **Visual Feedback**: Play/pause buttons show current state

### 2.3 Styling
- **Theme**: Dark mode (fixed, no toggle)
- **Background**: Dark neutral (e.g., dark gray/black)
- **Keys**: Standard piano colors (white for natural notes, black for accidentals)
- **Falling Notes**: Channel-based colors with pitch-based brightness modulation
- **Highlight Effect**: When note plays, key lights up and rectangle glows/brightens
- **Typography**: Clean, readable fonts for time display and labels

---

## 3. File Handling

### 3.1 MIDI File Support
- **Format**: Standard MIDI files (.mid, .midi)
- **File Size**: Assume files under 2MB
- **Multi-Track**: Support all 16 MIDI channels
- **Metadata Handling**: 
  - Parse tempo changes (respect tempo map)
  - Handle time signature changes
  - Ignore unsupported meta events gracefully
  - Support MIDI CC events needed for sustain simulation
  - CC64 (Sustain Pedal): Extend note duration to CC64 release

### 3.2 File Validation & Error Handling
- **Valid File**: If parseable, play it
- **Invalid File**: Display clear error message (e.g., "Error: Invalid MIDI file format")
- **Partial Corruption**: Attempt to parse and play what's readable
- **File Size Error**: If somehow >2MB, show message advising loading a smaller file

### 3.3 Channel Display
- **Display All Channels**: Show all 16 MIDI channels regardless of whether they have notes
- **Empty Channels**: Show visually but no animations occur
- **Channel Identification**: Each channel clearly labeled with its color and number

---

## 4. Audio Playback Behavior

### 4.1 Sound Characteristics
- **Note On**: When note_on event triggers, audio synthesizer begins playing
- **Note Off / Note Duration**: Note continues sounding until:
  - Explicit note_off event occurs, OR
  - Duration timeout (typical MIDI behavior), OR
  - Sustain pedal extends duration (CC64 active)
- **Velocity Support**: Note velocity (1-127) maps to audio amplitude/volume
- **Pitch**: Exact note pitch (0-127) from MIDI note number

### 4.2 Synchronization
- **Real-Time Sync**: Audio playback and visual animation must be perfectly synchronized
- **Seek Behavior**: When user seeks:
  - Stop current audio immediately
  - Clear currently animating notes
  - Recalculate fall animation for next 3-5 seconds from new position
  - Restart synthesis from new position
- **Tempo Changes**: Respect tempo changes in MIDI file (update falling speed accordingly)

### 4.3 Playback Controls
- **Play**: Start playing from current position (0 if no file loaded)
- **Pause**: Stop audio and animation, maintain position
- **Seek**: User drags slider or clicks on timeline, jumps to that position
  - Smooth scrubbing feedback (show time being seeked to)
  - Allow seeking during playback

---

## 5. Technical Architecture

### 5.1 Audio Synthesis
- **Library**: Tone.js (recommended for ease of MIDI synthesis) or raw Web Audio API
- **Synthesis Method**: Polyphonic wavetable/FM synthesis (default piano/synth sound)
- **Constraint**: Must support 128 simultaneous voices (polyphony for orchestral/complex pieces)
- **Fallback**: If Web Audio not supported, show clear message: "Your browser doesn't support Web Audio API"

### 5.2 Animation Rendering
- **Canvas**: Use HTML5 Canvas for efficient rendering of many animated rectangles
- **Alternative**: WebGL if Canvas performance insufficient at 2000+ simultaneous notes
- **Frame Rate**: Target 60 FPS, allow graceful degradation to maintain 30+ FPS under heavy load
- **Rendering Pipeline**:
  1. Calculate falling note positions based on elapsed time
  2. Render falling rectangles with appropriate colors/opacity
  3. Render keyboard with current note highlights
  4. Composite and display

### 5.3 MIDI Parsing
- **Library**: Tone.js Midi parser or ToneJS's built-in parsing
- **Alternative**: TinySynth, jsmidparser, or Tuna.js
- **Data Structure**: Parse MIDI into note event queue sorted by time
- **Tempo Map**: Extract and store all tempo changes for proper timing

### 5.4 State Management
- **Playback State**: Current time, paused/playing flag, loaded MIDI data
- **No Persistence**: Don't save settings or last-loaded file between sessions
- **Per-Session**: Settings (theme, etc.) only live during current browser session

---

## 6. Edge Cases & Robustness

### 6.1 Polyphonic Handling
- **Dense Chords**: Support 20+ simultaneous notes
- **Performance Target**: Achieve 60 FPS; if not possible, degrade gracefully to 30-40 FPS
- **Strategy**: Use RequestAnimationFrame for smooth animation, optimize Canvas rendering with dirty rectangles if needed

### 6.2 Sustain Pedal (CC64)
- **Behavior**: When CC64 > 64, extend note duration until CC64 < 64
- **Visual**: Note rectangle persists and highlights as long as sustain is active
- **Audio**: Audio sustains as long as sustain pedal is active

### 6.3 Rapid Note Changes
- **Fast Release-Attack**: Notes that end and immediately have note_on for next note should cross-fade smoothly
- **No Pops/Clicks**: MIDI synthesizer should handle note transitions without artifacts

### 6.4 Large Files
- **Expected Max**: 2MB .mid files
- **Parsing**: Should parse MIDI and be ready to play within 1-2 seconds
- **Memory**: Store entire MIDI in memory (2MB is reasonable for modern browsers)

### 6.5 Browser Compatibility
- **Target**: Modern browsers only (Chrome 90+, Firefox 88+, Safari 14+, Edge 90+)
- **Features Used**: 
  - Canvas 2D context
  - Web Audio API with AudioContext
  - Fetch API for file upload handling
  - Array/Object methods (ES6+)
- **No IE11 Support Required**

---

## 7. Color & Visual Scheme

### 7.1 Channel Color System
- **16 Channels**: Assign distinct base colors to each MIDI channel (or user-selectable)
- **Suggested Palette** (16 colors for 16 channels):
  ```
  1: Red        5: Cyan      9: Magenta    13: Lime
  2: Blue       6: Yellow    10: Orange    14: Teal
  3: Green      7: Purple    11: Pink      15: Indigo
  4: White/Gray 8: Brown     12: Turquoise 16: Salmon
  ```

### 7.2 Brightness Variation by Pitch
- **Formula**: `brightness = 0.3 + (note_number / 127) * 0.7`
  - MIDI note 0 (C-1): 30% brightness
  - MIDI note 127 (G9): 100% brightness
  - Creates gradient where lower notes are dimmer, higher notes are brighter
- **Hue**: Stay within channel's base hue but modulate saturation/lightness

### 7.3 Highlight Effect
- **Playing Note**: When note_on occurs, rectangle and key both highlight
- **Highlight Style**: Increase brightness by 20-30%, add subtle glow (box-shadow or blur)
- **Fade Out**: Over 100ms, smoothly reduce opacity from 1.0 to 0.0 when note_off occurs

### 7.4 Background & Accents
- **Main Background**: Dark gray (#1a1a1a) or black (#000000)
- **Key Background**: Dark with subtle borders to show piano keys clearly
- **Canvas Overlay**: Semi-transparent grid (optional) to show time sections

---

## 8. Responsive Design

### 8.1 Screen Sizes
- **Desktop (1920x1080 and larger)**: Full experience, comfortable key widths
- **Tablet (iPad, 768x1024)**: Responsive layout, slightly narrow keys but functional
- **Mobile (< 768px width)**: Keys become very narrow (acceptable), layout stacks vertically if needed
- **Minimum Width**: 320px (mobile phones), keys scale down acceptably

### 8.2 Responsive Behavior
- **Keyboard**: Always full width, adjust key width proportionally
- **Visualization Area**: Takes remaining vertical space above keyboard
- **Controls**: Stack responsively, buttons remain touch-friendly (44px+ tap targets)
- **Landscape/Portrait**: App adapts gracefully to both orientations

---

## 9. Accessibility & Error Handling

### 9.1 Error States
- **No File Loaded**: Show placeholder "Load a MIDI file to begin"
- **Invalid MIDI**: Show error message with details
- **Audio not supported**: Show "This browser doesn't support Web Audio API"
- **Parse Errors**: Show "Unable to parse MIDI file (corrupted or unsupported format)"

### 9.2 User Feedback
- **Loading State**: Brief feedback while parsing MIDI (or instant if <2MB)
- **Play/Pause Feedback**: Visual button state change
- **Seek Feedback**: Show time tooltip while dragging slider

### 9.3 Keyboard Accessibility
- **No keyboard shortcuts required** for MVP
- **Tab Navigation**: File input, Play button, Pause button, Seek slider are tab-able (browser default)
- **Touch Support**: All buttons have appropriate hit areas for touch interaction

---

## 10. Performance Requirements

### 10.1 Targets
- **Startup**: Parse and display MIDI within 2 seconds
- **Animation**: 60 FPS target, minimum 30 FPS under extreme load
- **Memory**: Support files up to 2MB in memory
- **CPU**: Single-threaded (browser main thread) acceptable for scope

### 10.2 Optimization Strategies
- **Canvas Optimization**: Clear only changed regions if possible (dirty rectangle rendering)
- **Note Rendering**: Batch rectangle renders, use efficient Canvas APIs
- **Audio**: Lazy-synthesis (only synthesize notes being heard), not pre-synthesis
- **Garbage Collection**: Reuse objects, avoid creating new objects in render loop

### 10.3 Testing Thresholds
- **Simple MIDI**: < 500 notes - 60 FPS expected
- **Medium Complexity**: 500-2000 notes - 50-60 FPS expected
- **Orchestral**: 2000+ notes - 30-60 FPS acceptable with graceful degradation

---

## 11. Future Enhancements (Out of Scope for MVP)

The following features are explicitly NOT included in v1.0 but may be considered for future versions:

- Multiple visualization modes (e.g., waveform, 3D visualization)
- MIDI editing/composition tools
- Video export/screen recording
- Theme switcher (light/dark mode toggle)
- Session persistence (remember last loaded file)
- Real-time MIDI input from controller/keyboard
- Sustain/reverb/effects processing beyond basic synthesis
- Custom color schemes or color picker
- Playlist support (load multiple MIDI files)
- Audio output recording
- Transpose/pitch shift controls
- Speed/tempo adjustment without resampling

---

## 12. Development Roadmap

### Phase 1: Core MVP (Priority)
1. **File Upload & MIDI Parsing**
   - File input element
   - MIDI binary parser (Tone.js or equivalent)
   - Error handling for invalid files

2. **Audio Synthesis & Playback**
   - Initialize Web Audio API
   - Tone.js Synth or Piano sampler
   - Note-on/note-off triggering
   - Sustain pedal handling

3. **Visualization Engine**
   - Canvas setup and rendering loop
   - Falling note rectangles with color gradients
   - Piano keyboard display with highlights
   - 3-5 second lookahead calculation

4. **Playback Controls**
   - Play/Pause buttons
   - Seek slider with time display
   - Transport control state management

5. **Responsive Layout**
   - Mobile-responsive CSS
   - Touch-friendly interface
   - Dark mode styling

### Phase 2: Polish & Testing
- Cross-browser testing
- Performance profiling and optimization
- Error message refinement
- Visual polish and animation smoothing

### Phase 3: Future Extensions (Post-MVP)
- Additional visualization modes
- Advanced playback features (speed, transpose)
- Export capabilities

---

## 13. Success Criteria

✅ **Completed** when:
- User can upload any standard .mid file
- Audio plays perfectly synchronized with falling note animations
- Piano-roll visualization clearly shows which notes are being played
- Colors properly reflect channels with brightness variation by pitch
- User can play, pause, and seek through the song
- App performs smoothly (60 FPS) on typical orchestra piece (2000+ notes)
- Mobile-responsive layout works on phones and tablets
- Clear error messages for invalid files
- No state persistence between sessions

---

## 14. Technology Stack (Recommended)

| Component | Technology | Justification |
|-----------|-----------|---------------|
| **Parsing** | Tone.js Midi parser | Built-in, reliable, lightweight |
| **Synthesis** | Tone.js Synth + PolySynth | Simple API, polyphonic, Web Audio wrapper |
| **Rendering** | HTML5 Canvas 2D | Fast, efficient for 2000+ rects, good browser support |
| **Framework** | Vanilla JS / React / Vue | Choose based on team preference; vanilla sufficient for this scope |
| **Build** | Vite / Webpack | Fast builds, dev server, modern tooling |
| **Styling** | CSS3 (Grid/Flexbox) | Responsive, dark theme support |

---

**End of Specification**
