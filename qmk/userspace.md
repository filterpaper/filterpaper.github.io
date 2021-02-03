# Introduction

One of the least obvious way of building QMK firmware is using the json file exported out of the [configurator](https://config.qmk.fm/). This method is favored personally for the following advantage:

* Ease of file maintenance with everything inside userspace—no need to go deep inside QMK source tree like `keyboards/kbdfans/kbd67/mkiirgb/keymaps`.
* Eliminates additional step `keymap.c` conversion using `qmk json2c` if configurator is a preferred key map editor. 
* Removes orenous `keymap.c` maintenance with text editors.
* Easy to extend support for more than one keyboard inside the same space.


# First time setup 
(Prerequisite: [QMK build environment](https://docs.qmk.fm/#/newbs_getting_started) must be setup correctly before proceeding.
The [QMK userspace](https://docs.qmk.fm/#/feature_userspace) should be your github name—"newbie" in this example, created inside `qmk_firmware/users/`:
```
mkdir ~/qmk_firmware/users/newbie/
```
Visit the [configurator](https://config.qmk.fm/) to customise a key map layout for your keyboard. It's important that the "key map name" should match your userpace, `newbie` in this example. 

Export the key layout and it will be saved as newbie.json. Move that into your QMK userspace:
```
mv newbie.json ~/qmk_firmware/users/newbie/
```


# Compiling first default firmware
That space is now setup to compile the first firmware with default setting. Simply run `qmk compile` with that .json file. Important: the command expects the full path, even if executed in the current folder:
```
qmk compile ~/qmk_firmware/users/newbie/newbie.json
```
If everything goes well the will be compiled with default settings from the keyboard source, and it is as simple as that.


# Customising the firmware
You can start customising the firmware with code files saved inside the userspace. Do see QMK guide on [customising keyboard behavior](https://docs.qmk.fm/#/custom_quantum_functions) and follow the file naming conventions of [QMK userspace](https://docs.qmk.fm/#/feature_userspace).

Hardware feature selection and QMK variables should be saved in the following files:
```
~/qmk_firmware/users/newbie/rules.mk
~/qmk_firmware/users/newbie/config.h
```
Instead `keymap.c`, programming codes can be added into your own `<name>.c` like:
```
~/qmk_firmware/users/newbie/newbie.c
```
QMK compiler must be informed of this file with the following line at the end of `rules.mk`:
```c
SRC += newbie.c
```

Running `qmk compile ~/qmk_firmware/users/newbie/newbie.json` will now pick up the custom codes in the userspace.


# Supporting more than one keyboard
Additional keyboards can be configured and supported in the same userspace with the following guideline:
* Provide distinct names for each keyboard's .json like `<keyboard-name>.json`.
* Liberal use of `#ifdef` to block out code sections for specific keyboard or feature set.

## `rules.mk`
QMK features can be enabled and disabled for distinct hardware define blocks:
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
```

## `config.h`
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
#	define RGB_MATRIX_KEYPRESSES
#	define RGB_DISABLE_WHEN_USB_SUSPENDED true
#	define RGB_MATRIX_MAXIMUM_BRIGHTNESS 100
#endif

#ifdef OLED_DRIVER_ENABLE
#	define OLED_TIMEOUT 5000
#endif
```

## `<name>.c`
There should also be judicious use of `#ifdef` blocks in the source to include and exclude codes to match those in `rules.mk` and `config.h`. Example:
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
Maintaining the userspace in this will keep your code files tidy in one location instead of scattering them all over the QMK source tree. Backup and git commits will be centralized to one location. However it is not without some drawbacks:
* Reliance on QMK Configurator to modify key map layout
* .json files are illegible to inspect
