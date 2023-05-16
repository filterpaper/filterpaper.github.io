# Standalone Userspace

Building QMK firmware with a JSON file and [userspace](https://docs.qmk.fm/#/feature_userspace) is a less common but effective method that has several advantages, including:

* Simplified file management: All files are kept in a single folder, which can help to keep your project organized and easy to navigate.
* No need to convert `keymap.json`: The JSON file can be used directly, which eliminates the need to use the qmk json2c tool.
* Easier editing of keymaps: The JSON file is much easier to edit than the keymap.c file, which makes it convenient to make changes to your keymap.
* Easy to add support for more keyboards with additional JSON files, which makes it a versatile and flexible option for building QMK firmware.


# Initial space setup
Before you begin, make sure that you have installed the QMK build environment correctly. You can find instructions on how to do this on the [QMK getting started](https://docs.qmk.fm/#/newbs_getting_started) page.

Once you have installed the QMK build environment, you need to create a userspace for your GitHub username. To do this, create a new directory inside the qmk_firmware/users/ directory and name it after your GitHub username.

For example, if your GitHub username is `newbie`, you would create a directory called `newbie` inside the `qmk_firmware/users/` directory:
```
mkdir ~/qmk_firmware/users/newbie/
```
To customize a keymap layout for your keyboard, you can use the [QMK Configurator](https://config.qmk.fm/). To do this, follow these steps:

1. Go to the QMK Configurator website.
1. Select your keyboard from the list of supported keyboards.
1. In the `KEYMAP NAME` field, enter the name of your userspace. In this example, the name is "newbie".
1. Click on the "Customize" button.
1. Use the Configurator to customize your keymap layout.
1. Click on the "Export" button.
1. The keymap layout will be saved as a JSON file called "`newbie.json`" by default.
1. Move the "`newbie.json`" file into the userspace folder.


# Compiling a default firmware
To build a firmware with your custom keymap layout, you can use the `qmk compile` command with your keymap JSON file as the parameter:
```
qmk compile ~/qmk_firmware/users/newbie/newbie.json
```
If everything goes smoothly, the command will build a firmware with your custom keymap layout. You can also run the `qmk flash` command to compile and flash the firmware to your keyboard in one step.


# Customising the firmware
There are many customisation options with QMK and you can start with the [custom quantum function page](https://docs.qmk.fm/#/custom_quantum_functions). The file naming conventions of [QMK userspace](https://docs.qmk.fm/#/feature_userspace) is the guide for organising this folder.

Hardware feature and QMK variables are placed in `rules.mk` and `config.h`. Both files should be created in the same folder:
```
~/qmk_firmware/users/newbie/rules.mk
~/qmk_firmware/users/newbie/config.h
```
Instead of `keymap.c`, custom source codes should be added into `<name>.c`, for example:
```
~/qmk_firmware/users/newbie/newbie.c
```
Unlike `keymap.c`, the `keymaps[]` array is excluded from this file because the layout is stored in the JSON file. To tell the QMK introspection build process to find your custom source codes during compile time, you need to add the following line to the rules.mk file:
```c
INTROSPECTION_KEYMAP_C = newbie.c
```
When you are done, the userspace folder will contain these 4 basic files:
```
~$ tree qmk_firmware/users/newbie/
qmk_firmware/users/newbie/
|-- config.h
|-- newbie.c
|-- newbie.json
`-- rules.mk

0 directories, 4 files
```
When you run the `qmk compile ~/qmk_firmware/users/newbie/newbie.json` command, the QMK build process will automatically include all files in the current directory. However, not all of these files are required. The only required file is the JSON layout file while the others are optional. 


# Supporting multiple keyboard
Additional keyboards can be configured in the same userspace with the following guideline:
* Use distinct keyboard file names for each JSON like `<keyboard-name>.json`.
* Use `#ifdef` conditional preprocessors for keyboard-specific codes.

The following are some examples:

## rules.mk
QMK features can be enabled only for specific hardware within `ifeq` blocks:
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

INTROSPECTION_KEYMAP_C = newbie.c
```

## config.h
QMK variables can be configured selectively inside conditional `#ifdef` blocks:
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
It is recommended to use `#ifdef` conditions in the source code to exclude code that is only meant for specific keyboards. This can be done by matching the `#ifdef` conditions to the keyboard definitions in `rules.mk` and `config.h`. Here is an example:
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
When you are finished, the shared userspace for multiple keyboards will be organized in a clean and efficient manner:
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
Compiling and flashing the firmware for each keyboard can be perform using `qmk flash` with the JSON file as parameter:
```
qmk flash ~/qmk_firmware/users/newbie/bm40rgb.json
qmk flash ~/qmk_firmware/users/newbie/crkbd.json
```

# Advance wrapper layout
The keymap JSON file that is exported from the Configurator is not always ideal for power users who prefer to edit layouts in text format. A keymap "wrapper" can be adapted to use JSON files, which can be more convenient for power users.
## planck.json example
Instead of exporting a keymap from the Configurator, you can create a `planck.json` file and define the keymap layout and macro names manually:
```json
{
    "author": "",
    "documentation": "Wrapper based keymap",
    "keyboard": "planck/rev6",
    "keymap": "newbie",
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
#include "quantum/keycodes.h"

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
Add the following line into `config.h` to include the wrapper macros into the build process: 
```
#ifndef __ASSEMBLER__ // Guard against use with non-C files
#    include "wrappers.h"
#endif
```
The QMK build process will automatically expand the macros defined in the `planck.json` file when compiling the firmware.

# GitHub Integration

## Userspace Repository
With userspace set up as a standalone folder, it can be stored in a personal GitHub repository that is *separate* from QMK firmware. The userspace folder `~/qmk_firmware/users/newbie/` can point to a different `git` origin. For example:
```
~/qmk_firmware/              : https://github.com/qmk/qmk_firmware
~/qmk_firmware/users/newbie/ : git@github.com:newbie/qmk_userspace.git
```
When configured in this manner, `git pull` inside `~/qmk_firmware/` will pull directly from QMK repository, while `~/qmk_firmware/users/newbie/` will use `newbie`'s GitHub repository.

## Building with GitHub Actions
[GitHub Actions](https://docs.github.com/en/actions) can be used to build QMK firmware without the need to set up a local build environment. To do this, create a workflow file in the userspace folder `~/qmk_firmware/users/newbie/.github/workflows/build-qmk.yml` with the following content:
{% raw %}
```yml
name: Build QMK firmware
on: [push, workflow_dispatch]

jobs:
  build:
    runs-on: ubuntu-latest
    container: qmkfm/qmk_cli
    strategy:
      fail-fast: false
      matrix:
# List of keymap json files to build
        file:
        - planck.json
        - crkbd.json
# End of json file list

    steps:
    - name: Disable git safe directory checks
      run : git config --global --add safe.directory '*'

    - name: Checkout QMK
      uses: actions/checkout@v3
      with:
        repository: qmk/qmk_firmware
# Uncomment the following for develop branch
#        ref: develop
        submodules: true

    - name: Checkout userspace
      uses: actions/checkout@v2
      with:
        path: users/${{ github.actor }}

    - name: Build firmware
      run: qmk compile "users/${{ github.actor }}/${{ matrix.file }}"

    - name: Archive firmware
      uses: actions/upload-artifact@v2
      with:
        name: ${{ matrix.file }}_${{ github.actor }}
        retention-days: 5
        path: |
          *.hex
          *.bin
          *.uf2
      continue-on-error: true
```
{% endraw %}
The `matrix.file:` node is the list of JSON files to be built (`planck.json` and `crkbd.json` are in the example). The workflow will clone a copy of QMK firmware and the userspace repository into a container on GitHub to build them. The output firmware zip files will be found in the Action tab. See [Building QMK with GitHub Userspace](https://docs.qmk.fm/#/newbs_building_firmware_workflow) for more details. Credit goes to [@caksoylar](https://github.com/caksoylar) for sharing this workflow.



# Links
* [QMK cheatsheet](https://jayliu50.github.io/qmk-cheatsheet/)
* [GitHub Actions](https://docs.github.com/en/actions)
