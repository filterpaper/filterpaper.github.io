# Standalone Userspace

One of the least obvious way of building QMK firmware is using the json file exported out of the [configurator](https://config.qmk.fm/) with userspace. This method is favored personally for the following advantage:

* Simplified file maintenance with everything inside userspace—avoids deep source tree like `keyboards/kbdfans/kbd67/mkiirgb/keymaps`.
* Skip `keymap.c` conversion step with `qmk json2c`. 
* Eliminates onerous `keymap.c` maintenance with text editors.
* Easy to extend support for additional keyboards in the same space.


# First setup 
(Prerequisite: [QMK build environment](https://docs.qmk.fm/#/newbs_getting_started) must be installed correctly before proceeding.) The [QMK userspace](https://docs.qmk.fm/#/feature_userspace) should be your github name—"newbie" in this example, created inside `qmk_firmware/users/`:
```
mkdir ~/qmk_firmware/users/newbie/
```
Visit the [configurator](https://config.qmk.fm/) to customise a key map layout for your keyboard. It is important for the "key map name" to match the userpace, "newbie" in this example. 

Export the key layout and it will be saved as `newbie.json` by default. Move the file into the userspace folder if it was not saved in there:
```
mv newbie.json ~/qmk_firmware/users/newbie/
```


# Compiling first default firmware
The space is now setup to compile with default setting. Simply run `qmk compile` on that .json file. Important: the command expects a full path, even if executed in the current folder:
```
qmk compile ~/qmk_firmware/users/newbie/newbie.json
```
If everything goes well, it will build a firmware with default settings from the keyboard source.


# Customising the firmware
You can start customising the firmware with code files saved inside the userspace. See QMK guide on [customising keyboard behavior](https://docs.qmk.fm/#/custom_quantum_functions) and follow the file naming conventions of [QMK userspace](https://docs.qmk.fm/#/feature_userspace).

Hardware features and QMK variables should configured in these files that will be a picked up automatically by the build process:
```
~/qmk_firmware/users/newbie/rules.mk
~/qmk_firmware/users/newbie/config.h
```
Instead `keymap.c`, programming codes should be added into your own `<name>.c` like:
```
~/qmk_firmware/users/newbie/newbie.c
```
QMK compiler must be informed of this file with the following line inside `rules.mk`:
```c
SRC += newbie.c
```

Customisation in all 3 files will be picked up by the `qmk compile ~/qmk_firmware/users/newbie/newbie.json` build command.


# Supporting more than one keyboard
Additional keyboards can be configured in the same userspace with the following guideline:a
* Provide distinct names for each keyboard's .json like `<keyboard-name>.json`.
* Liberal use of `#ifdef` to block out code sections for keyboard-specific features.

## rules.mk
QMK features can be enabled or disabled for specific hardware with define blocks:
```c
# Common feature for all keyboars
BOOTMAGIC_ENABLE = yes
EXTRAKEY_ENABLE = yes
RGBLIGHT_ENABLE = no
RGB_MATRIX_ENABLE = no

# RGB matrix lights for BM40
ifeq ($(strip $(KEYBOARD)), bm40hsrgb)
    RGB_MATRIX_ENABLE = yes
    RGB_MATRIX_CUSTOM_USER = yes
endif

# Split keyboard feature for Corne
ifeq ($(strip $(KEYBOARD)), crkbd/rev1/common)
    WPM_ENABLE = yes
    MOUSEKEY_ENABLE = yes
    OLED_DRIVER_ENABLE = yes
endif

SRC += newbie.c
```

## config.h
QMK variables can likewise be selectively configured inside define blocks:
```c
#pragma once

// Common QMK variables
#define TAPPING_TERM 250
#define PERMISSIVE_HOLD
#define IGNORE_MOD_TAP_INTERRUPT
#define LEADER_TIMEOUT 500
#define LEADER_PER_KEY_TIMING
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

## <name>.c
Judicious use of `#ifdef` blocks in the source is recommended to include and exclude relevant section, matching those in `rules.mk` and `config.h`. Example:
```c
#ifdef RGB_MATRIX_ENABLE
void matrix_init_user(void) {
    rgb_matrix_sethsv_noeeprom(HSV_OFF);
    rgb_matrix_mode_noeeprom(RGB_MATRIX_SOLID_COLOR);
}
#endif

#ifndef KEYBOARD_bm40hsrgb
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

# Summary
Maintaining the userspace in this manner will keep your code files tidy in one location instead of scattering them all over the QMK source tree. Backup and git commits will be centralized to one folder. However it is not without some drawbacks:
* Reliance on QMK Configurator to modify key map layout
* .json files are illegible to inspect
