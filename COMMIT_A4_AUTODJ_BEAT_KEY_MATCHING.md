# Commit A4: Auto DJ - Beat and Key Matching Prerequisites

## Status: 📋 Prepared (Ready for Implementation)

## Summary
Ensure intelligent crossfading only happens when tracks are properly synced:
- Beat/tempo alignment
- Key/harmonic compatibility

If conditions aren't met, fall back to traditional crossfade.

## Prerequisites for Intelligent Crossfade

### 1. Beat Matching
- Both tracks must have valid BPM
- Tempo should be within sync range (adjustable via rate slider)
- Beats must be aligned at crossfade point

### 2. Key Matching (Optional but Recommended)
- Keys should be harmonically compatible
- Use Camelot wheel / Open Key notation rules

## Files to Create/Modify

### 1. Create TransitionValidator Class

**`src/library/autodj/transitionvalidator.h`**
```cpp
#pragma once

#include "track/track_decl.h"

class TransitionValidator {
  public:
    struct ValidationResult {
        bool beatMatchPossible;
        bool keyCompatible;
        bool canUseIntelligentCrossfade;
        QString reason;  // Why it failed, if applicable
    };

    static ValidationResult validate(
        TrackPointer outgoingTrack,
        TrackPointer incomingTrack,
        double maxBpmDifference = 8.0);  // Default: ±8% range

    // Individual checks
    static bool checkBeatMatch(
        TrackPointer track1,
        TrackPointer track2,
        double maxBpmDifference);

    static bool checkKeyCompatibility(
        TrackPointer track1,
        TrackPointer track2);

  private:
    static bool areKeysCompatible(
        mixxx::track::io::key::ChromaticKey key1,
        mixxx::track::io::key::ChromaticKey key2);
};
```

### 2. Implementation

**`src/library/autodj/transitionvalidator.cpp`**
```cpp
#include "library/autodj/transitionvalidator.h"
#include "track/track.h"
#include "track/keyutils.h"

TransitionValidator::ValidationResult TransitionValidator::validate(
        TrackPointer outgoingTrack,
        TrackPointer incomingTrack,
        double maxBpmDifference) {

    ValidationResult result;
    result.beatMatchPossible = true;
    result.keyCompatible = true;
    result.canUseIntelligentCrossfade = true;

    // Check beat matching
    if (!checkBeatMatch(outgoingTrack, incomingTrack, maxBpmDifference)) {
        result.beatMatchPossible = false;
        result.canUseIntelligentCrossfade = false;
        result.reason = "BPM difference too large for sync";
    }

    // Check key compatibility (warning only, doesn't block)
    if (!checkKeyCompatibility(outgoingTrack, incomingTrack)) {
        result.keyCompatible = false;
        // Don't block intelligent crossfade, just warn
        result.reason += result.reason.isEmpty() ? "" : "; ";
        result.reason += "Keys not harmonically compatible";
    }

    return result;
}

bool TransitionValidator::checkBeatMatch(
        TrackPointer track1,
        TrackPointer track2,
        double maxBpmDifference) {

    double bpm1 = track1->getBpm();
    double bpm2 = track2->getBpm();

    // Both must have valid BPM
    if (bpm1 <= 0 || bpm2 <= 0) {
        return false;
    }

    // Calculate percentage difference
    double ratio = bpm2 / bpm1;

    // Allow for half-time / double-time (ratio near 0.5, 1.0, or 2.0)
    double normalizedRatio = ratio;
    if (ratio < 0.75) {
        normalizedRatio = ratio * 2.0;  // Half-time
    } else if (ratio > 1.5) {
        normalizedRatio = ratio / 2.0;  // Double-time
    }

    // Check if within range
    double percentDiff = std::abs(normalizedRatio - 1.0) * 100.0;
    return percentDiff <= maxBpmDifference;
}

bool TransitionValidator::checkKeyCompatibility(
        TrackPointer track1,
        TrackPointer track2) {

    auto key1 = KeyUtils::keyFromNumericValue(track1->getKey());
    auto key2 = KeyUtils::keyFromNumericValue(track2->getKey());

    // If either key is invalid/unknown, allow transition
    if (key1 == mixxx::track::io::key::INVALID ||
        key2 == mixxx::track::io::key::INVALID) {
        return true;
    }

    return areKeysCompatible(key1, key2);
}

bool TransitionValidator::areKeysCompatible(
        mixxx::track::io::key::ChromaticKey key1,
        mixxx::track::io::key::ChromaticKey key2) {

    // Use KeyUtils for harmonic compatibility check
    // Compatible keys: same key, relative major/minor, ±1 on Camelot wheel

    if (key1 == key2) {
        return true;  // Same key
    }

    // Check if harmonically compatible (within 1 step on Camelot wheel)
    int steps = KeyUtils::shortestStepsToCompatibleKey(key1, key2);
    return std::abs(steps) <= 1;
}
```

## Camelot Wheel Reference

```
        MINOR              MAJOR
         (A)                (B)

    5A = F minor      5B = Ab major
    6A = C minor      6B = Eb major
    7A = G minor      7B = Bb major
    8A = D minor      8B = F major
    9A = A minor      9B = C major
   10A = E minor     10B = G major
   11A = B minor     11B = D major
   12A = F# minor    12B = A major
    1A = Db minor     1B = E major
    2A = Ab minor     2B = B major
    3A = Eb minor     3B = F# major
    4A = Bb minor     4B = Db major

Compatible moves:
- Same number (8A ↔ 8B) = Relative major/minor
- ±1 number, same letter (8A ↔ 7A, 9A) = Energy shift
- Same number + letter = Perfect match
```

## Beat Alignment at Crossfade Point

### Quantize to Beat Grid

```cpp
double quantizeToBeat(double position, TrackPointer track) {
    const auto& beats = track->getBeats();
    if (!beats) {
        return position;
    }

    // Find nearest beat to position
    mixxx::audio::FramePos frame =
        mixxx::audio::FramePos(position * track->getDuration() *
                               track->getSampleRate());

    auto nearestBeat = beats->findClosestBeat(frame);
    if (nearestBeat.isValid()) {
        return nearestBeat.value() /
               (track->getDuration() * track->getSampleRate());
    }

    return position;
}
```

### Align Incoming Track

```cpp
void alignIncomingTrack(
        TrackPointer outgoing,
        TrackPointer incoming,
        double crossfadeStart) {

    // Get beat positions
    const auto& outgoingBeats = outgoing->getBeats();
    const auto& incomingBeats = incoming->getBeats();

    if (!outgoingBeats || !incomingBeats) {
        return;  // Can't align without beat grids
    }

    // Find phase difference and adjust incoming start position
    // ... (complex beat alignment logic)
}
```

## Fallback Behavior

When validation fails:
```cpp
void AutoDJProcessor::executeTransition() {
    auto validation = TransitionValidator::validate(
        m_currentTrack, m_nextTrack);

    if (validation.canUseIntelligentCrossfade) {
        executeIntelligentCrossfade();
    } else {
        // Fallback to traditional fixed-duration crossfade
        qInfo() << "Falling back to standard crossfade:"
                << validation.reason;
        executeStandardCrossfade();
    }
}
```

## User Configuration

### Settings in Auto DJ Preferences
```cpp
// Require beat match for intelligent crossfade
bool m_bRequireBeatMatch = true;

// Require key compatibility (stricter mode)
bool m_bRequireKeyMatch = false;

// Maximum BPM difference for sync
double m_maxBpmDifference = 8.0;  // ±8%
```

## Testing

### Test Cases

1. **Same BPM, same key**
   - Should allow intelligent crossfade ✓

2. **Same BPM, incompatible keys**
   - Should warn but allow (if not strict) ✓

3. **BPM within range (±8%)**
   - Should allow intelligent crossfade ✓

4. **BPM too different (>8%)**
   - Should fall back to standard crossfade

5. **Half-time/double-time (80 BPM → 160 BPM)**
   - Should recognize and allow sync ✓

6. **Missing BPM data**
   - Should fall back to standard crossfade

## Dependencies
- Uses: `src/track/keyutils.h` for key compatibility
- Uses: `src/track/beats.h` for beat grid access
- Used by: Commit A5 (Auto DJ Integration)
