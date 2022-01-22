# Standalone Userspace

One of the least obvious ways of building QMK firmware is using a json file with [userspace](https://docs.qmk.fm/#/feature_userspace). This method is favored personally for the following advantages:

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
If everything goes well, it will build a firmware with default settings using your custom key layout. You may also run `qmk flash` instead on the .json file for QMK to follow with flashing the keyboard after a successful build.


# Customising the firmware
There are many customisation options with QMK and you can start with the [custom quantum function page](https://docs.qmk.fm/#/custom_quantum_functions). The file naming conventions of [QMK userspace](https://docs.qmk.fm/#/feature_userspace) is the guide for organising this folder.

Hardware feature and QMK variables are placed in `rules.mk` and `config.h`. Both files should be created in the same folder:
```
~/qmk_firmware/users/newbie/rules.mk
~/qmk_firmware/users/newbie/config.h
```
Instead of `keymap.c`, custom programming codes should be saved in `<name>.c` like:
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
The `qmk compile ~/qmk_firmware/users/newbie/newbie.json` process will automatically include everything in that folder. Except for `newbie.json`, all files are optional. You can have just `config.h` for modifying QMK variables or just `rules.mk` for enabling one feature.


# Supporting multiple keyboard
Additional keyboards can be configured in the same userspace with the following guideline:
* Use distinct keyboard file names for each .json like `<keyboard-name>.json`.
* Use `#ifdef` conditional preprocessors for keyboard-specific codes.

The following are some examples.

## rules.mk
QMK features can be enabled exclusively for specific hardware with `ifeq` blocks:
```make
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
Rebuilding and flashing each keyboard's firmware is as simple as:
```
qmk flash ~/qmk_firmware/users/newbie/bm40rgb.json
qmk flash ~/qmk_firmware/users/newbie/crkbd.json
```

# Advance wrapper layout
Exported key map from the [Configurator](https://config.qmk.fm/) may not be favorable to power users that prefers editing layouts in `keymap.c` text format. Key map wrapper can be adapted to use json file.
## planck.json
Instead of exporting from the Configurator, create a `planck.json` file with *macro* names that make sense for each layer:
```json
{
    "author": "",
    "documentation": "Wrapper based keymap",
    "keyboard": "planck/rev6",
    "keymap": "filterpaper",
    "layers": [
        [ "BASE" ],
        [ "LOWER" ],
        [ "RAISE" ],
        [ "ADJUST" ]
    ],
    "layout": "LAYOUT_wrapper_ortho_4x12",
    "notes": "",
    "version": 1
}
```
## wrappers.h
Create a `wrappers.h` file to map those macro names to actual key layout (like a typical `keymap.c`):
```
#pragma once
#define LAYOUT_wrapper_ortho_4x12(...) LAYOUT_ortho_4x12(__VA_ARGS__)

// QWERTY
#define BASE \
KC_TAB,  KC_Q,    KC_W,    KC_E,    KC_R,    KC_T,    KC_Y,    KC_U,    KC_I,    KC_O,    KC_P,    KC_BSPC, \
KC_GESC, KC_A,    KC_S,    KC_D,    KC_F,    KC_G,    KC_H,    KC_J,    KC_K,    KC_L,    KC_SCLN, KC_QUOT, \
KC_LSFT, KC_Z,    KC_X,    KC_C,    KC_V,    KC_B,    KC_N,    KC_M,    KC_COMM, KC_DOT,  KC_SLSH, KC_ENT, \
KC_DEL,  KC_LALT, KC_LCTL, KC_LGUI, MO(1),   KC_SPC,  KC_SPC,  MO(2),   KC_LEFT, KC_DOWN, KC_UP,   KC_RGHT

// Lower
#define LOWER \
_______, _______, _______, KC_LPRN, KC_RPRN, _______, _______, KC_MINS, KC_EQL,  KC_BSLS, _______, _______, \
_______, _______, _______, KC_LCBR, KC_RCBR, _______, KC_LEFT, KC_DOWN, KC_UP,   KC_RGHT, _______, _______, \
KC_CAPS, _______, _______, KC_LBRC, KC_RBRC, _______, _______, _______, _______, _______, _______, _______, \
_______, _______, _______, _______, _______, _______, _______, MO(3),   KC_HOME, KC_PGDN, KC_PGUP, KC_END

// Raise
#define RAISE \
KC_TILD, KC_EXLM, KC_AT,   KC_HASH, KC_DLR,  KC_PERC, KC_CIRC, KC_AMPR, KC_ASTR, KC_LPRN, KC_RPRN, _______, \
KC_GRV,  KC_1,    KC_2,    KC_3,    KC_4,    KC_5,    KC_6,    KC_7,    KC_8,    KC_9,    KC_0,    _______, \
_______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, \
_______, _______, _______, _______, MO(3),   _______, _______, _______, _______, _______, _______, _______

// Adjust
#define ADJUST \
RESET,   KC_F1,   KC_F2,   KC_F3,   KC_F4,   KC_F5,   KC_F6,   KC_F7,   KC_F8,   KC_F9,   KC_F10,  _______, \
_______, KC_F11,  KC_F12,  _______, _______, _______, KC_HOME, KC_PGDN, KC_PGUP, KC_END,  _______, _______, \
_______, _______, _______, _______, _______, _______, _______, KC_INS,  KC_DEL,  _______, _______, _______, \
_______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______
```
Add the line `#include "wrappers.h"` into `config.h`. The `qmk` compile or flash process will combine everything into the firmware:
```
qmk flash ~/qmk_firmware/users/newbie/planck.json
```
The added advantage of using wrapper is ability to share layouts with different keyboards.

# Caveat / Limitations

`config.h` is the only header file included with the keymap built from a json file. Keyboard header `QMK_KEYBOARD_H` cannot be included in `config.h` because it will lead to preprocessor conflict in the build process. Thus custom keycodes that starts at `SAFE_RANGE` cannot be defined as an enumeration data type in `config.h`. Manually defining safe custom keycode range is the only workaround. Manually expanding the json file using `rules.mk` was also proposed in [PR #15480](https://github.com/qmk/qmk_firmware/pull/15480).

# GitHub Integration

## Userspace Repository
With userspace setup as an independent folder, it can be stored in a personal GitHub repository distinct from QMK firmware. The userspace folder `~/qmk_firmware/users/newbie/` can be setup to use a different origin, like `https://github.com/newbie/`. Example:
```
~/qmk_firmware/              : https://github.com/qmk/qmk_firmware
~/qmk_firmware/users/newbie/ : git@github.com:newbie/qmk_userspace.git
```
When setup in this manner, `git pull` inside `~/qmk_firmware/` will update directly from QMK repository, while `~/qmk_firmware/users/newbie/` will be pull and push from GitHub `newbie`.

## Building with GitHub Actions
[GitHub Actions](https://docs.github.com/en/actions) can be used to build QMK firmware, eliminating the need to setup a local build environment. To do so, create the workflow file within the userspace folder `~/qmk_firmware/users/newbie/.github/workflows/build-qmk.yml`, with this example [build-qmk.yml](build-qmk.yml).

The `matrix.keyboard:` list are names that matches the json files (`planck` and `crkbd` in the example). The workflow will clone QMK firmware and userspace repositories into a container on GitHub to build them. The output firmware zip files will be found in the Action tab. Credit goes to [@caksoylar](https://github.com/caksoylar) for sharing this workflow.

# Summary
Maintaining personal build environment this way will keep code files tidy in one location instead of scattering them all over the QMK source tree.

# Links
[QMK cheatsheet](https://jayliu50.github.io/qmk-cheatsheet/)
