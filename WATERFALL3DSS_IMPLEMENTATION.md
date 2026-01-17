# Waterfall 3DSS Implementation in deskHPSDR

## Summary
This document describes the implementation of the Yaesu 3DSS-style 3D waterfall in deskHPSDR, maintaining full compatibility with the existing 2D waterfall. The implementation is **COMPLETE** and includes advanced features like RxPGA-based auto-threshold adjustment, multiple color palettes, and interactive controls.

## Implementation Status: ✅ COMPLETE

All features have been implemented and are working:
- ✅ 3D OpenGL waterfall with Yaesu-style display
- ✅ Dynamic mode switching (2D ↔ 3DSS)
- ✅ RxPGA-based auto-threshold adjustment
- ✅ 7 color palettes with selection in display menu
- ✅ Interactive controls (tilt, zoom)
- ✅ Settings persistence
- ✅ Brightness/contrast optimization

## Created Files

### 1. waterfall3dss.c
Complete 3D OpenGL waterfall implementation with:
- OpenGL 3.3+ rendering using GLSL shaders (3.2 for macOS, 3.3+ for Linux)
- Yaesu-style 3D display with grid and perspective
- **7 color palettes**: Rainbow, Ocean, Green, Gray, Hot, Cool, Plasma
- Interactive controls (mouse drag for tilt, scroll for zoom)
- Spectrum history management with 120 lines of depth
- Support for pan/zoom and frequency changes
- **RxPGA auto-threshold adjustment**: Automatically compensates for gain changes
- **Brightness optimization**: 50% base threshold with 2.5x brightness boost
- **Stabilization system**: Tracks render/update cycles for smooth operation

### 2. waterfall3dss.h
Header file with public function declarations:
- `waterfall3dss_init()` - Initializes the 3D waterfall
- `waterfall3dss_update()` - Updates with new spectrum data
- OpenGL callbacks (realize, unrealize, render)

## Advanced Features

### RxPGA Auto-Threshold Adjustment
The waterfall automatically adjusts its display threshold when RxPGA (receiver gain) changes:

```c
typedef struct {
  // ... other fields ...
  
  // Auto-adjust threshold based on RxPGA changes
  int last_alex_attenuation;  // Track attenuation changes
  int last_preamp;            // Track preamp changes
  float base_threshold;       // Dynamically adjusted threshold (20%-80%)
} WaterfallGLState;
```

**How it works:**
1. Detects changes in `alex_attenuation` (steps of -10dB) or `preamp` (steps of +18dB)
2. Calculates gain delta in dB
3. Adjusts threshold proportionally to maintain consistent visual sensitivity
4. Prevents "breathing" effect - only adjusts on manual RxPGA changes, not continuously

**Example:**
- RxPGA at 18dB: threshold ~50%, shows strong signals only
- Increase to 38dB (+20dB): threshold auto-adjusts to ~70%
- Visual appearance remains consistent despite gain change

### Brightness and Contrast Optimization
Current color mapping strategy:

```c
const float noise_threshold = 0.50f;  // Base: 50% of dynamic range

if (p < noise_threshold) {
  // Below threshold: black background
  *r = *g = *b = 0.0f;
  *h01 = 0.0f;  // Flat (no height)
} else {
  // Above threshold: high brightness signals
  float signal = (p - threshold) / (1.0f - threshold);
  
  // Aggressive brightness boost
  float brightness = 0.6f + (signal * 2.5f);  // Range: 0.6 to 3.1x
  brightness = clamp(brightness, 0.6f, 3.0f);
  
  *r *= brightness;
  *g *= brightness;
  *b *= brightness;
}
```

**Key parameters:**
- **Base threshold**: 50% (adjusts with RxPGA)
- **Minimum brightness**: 0.6x (for weak signals above threshold)
- **Maximum brightness**: 3.0x (for strong signals)
- **Height scaling**: 1.8x for strong signals

### Color Palettes
7 palettes available via display menu:
0. **Rainbow**: Full spectrum transition
1. **Ocean**: Blue/cyan tones
2. **Green**: Monochrome green (classic radar)
3. **Gray**: Monochrome grayscale
4. **Hot**: Black → Red → Yellow → White
5. **Cool**: Black → Blue → Cyan → White
6. **Plasma**: White → Blue → Lilac → Red (default)

Each palette includes depth-based color variation for 3D effect.

## Required Changes (✅ ALL IMPLEMENTED)

### 1. receiver.h
Add fields for waterfall mode selection and palette:

```c
typedef struct _receiver {
  // ... existing fields ...
  
  int waterfall_low;
  int waterfall_high;
  int waterfall_automatic;
  int waterfall_mode;              // 0 = 2D (Cairo), 1 = 3DSS (OpenGL)
  int last_waterfall_mode;         // Track mode changes for dynamic switching
  int waterfall3dss_palette;       // Color palette: 0-6
  
  cairo_surface_t *panadapter_surface;
  GdkPixbuf *pixbuf;
  // ... rest of fields ...
} RECEIVER;
```

### 2. receiver.c
Add initialization and mode switching:

```c
#include "waterfall.h"
#include "waterfall3dss.h"  // NEW

RECEIVER *rx_create_receiver(int id, int pixels, int width, int height) {
  // ... existing code ...
  
  rx->waterfall_low = -100;
  rx->waterfall_high = -40;
  rx->waterfall_automatic = 0;
  rx->waterfall_mode = 0;  // NEW: Initialize in 2D mode by default
  
  // ... rest of initialization ...
}

void rx_reconfigure(RECEIVER *rx, int height) {
  // ... existing code ...
  
  if(rx->display_waterfall) {
    // Check if we need to recreate waterfall due to mode change
    int need_recreate = (rx->waterfall != NULL && 
                        rx->last_waterfall_mode != rx->waterfall_mode);
    
    if (rx->waterfall == NULL || need_recreate) {
      if (need_recreate) {
        // Remove old waterfall before creating new one
        gtk_container_remove(GTK_CONTAINER(rx->panel), rx->waterfall);
        rx->waterfall = NULL;
      }
      
      // Create waterfall in selected mode
      if (rx->waterfall_mode == 1) {
        waterfall3dss_init(rx, rx->width, myheight);
      } else {
        waterfall_init(rx, rx->width, myheight);
      }
      
      gtk_fixed_put(GTK_FIXED(rx->panel), rx->waterfall, 0, y);
      rx->last_waterfall_mode = rx->waterfall_mode;
    }
    // ... rest of code ...
  }
}
```

### 3. rx_panadapter.c (or wherever waterfall_update is called)
Modify to call the appropriate function:

```c
static gint update_display(gpointer data) {
  RECEIVER *rx = (RECEIVER *)data;
  
  if (rx->displaying) {
    if (rx->pixels > 0) {
      g_mutex_lock(&rx->display_mutex);
      GetPixels(rx->id, 0, rx->pixel_samples, &rc);
      if (rc) {
        if (rx->display_panadapter) {
          rx_panadapter_update(rx);
        }
        if (rx->display_waterfall) {
          // MODIFIED: Call appropriate function
          if (rx->waterfall_mode == 1) {
            waterfall3dss_update(rx);
          } else {
            waterfall_update(rx);
          }
        }
      }
      g_mutex_unlock(&rx->display_mutex);
    }
  }
  return TRUE;
}
```

### 4. display_menu.c (or configuration menu)
Add options to toggle mode and select palette:

```c
static void waterfall_mode_cb(GtkWidget *widget, gpointer data) {
  active_receiver->waterfall_mode = gtk_toggle_button_get_active(GTK_TOGGLE_BUTTON(widget)) ? 1 : 0;
  radio_reconfigure();  // Triggers rx_reconfigure with dynamic switching
}

static void waterfall3dss_palette_cb(GtkWidget *widget, gpointer data) {
  int new_palette = gtk_combo_box_get_active(GTK_COMBO_BOX(widget));
  if (active_receiver->waterfall3dss_palette != new_palette) {
    active_receiver->waterfall3dss_palette = new_palette;
    // No need to recreate waterfall, just redraw with new palette
  }
}

// In display menu:
GtkWidget *waterfall_mode_switch = gtk_check_button_new_with_label("Waterfall 3DSS");
gtk_toggle_button_set_active(GTK_TOGGLE_BUTTON(waterfall_mode_switch), 
                              active_receiver->waterfall_mode);
g_signal_connect(waterfall_mode_switch, "toggled", G_CALLBACK(waterfall_mode_cb), NULL);

// Palette selection combo box
GtkWidget *palette_combo = gtk_combo_box_text_new();
gtk_combo_box_text_append_text(GTK_COMBO_BOX_TEXT(palette_combo), "Rainbow");
gtk_combo_box_text_append_text(GTK_COMBO_BOX_TEXT(palette_combo), "Ocean");
gtk_combo_box_text_append_text(GTK_COMBO_BOX_TEXT(palette_combo), "Green");
gtk_combo_box_text_append_text(GTK_COMBO_BOX_TEXT(palette_combo), "Gray");
gtk_combo_box_text_append_text(GTK_COMBO_BOX_TEXT(palette_combo), "Hot");
gtk_combo_box_text_append_text(GTK_COMBO_BOX_TEXT(palette_combo), "Cool");
gtk_combo_box_text_append_text(GTK_COMBO_BOX_TEXT(palette_combo), "Plasma");
gtk_combo_box_set_active(GTK_COMBO_BOX(palette_combo), active_receiver->waterfall3dss_palette);
g_signal_connect(palette_combo, "changed", G_CALLBACK(waterfall3dss_palette_cb), NULL);
```

### 5. property.c (save/restore settings)
Add support for saving waterfall mode:

```c
void rx_save_state(const RECEIVER *rx) {
  // ... existing code ...
  
  sprintf(name, "receiver.%d.waterfall_mode", rx->id);
  sprintf(value, "%d", rx->waterfall_mode);
  setProperty(name, value);
  
  // ... rest of code ...
}

void rx_restore_state(RECEIVER *rx) {
  // ... existing code ...
  
  sprintf(name, "receiver.%d.waterfall_mode", rx->id);
  value = getProperty(name);
  if (value) rx->waterfall_mode = atoi(value);
  
  // ... rest of code ...
}
```

### 6. Makefile
Add waterfall3dss to files to compile:

```makefile
OBJS = main.o \
       receiver.o \
       waterfall.o \
       waterfall3dss.o \
       # ... other files ...
```

## Dependencies

### Required libraries (already present in deskhpsdr):
- GTK+3 with GtkGLArea support
- OpenGL 3.3+ (via libepoxy)
- Cairo (for 2D waterfall)

### Compilation:
```bash
gcc -c waterfall3dss.c `pkg-config --cflags gtk+-3.0 epoxy`
```

## Interactive Controls (3DSS mode only)

### Mouse:
- **Drag (left button)**: Adjusts 3D display tilt
- **Scroll**: Adjusts zoom (camera distance)

### Adjustable parameters:
- `WATERFALL_DEPTH`: Number of history lines (default: 120)
- `WATERFALL_Z_SPAN`: Display depth in 3D units (default: 1.60)
- Tilt angle: 0.0 to 5.0 (adjustable via mouse)
- Zoom level: 1.0 to 4.0 (adjustable via scroll)

## Differences between 2D and 3DSS

### 2D Waterfall (Cairo - current):
- ✅ Fast and lightweight rendering
- ✅ Works on any hardware
- ✅ Traditional flat display
- ❌ No 3D perspective
- ❌ No interactive viewing controls

### 3DSS Waterfall (OpenGL - new):
- ✅ Yaesu FT-dx10 style 3D display
- ✅ Visual perspective and depth
- ✅ Interactive controls (tilt/zoom)
- ✅ 3D grid for reference
- ✅ Advanced color palettes
- ⚠️ Requires OpenGL 3.3+
- ⚠️ Higher GPU usage

## Data Flow

```
WDSP/Radio → rx->pixel_samples[] 
           ↓
waterfall_mode == 0 → waterfall_update() → Cairo 2D
           ↓
waterfall_mode == 1 → waterfall3dss_update() → OpenGL 3D
           ↓
          Display in GTK widget (waterfall)
```

## Testing

1. **Compile** with the new changes
2. **Start** deskHPSDR normally (2D mode by default)
3. **Display Menu** → Select "Waterfall 3DSS"
4. **Verify** 3D rendering
5. **Test** controls (drag/scroll)
6. **Switch** back to 2D

## Compatibility

- ✅ Maintains 100% compatibility with existing 2D waterfall
- ✅ Does not break existing functionality
- ✅ 2D mode remains default
- ✅ Users can freely choose between modes
- ✅ Configuration automatically saved/restored

## Implementation Notes

1. 3D waterfall uses GtkGLArea (GTK3), no need to create separate OpenGL window
2. Spectrum history is managed in circular buffer (ring buffer)
3. Frequency rotation (VFO changes) is handled by shifting history horizontally
4. AGC stabilization: first 5 updates are ignored to avoid artifacts
5. Mutex (`display_mutex`) protects concurrent access to spectrum data

## Implementation Completed ✅

1. ✅ Create waterfall3dss.c and waterfall3dss.h
2. ✅ Modify receiver.h to add `waterfall_mode`, `last_waterfall_mode`, `waterfall3dss_palette`
3. ✅ Update receiver.c to support both modes with dynamic switching
4. ✅ Add toggle and palette selector in display menu
5. ✅ Implement save/restore of mode and palette
6. ✅ Update Makefile
7. ✅ Test and optimize visual parameters (brightness, contrast, threshold)
8. ✅ Implement RxPGA auto-threshold adjustment
9. ✅ Add 7 color palettes
10. ✅ Test on Linux (primary target platform)

## Conclusion

The waterfall 3DSS implementation in deskHPSDR is **COMPLETE** and maintains full compatibility with existing code. Users can choose between:

### 2D Waterfall (Default)
- Lightweight Cairo-based rendering
- Compatible with any hardware
- Traditional flat display
- Lower GPU usage

### 3DSS Waterfall (Yaesu-style)
- OpenGL 3.3+ rendering (3.2 on macOS)
- Yaesu FT-dx10 inspired 3D perspective
- 7 color palettes
- Interactive controls (drag tilt, scroll zoom)
- RxPGA auto-threshold adjustment
- Optimized brightness/contrast
- Higher visual fidelity

**Note:** This fork focuses exclusively on **LINUX** systems, as macOS does not support GTK3. There are no plans to migrate to GTK4 or implement alternative solutions at this time.

### Latest Improvements (Jan 2026)
- Enhanced brightness: 50% base threshold with 2.5x boost
- RxPGA-based auto-threshold: maintains consistent display regardless of gain settings
- Optimized color mapping: black background for noise, vibrant colors for signals
- Stabilization system: smooth rendering with minimal artifacts

Users can switch between modes at any time through the display menu. Settings are automatically saved and restored.
