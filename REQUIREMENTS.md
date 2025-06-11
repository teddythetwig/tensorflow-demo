# Exercise Tracking System - Technical Requirements

## Overview
The Exercise Tracking System is a web-based application that uses computer vision to track body movements during exercises. It leverages TensorFlow.js and the MoveNet pose detection model to provide real-time feedback on exercise form and progress.

## Core Components

### 1. Pose Detection
- Uses TensorFlow.js and MoveNet for real-time pose detection
- Tracks 17 key body points (joints) including shoulders, elbows, wrists, hips, knees, and ankles
- Provides confidence scores for each detected joint
- Minimum confidence threshold of 0.3 (30%) for valid joint detection

### 1.1 Tracked Joints
The system tracks the following 17 keypoints on the human body:

Upper Body:
- nose
- left_eye, right_eye
- left_ear, right_ear
- left_shoulder, right_shoulder
- left_elbow, right_elbow
- left_wrist, right_wrist

Lower Body:
- left_hip, right_hip
- left_knee, right_knee
- left_ankle, right_ankle

Additional Points:
- center_hip (calculated as midpoint between left and right hip)

Common Angle Measurements:
- Shoulders: angle between shoulder-elbow-wrist
- Elbows: angle between shoulder-elbow-wrist
- Hips: angle between shoulder-hip-knee
- Knees: angle between hip-knee-ankle

### 2. Exercise Definition
- Supports two types of exercise stages:
  - **Movement Stages**: Track transitions between specific joint angles
  - **Hold Stages**: Maintain specific joint angles for a duration
- Exercise definitions stored in `exercises.json`
- Each exercise can have multiple stages in sequence
- Only relevant joints for the current stage are tracked and visualized

#### 2.1 Movement Stages
Movement stages track dynamic transitions between two positions. They are defined by:
- Starting angles (alpha) for each tracked joint
- Ending angles (omega) for each tracked joint
- Tolerance ranges for both starting and ending positions
- Required joints that must be tracked for the movement
- Optional joints that can be ignored if not visible

#### 2.4 Joint Visibility Rules
- Only joints specified in the current stage's definition are visualized
- Required joints are always shown when detected
- Optional joints are only shown when:
  1. They are detected with confidence > 0.3
  2. They are relevant to the current movement/hold
  3. They are connected to a required joint
- Non-exercise joints (e.g., eyes, ears) are never shown
- Connected joints rules:
  - If shoulder is required, show connected elbow
  - If elbow is required, show connected wrist
  - If hip is required, show connected knee
  - If knee is required, show connected ankle
- Joint connections (lines) are only drawn between visible joints

Example of selective visualization:
```json
{
  "name": "Knee Bend",
  "type": "movement",
  "movements": {
    "left_knee": {
      "start": 180,
      "end": 90,
      "required": true
    },
    "right_knee": {
      "start": 180,
      "end": 90,
      "required": true
    }
  },
  "visualization": {
    "show_connected": true,     // Show joints connected to required joints
    "additional_joints": [      // Extra joints to show for context
      "left_hip",
      "right_hip"
    ]
  }
}
```

#### 2.2 Hold Stages
Hold stages require maintaining specific joint angles for a duration. They are defined by:
- Target angle for each tracked joint
- Hold duration in seconds
- Tolerance range for maintaining position
- Maximum cumulative time outside position
- Required and optional joints

Example Hold Stage (Wall Squat - Hold):
```json
{
  "name": "Hold Squat",
  "type": "hold",
  "hold": {
    "duration": 30,    // Hold for 30 seconds
    "joints": {
      "left_hip": {
        "angle": 90,
        "required": true
      },
      "right_hip": {
        "angle": 90,
        "required": true
      },
      "left_knee": {
        "angle": 90,
        "required": true
      },
      "right_knee": {
        "angle": 90,
        "required": true
      }
    },
    "tolerance": {
      "maintain": 15,      // Degrees of tolerance while holding
      "max_deviation": 5,  // Seconds allowed outside position
      "reset_after": 3     // Seconds of good form to reset deviation timer
    }
  }
}
```

#### 2.3 Stage Progression Rules

##### Joint Locking Mechanism
- Joints become "locked" when:
  1. Joint angle is within tolerance of target angle
  2. Joint has maintained position for minimum lock time (default: 500ms)
  3. Joint confidence remains above threshold (0.3)
- Joints become "unlocked" when:
  1. Joint angle moves outside tolerance range
  2. Joint confidence drops below threshold
  3. Joint position is lost for > 5 seconds
- Lock status is visually indicated:
  - Unlocked: Normal joint color
  - Locking: Yellow progress indicator
  - Locked: Green indicator with checkmark
  - Lost: Red indicator with warning

##### Movement Stage Progression
- Movement stages progress when:
  1. All required joints reach and lock in starting position
  2. Movement to ending position is detected
  3. All required joints reach and lock in ending position
  4. Minimum hold time at end position is met (default: 500ms)

##### Hold Stage Progression
- Hold stages progress when:
  1. All required joints reach and lock in target position
  2. Joints remain locked for specified duration
  3. Maximum deviation time is not exceeded
  4. If position is lost, timer pauses until position is regained

##### Lock Timeout Rules
- Joint locks expire after:
  1. 5 seconds of lost tracking
  2. 3 seconds outside tolerance range
  3. Confidence drops below threshold for > 2 seconds
- Lock timeouts can be configured per exercise/stage
- Timeout events trigger visual and audio feedback
- After timeout:
  1. Stage resets to initial state
  2. All joint locks are cleared
  3. User must re-establish starting position

##### Stage Failure Conditions
- Required joints lost for > 5 seconds
- Hold position lost for > max_deviation seconds
- Movement in wrong direction detected
- Exercise timeout reached (configurable)
- Too many lock timeouts (default: 3 attempts)

### 3. Joint Angle Tracking
- Calculates angles between connected joints in real-time
- Supports angle smoothing to reduce jitter
- Configurable angle tolerance (1° to 30°)
- Maintains separate tolerances for:
  - Initial position detection
  - Position maintenance

### 4. State Machine
- Tracks exercise progress through defined states:
  - WAITING_FOR_START: Waiting for correct starting position
  - READY_FOR_END: Moving to end position
  - HOLDING: Maintaining a specific position
- Joint locking system:
  - Locks joints when they reach target angles
  - 5-second timeout for losing position
  - Requires all active joints to be locked to progress

## User Interface

### 1. Video Display
- Real-time webcam feed
- Selective pose skeleton overlay:
  - Only shows joints relevant to current exercise stage
  - Required joints shown in primary color
  - Optional joints shown in secondary color
  - Connected joints shown in tertiary color
  - Non-exercise joints hidden
- Joint angle visualizations only for tracked angles
- Rep counter
- State indicator

### 2. Exercise Controls
- Exercise selector dropdown
- Stage progress indicator
- Joint toggle switches
- Angle range controls for each joint
- Global detection tolerance slider

### 3. Debug Information
- Current state display
- Transition conditions
- Joint angles and lock status
- Hold duration countdown (for hold stages)

## Technical Requirements

### Hardware
- Webcam
- Modern web browser (Chrome recommended)
- Sufficient CPU/GPU for real-time pose detection

### Network
- Initial download of TensorFlow.js (~2MB)
- One-time download of MoveNet model (~6MB)
- Local HTTP server for development

### Browser Requirements
- WebGL support
- WebRTC (for webcam access)
- JavaScript enabled
- Minimum resolution: 640x480

## Data Structure

### Exercise Schema
```json
{
  "exercise_name": [
    {
      "name": "Stage Name",
      "type": "movement|hold",
      "movements": {
        "joint_name": {
          "start": angle,
          "end": angle
        }
      },
      "hold": {
        "duration": seconds,
        "joints": {
          "joint_name": {
            "angle": degrees
          }
        }
      }
    }
  ]
}
```

### Joint Configuration
- Each joint has:
  - Active state (enabled/disabled)
  - Current angle
  - Smoothed angle
  - Lock status
  - Last valid position timestamp
  - Color coding for visualization

## Performance Considerations
- Frame rate target: 30 FPS
- Pose detection latency: <100ms
- Angle calculation smoothing: 0.85 factor
- Position smoothing: 0.7 factor
- Debounce time for rep counting: 500ms

## Known Limitations
- Single person detection only
- Requires good lighting conditions
- Limited to exercises visible from a single camera angle
- May have reduced accuracy with loose clothing
- Requires clear view of all tracked joints 