# Introduction

One of the least obvious way of building QMK firmware is using the json file exported out of the [configurator](https://config.qmk.fm/). This method is favored personally for the following advantage:

* Ease of file maintenance with everything inside userspace—no need to go deep inside QMK source tree like `keyboards/kbdfans/kbd67/mkiirgb/keymaps`.
* Eliminates additional step `keymap.c` conversion using `qmk json2c` if configurator is a preferred key map editor. 
* Removes orenous `keymap.c` maintenance with text editors.
* Easy to extend support for more than one keyboard inside the same space.

# First time setup 
The [QMK userspace](https://docs.qmk.fm/#/feature_userspace) name should be the github name—"newbie" in this example, created inside `qmk_firmware/users/`:
```
mkdir ~/qmk_firmware/users/newbie/
```
Visit the [configurator](https://config.qmk.fm/) to customise a key map layout for your keyboard. It's important that the "key map name" should match your userpace, `newbie` in this example. 

Export the key layout and it will be saved as newbie.json. Move that into user space:
```
mv newbie.json ~/qmk_firmware/users/newbie/
```

# Compiling first default firmware
The userspace is now setup to compile the first firmware with default setting. Simply run `qmk compile` with that .json file. Important: the command expects the full path, even if it is saved in the current folder:
```
qmk compile ~/qmk_firmware/users/newbie/newbie.json
```
If everything goes well the will be compiled with default settings from the keyboard source code. 

# Customising the firmware
Do see QMK guide on [customising keyboard behavior](https://docs.qmk.fm/#/custom_quantum_functions) and follow the file naming conventions of [QMK userspace](https://docs.qmk.fm/#/feature_userspace). Hardware feature selection and QMK variables should be saved in the following files:
```
~/qmk_firmware/users/newbie/rules.mk
~/qmk_firmware/users/newbie/config.h
```
Instead using `keymap.c`, programming codes can be added into your own `<name>.c` files of your choosing like:
```
~/qmk_firmware/users/newbie/newbie.c
```
QMK compiler must be informed of this file with the following line at the end of `rules.mk`:
```c
SRC += newbie.c
```
The `qmk compile ~/qmk_firmware/users/newbie/newbie.json` process will now process those custom codes found inside the userspace directory.

# Supporting more than one keyboard
More than one keyboard can be configured and supported in the same userspace with the following ways:
* Provide distinct names for each keyboard's layout .json like `<keyboard-name>.json`.
* Liberal use of `#ifdef` to block out code sections for specific keyboard or feature set.

