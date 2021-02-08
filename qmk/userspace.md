# Standalone Userspace

One of the least obvious ways of building QMK firmware is using the json file exported out of the [Configurator](https://config.qmk.fm/) with [userspace](https://docs.qmk.fm/#/feature_userspace). This method is favored personally for the following advantages:

* Simplified file management with everything in one folder—avoids deep source tree like `keyboards/kbdfans/kbd67/mkiirgb/keymaps`.
* Skips `keymap.c` conversion with `qmk json2c`. 
* Avoids onerous text editing of `keymaps[]` in `keymap.c`.
* Easy to extend support for additional keyboards.


# Initial space setup
Prerequisite: [QMK build environment](https://docs.qmk.fm/#/newbs_getting_started) must be installed correctly before proceeding. The [QMK userspace](https://docs.qmk.fm/#/feature_userspace) should be your github name. Create that inside `qmk_firmware/users/`:
```
mkdir ~/qmk_firmware/users/newbie/
```
Visit the [Configurator](https://config.qmk.fm/) to customise a key map layout for your keyboard. It is important for the Configurator's *KEYMAP NAME* field to match userspace, "newbie" in this example. 

Export that layout and it will be saved as `newbie.json` by default. Move the file into the userspace folder if not already saved inside:
```
mv newbie.json ~/qmk_firmware/users/newbie/
```


# Compiling a default firmware
Building a firmware is as simple as running `qmk compile` on that .json file:
```
qmk compile ~/qmk_firmware/users/newbie/newbie.json
```
If everything goes well, it will build a firmware with default settings using your custom key layout.


# Customising the firmware
See QMK guide on [customising keyboard behavior](https://docs.qmk.fm/#/custom_quantum_functions) and follow the file naming convention of [QMK userspace](https://docs.qmk.fm/#/feature_userspace).

Hardware feature and QMK variables should be configured in `rules.mk` and `config.h`:
```
~/qmk_firmware/users/newbie/rules.mk
~/qmk_firmware/users/newbie/config.h
```
Instead of `keymap.c`, programming codes should be saved in `<name>.c` like:
```
~/qmk_firmware/users/newbie/newbie.c
```
Unlike `keymap.c`, the `keymaps[]` array is excluded from this file because key layout is stored in the .json file. Do add the following line into `rules.mk` for QMK build process to find it during compile time:
```c
SRC += newbie.c
```
When you are done, userspace will have these 4 basic files:
```
~$ tree qmk_firmware/users/newbie/
qmk_firmware/users/newbie/
|-- config.h
|-- newbie.c
|-- newbie.json
`-- rules.mk

0 directories, 4 files
```
The `qmk compile ~/qmk_firmware/users/newbie/newbie.json` process will automatically include everything in that folder. Except for `newbie.json`, all files are optional. The .c source file is not required for example if you just simply need some QMK variables changed inside `config.h`.


# Supporting multiple keyboard
Additional keyboards can be configured in the same userspace with the following guideline:
* Use distinct keyboard file names for each .json like `<keyboard-name>.json`.
* Use `#ifdef` conditional preprocessors for keyboard-specific codes.

The following are some examples.

## rules.mk
QMK features can be enabled exclusively for specific hardware with `ifeq` blocks:
```c
# Common feature for all keyboards
BOOTMAGIC_ENABLE = yes
EXTRAKEY_ENABLE = yes
RGBLIGHT_ENABLE = no
RGB_MATRIX_ENABLE = no

# RGB matrix lights for BM40
ifeq ($(strip $(KEYBOARD)), bm40hsrgb)
    RGB_MATRIX_ENABLE = yes
    RGB_MATRIX_CUSTOM_USER = yes
endif

# Split features for Corne
ifeq ($(strip $(KEYBOARD)), crkbd/rev1/common)
    WPM_ENABLE = yes
    MOUSEKEY_ENABLE = yes
    OLED_DRIVER_ENABLE = yes
endif

SRC += newbie.c
```

## config.h
QMK variables can likewise be selectively configured inside conditional `#ifdef` blocks:
```c
#pragma once

// Common QMK variables
#define TAPPING_TERM 250
#define PERMISSIVE_HOLD
#define IGNORE_MOD_TAP_INTERRUPT
#define TAP_CODE_DELAY 10

#ifdef RGB_MATRIX_ENABLE
#  define RGB_MATRIX_KEYPRESSES
#  define RGB_DISABLE_WHEN_USB_SUSPENDED true
#  define RGB_MATRIX_MAXIMUM_BRIGHTNESS 100
#endif

#ifdef OLED_DRIVER_ENABLE
#  define OLED_TIMEOUT 5000
#endif
```

## newbie.c
Judicious use of `#ifdef` conditions in the source is recommended for relevant sections, matching those in `rules.mk` and `config.h`. This will exclude code meant only for specific keyboards:
```c
// Init effect for RGB boards only
#ifdef RGB_MATRIX_ENABLE
void matrix_init_user(void) {
    rgb_matrix_mode_noeeprom(RGB_MATRIX_SOLID_COLOR);
}
#endif

// Leader key feature for all boards
LEADER_EXTERNS();
void matrix_scan_user(void) {
    LEADER_DICTIONARY() {
        leading = false;
        leader_end();
        SEQ_ONE_KEY(KC_P) { SEND_STRING("()"); }
        SEQ_ONE_KEY(KC_B) { SEND_STRING("{}"); }
    }
}

// Default layer effects for BM40 only
#ifdef KEYBOARD_bm40hsrgb
layer_state_t layer_state_set_user(layer_state_t state) {

    // Default layer keypress effects
    switch (get_highest_layer(default_layer_state)) {
    case 1:
        rgb_matrix_mode_noeeprom(RGB_MATRIX_BREATHING);
        break;
    case 0:
        rgb_matrix_mode_noeeprom(RGB_MATRIX_RAINDROPS);
        break;
    }
    return state;
}
#endif // KEYBOARD_bm40hsrgb
```

## Collectively
When you're done, the shared userspace for multiple keyboards will be in a neat structure:
```
~$ tree qmk_firmware/users/newbie/
qmk_firmware/users/newbie/
|-- bm40rgb.json
|-- config.h
|-- crkbd.json
|-- newbie.c
`-- rules.mk

0 directories, 5 files
```
Rebuilding each keyboard's firmware is as simple as:
```
qmk compile ~/qmk_firmware/users/newbie/bm40rgb.json
qmk compile ~/qmk_firmware/users/newbie/crkbd.json
```


# Summary
Maintaining personal build environment this way will keep code files tidy in one location instead of scattering them all over the QMK source tree. However it is not without some drawbacks:
* Dependency on QMK Configurator to modify key map layout
* .json files are *illegible* for casual inspection
