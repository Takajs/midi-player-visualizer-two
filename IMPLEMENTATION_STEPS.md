# MIDI Player Visualizer - Implementation Steps

**Project**: Web-based MIDI player with real-time piano-roll animation  
**Framework**: React + Tone.js + Vite  
**Build Target**: GitHub Pages (static hosting, prepare but don't deploy yet)

---

## Tech Stack Summary

| Component | Technology | Reason |
|-----------|-----------|--------|
| **Build** | Vite | Fast dev server, optimized builds |
| **Framework** | React 18+ | State management, component reusability |
| **Audio** | Tone.js | MIDI parsing + synthesis bundled |
| **Rendering** | Canvas 2D | Efficient for 2000+ animated rectangles |
| **Styling** | Tailwind + CSS | Responsive, dark theme, custom Canvas styles |
| **State** | React Hooks + Context | Central playback state, audio control |

---

## Phase 1: Project Setup & Architecture

### Step 1.1: Initialize React + Vite Project
```bash
npm create vite@latest midi-player -- --template react
cd midi-player
npm install
npm install tone canvas-confetti
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### Step 1.2: Define Project Structure
```
src/
├── components/           # React UI components
│   ├── App.jsx          # Root component
│   ├── FileUpload.jsx   # File input element
│   ├── Controls.jsx      # Play/Pause/Seek buttons
│   ├── Visualizer.jsx    # Canvas wrapper
│   ├── Keyboard.jsx      # Piano keyboard display
│   └── TimeDisplay.jsx   # MM:SS / MM:SS display
│
├── audio/               # Audio synthesis logic
│   ├── synth.js        # Tone.js synth initialization
│   ├── audioController.js # Play/stop/pause/seek
│   └── midiEvents.js   # Note on/off handling
│
├── canvas/             # Canvas rendering
│   ├── renderer.js     # Main render loop
│   ├── drawNotes.js    # Falling note rectangles
│   ├── drawKeyboard.js # Piano key rendering
│   └── colorUtils.js   # Channel/pitch color logic
│
├── midi/               # MIDI file handling
│   ├── parser.js       # Tone.js MIDI parsing
│   ├── eventQueue.js   # Sorted note event queue
│   ├── timingUtils.js  # Tick-to-time conversion
│   └── tempoMap.js     # Tempo change tracking
│
├── utils/              # General utilities
│   ├── constants.js    # Color palette, dimensions
│   ├── helpers.js      # Common utility functions
│   └── keyboard.js     # MIDI note/key mappings
│
├── context/            # React Context
│   └── PlaybackContext.js # Global playback state
│
├── App.css             # Component styles
├── index.css           # Global + Tailwind styles
└── main.jsx            # React entry point
```

### Step 1.3: Configure Tailwind
**tailwind.config.js**:
```javascript
export default {
  content: [
    "./index.html",
    "./src/**/*.{js,jsx}",
  ],
  theme: {
    extend: {
      colors: {
        dark: '#1a1a1a',
        'dark-light': '#2d2d2d',
      },
    },
  },
  plugins: [],
}
```

### Step 1.4: Setup Build for GitHub Pages
**vite.config.js**:
```javascript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  base: '/midi-player-visualizer/', // For GitHub Pages (repo name)
})
```

---

## Phase 2: State Management & Core Architecture

### Step 2.1: Create PlaybackContext
- **File**: `src/context/PlaybackContext.js`
- **Purpose**: Centralized state for entire app
- **State Variables**:
  ```javascript
  {
    midiData: null,           // Parsed MIDI object
    isLoading: false,         // File parsing in progress
    isPlaying: false,         // Audio playing flag
    currentTime: 0,           // Playback position (seconds)
    duration: 0,              // Total song duration (seconds)
    events: [],               // All note events (time-sorted)
    tempoMap: [],             // Tempo changes
    activeNotes: {},          // { noteKey: { startTime, endTime, velocity, channel } }
    error: null,              // Error message if any
  }
  ```
- **Context Methods**:
  - `loadMidiFile(file)` - Parse and store MIDI
  - `play()` - Start playback
  - `pause()` - Pause playback
  - `seek(timeInSeconds)` - Jump to position
  - `clearState()` - Reset for new file
  - `setCurrentTime(time)` - Update playback position
  - `setActiveNotes(noteMap)` - Update currently playing notes

### Step 2.2: Define Constants
- **File**: `src/utils/constants.js`
- **Contents**:
  ```javascript
  // Color palette for 16 MIDI channels
  export const CHANNEL_COLORS = {
    0: '#FF6B6B', 1: '#4ECDC4', 2: '#45B7D1', 3: '#96CEB4',
    4: '#FFEAA7', 5: '#DDA15E', 6: '#BC6C25', 7: '#C9ADA7',
    8: '#9B59B6', 9: '#E74C3C', 10: '#3498DB', 11: '#2ECC71',
    12: '#F39C12', 13: '#1ABC9C', 14: '#34495E', 15: '#E67E22',
  }
  
  // Visual constants
  export const MIDI_NOTES_COUNT = 128
  export const PIXELS_PER_SECOND = 100  // Vertical pixels per second of lookahead
  export const LOOKAHEAD_SECONDS = 4     // 3-5 seconds, default 4
  export const NOTE_FADE_DURATION = 0.1  // 100ms fade out
  export const HIGHLIGHT_BRIGHTNESS = 1.3 // 30% brighter when playing
  
  // MIDI note ranges
  export const MIDI_RANGE = { min: 0, max: 127 }
  export const NOTE_NAMES = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B']
  ```

### Step 2.3: Create MIDI Constants & Helpers
- **File**: `src/utils/keyboard.js`
- **Purpose**: MIDI note number ↔ keyboard key mapping
- **Functions**:
  ```javascript
  export function midiNoteToName(noteNum) {
    // 0 => "C-1", 60 => "C4", 127 => "G9"
  }
  
  export function isBlackKey(noteNum) {
    // Return true if note is C#, D#, F#, G#, A#
  }
  
  export function getKeyPosition(noteNum) {
    // Return pixel position on keyboard for note (0-127)
  }
  ```

---

## Phase 3: MIDI File Parsing

### Step 3.1: MIDI Parser Module
- **File**: `src/midi/parser.js`
- **Purpose**: Use Tone.js to parse MIDI and extract events
- **Logic**:
  ```javascript
  import * as Tone from 'tone'
  
  export async function parseMidiFile(file) {
    const arrayBuffer = await file.arrayBuffer()
    const midi = new Tone.Midi(arrayBuffer)
    
    // Extract all tracks and events
    const allEvents = []
    const tempoChanges = []
    
    // Process tempo events
    midi.header.tempos.forEach(tempo => {
      tempoChanges.push({
        time: tempo.time,
        bpm: tempo.bpm,
      })
    })
    
    // Process all tracks
    midi.tracks.forEach((track, trackIndex) => {
      track.forEach(note => {
        allEvents.push({
          type: 'noteOn',
          time: note.time,
          noteNum: note.midi,
          velocity: note.velocity * 127, // Normalize to 1-127
          channel: trackIndex % 16,
          duration: note.duration,
        })
        
        allEvents.push({
          type: 'noteOff',
          time: note.time + note.duration,
          noteNum: note.midi,
          channel: trackIndex % 16,
        })
      })
      
      // Process CC events (sustain)
      track.controlChanges[64]?.forEach(cc => {
        allEvents.push({
          type: 'controlChange',
          time: cc.time,
          control: 64,
          value: cc.value,
          channel: trackIndex % 16,
        })
      })
    })
    
    // Sort all events by time
    allEvents.sort((a, b) => a.time - b.time)
    
    return {
      events: allEvents,
      tempoChanges,
      duration: midi.duration,
    }
  }
  ```

### Step 3.2: Event Queue & Timing Utilities
- **File**: `src/midi/eventQueue.js`
- **Purpose**: Build searchable event queue for fast seeking
- **Logic**:
  - Store events in sorted array indexed by time
  - Provide binary search for seek (find events at specific time)
  - Track which events have been processed during playback

### Step 3.3: Tempo Mapping
- **File**: `src/midi/tempoMap.js`
- **Purpose**: Convert MIDI ticks to real time considering tempo changes
- **Functions**:
  ```javascript
  export function buildTempoMap(midiObject, tempoEvents) {
    // Create time segments where each segment has a constant BPM
    // Use for converting tick-based timing to seconds
  }
  
  export function timeToAbsoluteSeconds(midiTime, tempoMap) {
    // Given MIDI time and tempo map, return absolute seconds
    // Account for tempo changes
  }
  ```

---

## Phase 4: Audio Synthesis Engine

### Step 4.1: Initialize Tone.js Synth
- **File**: `src/audio/synth.js`
- **Purpose**: Create polyphonic synth for MIDI playback
- **Logic**:
  ```javascript
  import * as Tone from 'tone'
  
  export class MidiSynth {
    constructor() {
      // Create polyphonic synth (128 voices for orchestral)
      this.polySynth = new Tone.PolySynth(Tone.Synth, {
        oscillator: { type: 'triangle' },
        envelope: {
          attack: 0.005,
          decay: 0.1,
          sustain: 0.3,
          release: 0.5,
        },
      }).toDestination()
      
      this.activeNotes = new Map() // { noteKey: { noteNum, velocity } }
    }
    
    noteOn(noteNum, velocity = 100) {
      const freq = Tone.Midi(noteNum).toFrequency()
      // Map velocity to volume (0-1)
      const volume = velocity / 127
      this.polySynth.triggerAttack(noteNum, Tone.now(), volume)
      this.activeNotes.set(noteNum, { velocity, startTime: Tone.now() })
    }
    
    noteOff(noteNum) {
      this.polySynth.triggerRelease(noteNum, Tone.now())
      this.activeNotes.delete(noteNum)
    }
    
    stopAll() {
      this.polySynth.triggerRelease(Array.from(this.activeNotes.keys()))
      this.activeNotes.clear()
    }
  }
  ```

### Step 4.2: Audio Controller
- **File**: `src/audio/audioController.js`
- **Purpose**: Manage playback state synchronized with animation
- **Key Logic**:
  ```javascript
  export class AudioController {
    constructor(events, tempoMap, synth) {
      this.events = events        // Sorted note events
      this.tempoMap = tempoMap
      this.synth = synth
      this.audioContext = Tone.getContext()
      this.isPlaying = false
      this.currentEventIndex = 0
      this.seekQueueTime = null   // Time to seek to
      this.sustainPedal = false   // CC64 state
    }
    
    play(fromTime = 0) {
      if (this.audioContext.state === 'suspended') {
        this.audioContext.resume()
      }
      
      this.isPlaying = true
      this.startTime = this.audioContext.currentTime - fromTime
      this.currentEventIndex = this.findEventIndex(fromTime)
      this.scheduleNextEvents()
    }
    
    pause() {
      this.isPlaying = false
      this.synth.stopAll()
    }
    
    seek(targetTime) {
      // Stop current audio
      this.synth.stopAll()
      this.pause()
      
      // Find events from new position
      this.currentEventIndex = this.findEventIndex(targetTime)
      
      // Resume if was playing
      if (this.wasPlaying) {
        this.play(targetTime)
      }
    }
    
    findEventIndex(timeInSeconds) {
      // Binary search in events array
      // Find first event at or after timeInSeconds
    }
    
    scheduleNextEvents() {
      const currentAudioTime = this.audioContext.currentTime
      const currentMidiTime = currentAudioTime - this.startTime
      
      // Schedule all events within next 100ms
      while (this.currentEventIndex < this.events.length) {
        const event = this.events[this.currentEventIndex]
        
        if (event.time > currentMidiTime + 0.1) break
        
        if (event.type === 'noteOn') {
          this.synth.noteOn(event.noteNum, event.velocity)
        } else if (event.type === 'noteOff') {
          this.synth.noteOff(event.noteNum)
        } else if (event.type === 'controlChange' && event.control === 64) {
          this.sustainPedal = event.value > 64
        }
        
        this.currentEventIndex++
      }
      
      if (this.isPlaying) {
        requestAnimationFrame(() => this.scheduleNextEvents())
      }
    }
    
    getCurrentTime() {
      if (!this.isPlaying) return 0
      return this.audioContext.currentTime - this.startTime
    }
  }
  ```

---

## Phase 5: Canvas Visualization Engine

### Step 5.1: Canvas Renderer Core
- **File**: `src/canvas/renderer.js`
- **Purpose**: Main render loop managing animation timing
- **Logic**:
  ```javascript
  export class CanvasRenderer {
    constructor(canvas, events, tempoMap) {
      this.canvas = canvas
      this.ctx = canvas.getContext('2d')
      this.events = events
      this.tempoMap = tempoMap
      
      // Set canvas dimensions with retina support
      const dpr = window.devicePixelRatio || 1
      this.width = canvas.clientWidth
      this.height = canvas.clientHeight
      this.canvas.width = this.width * dpr
      this.canvas.height = this.height * dpr
      this.ctx.scale(dpr, dpr)
      
      this.animationId = null
      this.currentTime = 0
      this.activeNotes = {}
      this.fadeOutNotes = []
    }
    
    startRender() {
      const loop = (timestamp) => {
        this.render()
        this.animationId = requestAnimationFrame(loop)
      }
      this.animationId = requestAnimationFrame(loop)
    }
    
    stopRender() {
      if (this.animationId) {
        cancelAnimationFrame(this.animationId)
      }
    }
    
    render() {
      // Clear canvas
      this.ctx.fillStyle = '#000000'
      this.ctx.fillRect(0, 0, this.width, this.height)
      
      // Draw falling notes
      this.drawFallingNotes()
      
      // Draw keyboard
      this.drawKeyboard()
    }
    
    setCurrentTime(seconds) {
      this.currentTime = seconds
    }
    
    setActiveNotes(noteMap) {
      this.activeNotes = noteMap
    }
    
    startNoteAnimation(noteNum, channel, duration) {
      // Trigger animation for note starting
    }
    
    stopNoteAnimation(noteNum) {
      // Trigger 100ms fade out for note
      this.fadeOutNotes.push({
        noteNum,
        startTime: Date.now(),
        duration: 100,
      })
    }
  }
  ```

### Step 5.2: Draw Falling Notes
- **File**: `src/canvas/drawNotes.js`
- **Purpose**: Render rectangles for each falling note
- **Algorithm**:
  ```javascript
  export function drawFallingNotes(
    ctx, events, currentTime, activeNotes,
    tempoMap, width, height, LOOKAHEAD_SECONDS
  ) {
    const PIXELS_PER_SECOND = 100
    const bottomY = height * 0.8 // Keyboard is at 80% height
    
    // Filter events to those within lookahead window
    const visibleEvents = events.filter(e => 
      e.time >= currentTime && 
      e.time <= currentTime + LOOKAHEAD_SECONDS &&
      e.type === 'noteOn'
    )
    
    visibleEvents.forEach(noteEvent => {
      const timeTillPlayback = noteEvent.time - currentTime
      const yPosition = bottomY - (timeTillPlayback * PIXELS_PER_SECOND)
      
      // Calculate note dimensions
      const noteDurationPixels = Math.max(
        noteEvent.duration * PIXELS_PER_SECOND,
        2 // Minimum visible height
      )
      const keyWidth = width / 128
      const xPosition = noteEvent.noteNum * keyWidth
      
      // Get color based on channel and pitch
      const color = getNotColor(noteEvent.channel, noteEvent.noteNum)
      const isActive = activeNotes[noteEvent.noteNum]
      
      if (isActive) {
        // Draw with highlight glow
        ctx.fillStyle = brighten(color, HIGHLIGHT_BRIGHTNESS)
        ctx.shadowColor = color
        ctx.shadowBlur = 15
      } else {
        ctx.fillStyle = color
        ctx.shadowBlur = 0
      }
      
      // Draw rectangle
      ctx.fillRect(xPosition, yPosition, keyWidth, noteDurationPixels)
      ctx.strokeStyle = 'rgba(255,255,255,0.2)'
      ctx.lineWidth = 1
      ctx.strokeRect(xPosition, yPosition, keyWidth, noteDurationPixels)
    })
  }
  
  function getNotColor(channel, noteNum) {
    const baseColor = CHANNEL_COLORS[channel]
    const brightness = 0.3 + (noteNum / 127) * 0.7
    return adjustBrightness(baseColor, brightness)
  }
  ```

### Step 5.3: Draw Piano Keyboard
- **File**: `src/canvas/drawKeyboard.js`
- **Purpose**: Render 128-key piano keyboard at bottom
- **Algorithm**:
  ```javascript
  export function drawKeyboard(
    ctx, width, height, activeNotes
  ) {
    const keyboardHeight = height * 0.2
    const keyboardY = height * 0.8
    const keyWidth = width / 128
    const NATURAL_KEY_HEIGHT = keyboardHeight
    const BLACK_KEY_HEIGHT = keyboardHeight * 0.6
    
    // Draw white keys first
    for (let i = 0; i < 128; i++) {
      if (!isBlackKey(i)) {
        const x = i * keyWidth
        const isActive = !!activeNotes[i]
        
        ctx.fillStyle = isActive ? '#DDDDDD' : '#FFFFFF'
        ctx.fillRect(x, keyboardY, keyWidth - 1, NATURAL_KEY_HEIGHT)
        ctx.strokeStyle = '#333333'
        ctx.lineWidth = 2
        ctx.strokeRect(x, keyboardY, keyWidth - 1, NATURAL_KEY_HEIGHT)
      }
    }
    
    // Draw black keys on top
    for (let i = 0; i < 128; i++) {
      if (isBlackKey(i)) {
        const x = i * keyWidth + (keyWidth * 0.6)
        const isActive = !!activeNotes[i]
        
        ctx.fillStyle = isActive ? '#555555' : '#000000'
        ctx.fillRect(x, keyboardY, keyWidth * 0.8, BLACK_KEY_HEIGHT)
        ctx.strokeStyle = '#111111'
        ctx.lineWidth = 1
        ctx.strokeRect(x, keyboardY, keyWidth * 0.8, BLACK_KEY_HEIGHT)
      }
    }
  }
  ```

### Step 5.4: Color Utilities
- **File**: `src/canvas/colorUtils.js`
- **Purpose**: Color interpolation and brightness adjustment
- **Functions**:
  ```javascript
  export function adjustBrightness(hexColor, brightness) {
    // Convert hex to HSL, adjust lightness, return hex
    // brightness: 0.3 (dark) to 1.0 (bright)
  }
  
  export function brighten(hexColor, factor) {
    // Multiply lightness by factor
  }
  ```

---

## Phase 6: React Components & UI

### Step 6.1: App Root Component
- **File**: `src/components/App.jsx`
- **Structure**:
  ```jsx
  import { PlaybackProvider } from '../context/PlaybackContext'
  import FileUpload from './FileUpload'
  import Controls from './Controls'
  import Visualizer from './Visualizer'
  import Keyboard from './Keyboard'
  
  export default function App() {
    return (
      <PlaybackProvider>
        <div className="min-h-screen bg-dark flex flex-col">
          {/* Header with controls */}
          <div className="p-6 bg-dark-light">
            <FileUpload />
            <Controls />
          </div>
          
          {/* Main visualization area */}
          <div className="flex-1">
            <Visualizer />
          </div>
          
          {/* Piano keyboard footer */}
          <div className="bg-dark-light">
            <Keyboard />
          </div>
        </div>
      </PlaybackProvider>
    )
  }
  ```

### Step 6.2: File Upload Component
- **File**: `src/components/FileUpload.jsx`
- **Features**:
  - Input element with `.mid` file filter
  - Loading indicator during parsing
  - Error message display
  - Loading bar showing parse progress (if multi-step)

### Step 6.3: Controls Component
- **File**: `src/components/Controls.jsx`
- **Elements**:
  - Play button (shown when paused or not playing)
  - Pause button (shown when playing)
  - Seek slider (range input)
  - Time display (current / total)

### Step 6.4: Visualizer Component
- **File**: `src/components/Visualizer.jsx`
- **Logic**:
  - Canvas element with ref
  - Initialize renderer on mount
  - Update renderer currentTime on playback update
  - Clean up on unmount

### Step 6.5: Keyboard Component
- **File**: `src/components/Keyboard.jsx`
- **Logic**:
  - Display piano keys (128 total)
  - Highlight active keys based on context
  - Responsive to screen width

---

## Phase 7: State Management & Synchronization

### Step 7.1: PlaybackContext Provider
- **File**: `src/context/PlaybackContext.js`
- **Key Functions**:
  ```javascript
  export const PlaybackContext = createContext()
  
  export function PlaybackProvider({ children }) {
    const [state, dispatch] = useReducer(playbackReducer, initialState)
    
    // Load MIDI file
    const loadMidiFile = async (file) => {
      dispatch({ type: 'SET_LOADING', payload: true })
      try {
        const midiData = await parseMidiFile(file)
        dispatch({ type: 'SET_MIDI_DATA', payload: midiData })
        dispatch({ type: 'SET_ERROR', payload: null })
      } catch (err) {
        dispatch({ type: 'SET_ERROR', payload: err.message })
      }
      dispatch({ type: 'SET_LOADING', payload: false })
    }
    
    // Play
    const play = async () => {
      await Tone.start()
      // Initialize audio controller and start playback
      dispatch({ type: 'SET_PLAYING', payload: true })
    }
    
    // Pause
    const pause = () => {
      dispatch({ type: 'SET_PLAYING', payload: false })
    }
    
    // Seek
    const seek = (timeInSeconds) => {
      dispatch({ type: 'SET_CURRENT_TIME', payload: timeInSeconds })
    }
    
    return (
      <PlaybackContext.Provider value={{ state, loadMidiFile, play, pause, seek }}>
        {children}
      </PlaybackContext.Provider>
    )
  }
  ```

### Step 7.2: Animation Loop Synchronization
- **Where**: In `Visualizer` component
- **Logic**:
  ```javascript
  useEffect(() => {
    if (!state.isPlaying) return
    
    const syncLoop = setInterval(() => {
      // Get current time from audio controller
      const audioTime = audioController.getCurrentTime()
      
      // Update canvas renderer
      renderer.setCurrentTime(audioTime)
      renderer.setActiveNotes(state.activeNotes) // From context
      
      // Process upcoming events and update active notes
      // This drives the animation
    }, 16) // ~60 FPS
    
    return () => clearInterval(syncLoop)
  }, [state.isPlaying, audioController, renderer])
  ```

---

## Phase 8: Error Handling & Edge Cases

### Step 8.1: File Upload Errors
- Invalid MIDI file: Show "Unable to parse MIDI file"
- Unsupported browser: Show "Web Audio API not supported"
- File too large: Show "File exceeds 2MB limit"

### Step 8.2: Playback Edge Cases
- **Seek during playback**: Stop synth, jump events queue, resume
- **Rapid note events**: Batch-process events within same frame
- **Sustain pedal (CC64)**: Track state, extend note off until pedal release
- **Very dense chords**: Render all, accept 30+ FPS under extreme load

### Step 8.3: Resource Cleanup
- On file reload: Stop audio, clear canvas, reset state
- On pause: Stop synth, pause animation loop
- On unmount: Cancel animation frames, dispose Tone.js, clear context

---

## Phase 9: Responsive Design

### Step 9.1: Tailwind Responsive Classes
- Use `md:`, `lg:` breakpoints for screen-size adjustments
- Buttons stay 44px+ touch-friendly
- Canvas scales with container width

### Step 9.2: High-DPI Display Support
- Set canvas internal resolution to `width * dpr`, then scale context
- Result: crisp rendering on Retina/high-DPI screens

### Step 9.3: Mobile Touch
- Seek slider: larger touch area (easier dragging)
- Buttons: standard browser touch handling
- No custom gestures (zoom, pan, etc.)

---

## Phase 10: Implementation Order

### Sprint 1: Foundation (Days 1-2)
1. Setup Vite + React + Tailwind environment
2. Define project structure and constants
3. Create PlaybackContext provider skeleton
4. Build basic App layout with file upload form
5. Create FileUpload, Controls, Visualizer, Keyboard components (UI only, no logic yet)

### Sprint 2: MIDI Parsing & Audio Engine (Days 3-4)
1. Implement MIDI parser using Tone.js
2. Build event queue and timing utilities
3. Implement tempo map for proper timing
4. Build Tone.js synth initialization
5. Create AudioController for play/pause/seek
6. Test with sample MIDI files

### Sprint 3: Canvas Rendering (Days 5-6)
1. Implement CanvasRenderer core loop
2. Build falling notes drawing logic with color mapping
3. Implement piano keyboard rendering
4. Add retina/high-DPI display support
5. Implement note fade-out animation (100ms)
6. Test rendering performance with 2000-note piece

### Sprint 4: Integration & Synchronization (Days 7-8)
1. Wire PlaybackContext to components
2. Implement animation loop that syncs with audio
3. Connect file upload to MIDI parsing and state
4. Implement play/pause/seek controls
5. Sync active notes between audio engine and renderer
6. Test audio-visual synchronization

### Sprint 5: Polish & Edge Cases (Days 9-10)
1. Implement sustain pedal (CC64) logic
2. Add error handling and user feedback
3. Implement loading indicator during MIDI parse
4. Test responsive design on mobile/tablet
5. Optimize Canvas rendering if needed (batch rendering, dirty rectangles)
6. Cross-browser testing

### Sprint 6: Final Testing & Optimization (Days 11-12)
1. Performance profiling with large MIDI files
2. Frame rate monitoring and optimization if needed
3. Resource cleanup verification
4. Error message refinement
5. Accessibility verification
6. Final polish and bug fixes

---

## Phase 11: Critical Implementation Details

### Synchronization Strategy: Audio-Driven
- **Source of Truth**: `audioContext.currentTime` (Web Audio API time)
- **Why**: Web Audio time is hardware-synced and most accurate
- **Implementation**:
  1. When user plays, record `startTime = audioContext.currentTime`
  2. On each animation frame: `currentTime = audioContext.currentTime - startTime`
  3. This `currentTime` is used to:
     - Find which notes should be visible (falling)
     - Query which notes are currently active (playing)
     - Update Canvas renderer position

### Event Scheduling Strategy
- **Pre-parse All Events**: Parse entire MIDI into flat `events[]` array during file load
- **Binary Search on Seek**: Use binary search to find events after seek position
- **Schedule Lazily**: Only trigger audio synth events that need to play soon
- **Why This Works**:
  - Seeking is instant (no re-parsing needed)
  - Audio-visual sync is automatic (driven by audioContext.currentTime)
  - Memory usage is reasonable (pre-parsed events)

### Canvas Rendering Optimization
- **Full Redraw Each Frame**: Start with full canvas clear + redraw
- **Rationale**: Simpler to implement correctly, still fast enough for 2000 notes
- **If Performance Issues Arise**:
  - Implement dirty rectangle tracking
  - Only draw changed regions and piano keys when they change
  - Use batch rendering for notes (DrawCall batching)
- **Performance Target**: 60 FPS for <2000 notes, 30+ FPS for orchestral pieces

### Note Timing Calculations
```
Note visible when:  currentTime <= noteEvent.time <= currentTime + LOOKAHEAD_SECONDS

Y Position when visible:
  timeTillPlayback = noteEvent.time - currentTime
  pixelsFromBottom = timeTillPlayback * PIXELS_PER_SECOND (100px/sec)
  yPosition = canvasHeight - keyboardHeight - pixelsFromBottom

Note Highlighted when:
  noteNum in activeNotes (determined by current playback time)
  
Note Fade-Out when:
  noteOff event triggers → transition opacity 1.0→0.0 over 100ms
```

### Sustain Pedal Logic
```
When CC64 (sustain) > 64:
  - Sustain flag = true
  - Currently-playing notes don't end on noteOff
  - Instead, notes persist until CC64 < 64

Implementation:
  1. Track CC64 state in audioController
  2. On noteOff: check sustainPedal flag
  3. If sustain active: queue release for later
  4. On CC64 release: trigger all queued note-offs
```

### Color Algorithm
```
Base Channel Color: CHANNEL_COLORS[channel % 16]

Pitch-Based Brightness:
  brightness = 0.3 + (noteNum / 127) * 0.7
  
  Where:
  - noteNum 0 (C-1) → brightness 0.3 (darkest 30%)
  - noteNum 127 (G9) → brightness 1.0 (brightest 100%)
  
  Adjust HSL: keep H/S from base color, modify L (lightness)

When Note Plays:
  brightness *= HIGHLIGHT_BRIGHTNESS (1.3)
  Add glow: ctx.shadowColor = color, ctx.shadowBlur = 15
```

---

## Phase 12: Testing Checklist

### Unit Testing (Manual)
- [ ] MIDI parser correctly extracts all notes from sample files
- [ ] Event queue is sorted by time
- [ ] Binary search finds correct event index after seek
- [ ] Tempo map correctly converts MIDI ticks to seconds
- [ ] Color gradient works for all 128 notes in all 16 channels

### Integration Testing
- [ ] Select MIDI file → parses → state updated
- [ ] Play button triggers audio and animation
- [ ] Notes fall at correct speed tied to song tempo
- [ ] Seek slider updates visualizer position
- [ ] Notes highlight when playing
- [ ] Sustain pedal extends note duration

### Performance Testing
- [ ] Simple MIDI (<500 notes): 60 FPS sustained
- [ ] Medium MIDI (500-2000 notes): 50-60 FPS sustained
- [ ] Orchestral MIDI (2000+ notes): 30+ FPS (acceptable degradation)
- [ ] No memory leaks after multiple file loads
- [ ] Cleanup successful: no lingering audio/Canvas state

### Browser Testing
- [ ] Chrome 90+: Full functionality
- [ ] Firefox 88+: Full functionality
- [ ] Safari 14+: Full functionality
- [ ] Edge 90+: Full functionality

### Responsive Testing
- [ ] Desktop (1920x1080): Comfortable key widths, smooth animation
- [ ] Tablet (iPad 768x1024): Functional, slightly narrow keys
- [ ] Mobile (375x667): Very narrow keys acceptable, all controls accessible
- [ ] Touch: Seek slider drags smoothly, buttons respond immediately

### Accessibility Testing
- [ ] File input accessible
- [ ] Play/Pause buttons clearly labeled
- [ ] Time display readable
- [ ] Error messages visible and clear
- [ ] Tab navigation works through controls

### Edge Case Testing
- [ ] Empty MIDI file: Clear error message
- [ ] Corrupted MIDI: Graceful error handling
- [ ] Rapid seek actions: No crashes or sync bugs
- [ ] Fast note sequences: No visual pop-in or stuttering
- [ ] Very high MIDI velocities: Audio doesn't clip
- [ ] Very low velocities: Still audible and visible
- [ ] Channel 10 (drum channel): Works like other channels

---

## Phase 13: Build & Deployment Prep

### Build Configuration
```bash
# Development
npm run dev

# Production build (for GitHub Pages)
npm run build
# Output: dist/ folder
# Deploy to: GitHub Pages with base URL = repository name
```

### Vite Config for GitHub Pages
```javascript
// vite.config.js already configured with:
base: '/midi-player-visualizer/'
// This makes assets load from correct path on GitHub Pages
```

### No Deployment Yet
- Build artifacts will be generated
- But do NOT push to `gh-pages` branch
- Just ensure build process works and output is correct

---

## Phase 14: Known Limitations & Future Improvements

### Limitations (MVP Scope)
- Single MIDI file at a time (no playlist)
- No MIDI editing or composition
- No audio effects (reverb, EQ, etc.)
- No recording/export of visualization
- No theme switcher (dark mode only)
- No state persistence between sessions

### Future Enhancements
- [ ] Multiple visualization modes (waveform, 3D, abstract)
- [ ] Video export of visualization
- [ ] Real-time MIDI input from controllers
- [ ] Transpose/pitch shift
- [ ] Speed/tempo adjustment
- [ ] Custom color customization
- [ ] Playlist support
- [ ] Drag/drop file upload
- [ ] Audio analysis (waveform display)
- [ ] Performance metrics display (FPS, latency)

---

## Phase 15: Key Decisions Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Framework** | React 18+ | Modern, component-based, easy state mgmt |
| **Audio Lib** | Tone.js | Bundled parsing + synthesis, polyphonic |
| **Rendering** | Canvas 2D | Efficient for 2000+ rects, good perf |
| **Build Tool** | Vite | Fast, modern, great dev experience |
| **Styling** | Tailwind + CSS | Responsive, dark theme, customizable |
| **Sync Strategy** | Audio-driven | Most reliable, hardware-synced timing |
| **Event Parsing** | Pre-parse all | Instant seeking, manageable memory |
| **Polish** | Full redraw** | Simpler initially, optimize if needed |
| **Scope** | MVP only | No editing, effects, export, features |
| **Tests** | Manual | No automated tests for MVP |
| **Deploy** | GitHub Pages | Free, static hosting, prepare not deploy |

---

## Implementation Success Criteria

### Core Functionality ✅
- User uploads .mid file
- Audio synthesized and played correctly
- Piano-roll notes fall and animate
- Notes highlight when playing
- Seek slider works correctly

### Audio Sync ✅
- Audio and animation perfectly in sync
- Seeking maintains synchronization
- Sustain pedal extends both audio and visual

### Visuals ✅
- 128-key keyboard displayed full width
- Notes colored by channel with pitch brightness
- Falling notes proportional to duration
- 3-5 second lookahead visible
- 100ms smooth fade on note end
- Mobile responsive

### Performance ✅
- 60 FPS on simple MIDI
- 50-60 FPS on medium MIDI
- 30+ FPS on orchestral MIDI
- No memory leaks
- Clean resource cleanup

### Robustness ✅
- Clear error messages
- Graceful handling of corrupt MIDI
- No crashes on extreme polyphony
- Proper cleanup on state transitions

---

**End of Implementation Steps**
