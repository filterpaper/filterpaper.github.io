# Saving Bytes with QMK

My CRKBD firmware went above its size limit by 100 bytes at one point, even with `LTO_ENABLE = yes`. Documented below are various ways to code QMK with for better space efficiency if you need to bring the firmware size down by just a bit to fit inside the MCU.

## OLED functions are expensive
This is an easy to write OLED display function:
```c
static void render_logo(void) {
	oled_write_P(PSTR("corne"), false);
}

static void render_layer_state(void) {
	if (layer_state_is(ADJ)) { oled_write_P(PSTR("ADJ"), false); }
}

void render_status(void) {
	render_logo();
	oled_set_cursor(0,3);
	render_layer_state();
}
```
However `oled_set_cursor()` is costly in bytes. If it is used to just skip a few lines, that can replaced by rendering carriage return inside the initial `oled_write_P()`, saving 34 bytes:
```c
static void render_logo(void) {
	oled_write_P(PSTR("corne\n\n"), false);
}

static void render_layer_state(void) {
	if (layer_state_is(ADJ)) { oled_write_P(PSTR("ADJ"), false); }
}

void render_status(void) {
	render_logo();
	render_layer_state();
}
```
## Avoid multiple functions inside "if" condition
The first `if` statement evaluates two conditions that invokes two external function call. Machine codes generated can be space consuming:
```c
if (get_mods() & MOD_MASK_SHIFT || host_keyboard_led_state().caps_lock) { render_luna_bark(); }
else if (get_mods() & MOD_MASK_CAG) { render_luna_sneak(); }
else if (elapsed_time <LUNA_FRAME_DURATION*2) { render_luna_run(); }
else if (elapsed_time <LUNA_FRAME_DURATION*15) { render_luna_walk(); }
else { render_luna_sit(); }
```
Saving the boolean output of `host_keyboard_led_state().caps_lock` into a variable to use in the `if` statement saved 26 bytes:
```c
bool const caps = host_keyboard_led_state().caps_lock;

if (get_mods() & MOD_MASK_SHIFT || caps) { render_luna_bark(); }
else if (get_mods() & MOD_MASK_CAG) { render_luna_sneak(); }
else if (elapsed_time <LUNA_FRAME_DURATION*2) { render_luna_run(); }
else if (elapsed_time <LUNA_FRAME_DURATION*15) { render_luna_walk(); }
else { render_luna_sit(); }
```
## Avoid repeated calls:
This is the vanilla Bongocat animation function, driven by WPM:
```c
uint32_t anim_timer = 0;
uint32_t anim_sleep = 0;

static void animate_cat(void) {
	void animation_phase(void) {
		if (get_current_wpm() > TAP_SPEED) { render_cat_tap(); }
		else if (get_current_wpm() > IDLE_SPEED) { render_cat_prep(); }
		else { render_cat_idle(); }
	}

	if (get_current_wpm() >0) {
		oled_on();
		if (timer_elapsed32(anim_timer) > FRAME_DURATION) {
			anim_timer = timer_read32();
			animation_phase();
		}
		anim_sleep = timer_read32();
	} else {
		if (timer_elapsed32(anim_sleep) > OLED_TIMEOUT) {
			oled_off();
		} else {
			if (timer_elapsed32(anim_timer) > FRAME_DURATION) {
				anim_timer = timer_read32();
				animation_phase();
			}
		}
	}
}
```
The main animation `if`-`else` statements is slightly obfuscated and calls `animation_phase()` twice. Reducing that and eliminating the sleep timer saved 182 bytes in firmware size:
```c
uint32_t anim_timer = 0;

static void animate_cat(void) {
	void animation_phase(void) {
		if (get_current_wpm() > TAP_SPEED) { render_cat_tap(); }
		else if (get_current_wpm() > IDLE_SPEED) { render_cat_prep(); }
		else { render_cat_idle(); }
	}

	if (!get_current_wpm()) {
		oled_off();
	} else if (timer_elapsed32(anim_timer) > FRAME_DURATION) {
		anim_timer = timer_read32();
		animation_phase();
	}
}
```
## Use smaller unsigned integers
In the last example above `anim_timer` is an unsigned 32-bit integer but `FRAME_DURATION` is 200, the maximum value evaluated by that `if` statement. Reducing that variable to unsigned 16-bit saved another 50 bytes:
```c
uint16_t anim_timer = 0;

static void animate_cat(void) {
	void animation_phase(void) {
		if (get_current_wpm() > TAP_SPEED) { render_cat_tap(); }
		else if (get_current_wpm() > IDLE_SPEED) { render_cat_prep(); }
		else { render_cat_idle(); }
	}

	if (!get_current_wpm()) {
		oled_off();
	} else if (timer_elapsed(anim_timer) > FRAME_DURATION) {
		anim_timer = timer_read();
		animation_phase();
	}
}
```
## Decreasing "for" Loop
The following is modifier key lighting code, using a typical `for` loop with incrementing counter:
```c
if (get_mods() & MOD_MASK_CSAG) {
	for (uint_fast8_t i = 0; i <DRIVER_LED_TOTAL; ++i) {
		if (HAS_FLAGS(g_led_config.flags[i], LED_FLAG_MODIFIER)) {
			rgb_matrix_set_color(i, RGB_MODS);
		}
	}
}
```
You can save 10 bytes by decrementing to 0 because machine language will exit zero state with lesser code:
```c
for (uint_fast8_t i = DRIVER_LED_TOTAL; i !=0; --i) {
	if (HAS_FLAGS(g_led_config.flags[i-1], LED_FLAG_MODIFIER)) {
		rgb_matrix_set_color(i-1, RGB_MODS);
	}
}
```
## Use "if" instead of "switch" for non-sequential
The `switch` statement is easy to read for multi-choice condition, but it can also be space consuming when matched cases are not sequential. In the following "capsword" function example, the first switch statement filters for modifier and layer tap keycodes to apply a bitmask:
```c
static void process_caps_word(uint_fast16_t keycode, keyrecord_t const *record) {
	// Get base key code of mod or layer tap with bitmask
	switch (keycode) {
	case QK_MOD_TAP ... QK_MOD_TAP_MAX:
	case QK_LAYER_TAP ... QK_LAYER_TAP_MAX:
		if (record->tap.count) { keycode = keycode & 0xFF; }
	}
	// Toggle caps lock with the following key codes
	switch (keycode) {
	case KC_TAB:
	case KC_ESC:
	case KC_SPC:
	case KC_ENT:
	case KC_DOT:
		if (record->event.pressed) { tap_code(KC_CAPS); }
	}
}
```
The `switch` statement is comparing cases that are not sequential. Replacing that with a multi-conditional statement saved 6 bytes:
```c
static void process_caps_word(uint_fast16_t keycode, keyrecord_t const *record) {
	// Get base key code of mod or layer tap with bitmask
	if (((keycode >=QK_MOD_TAP && keycode <=QK_MOD_TAP_MAX) ||
		(keycode >=QK_LAYER_TAP && keycode <=QK_LAYER_TAP_MAX)) &&
		(record->tap.count)) { keycode = keycode & 0xFF; }
	// Toggle caps lock with the following key codes
	switch (keycode) {
	case KC_TAB:
	case KC_ESC:
	case KC_SPC:
	case KC_ENT:
	case KC_DOT:
		if (record->event.pressed) { tap_code(KC_CAPS); }
	}
}
```
## Use "switch" instead of "if" for sequential matches
On the other hand, `switch` statements will generate smaller code than `if`-`else` for sequential cases. This is a conditional code to display OLED layer state:
```c
if (layer_state_is(ADJ)) { oled_write_P(adjust_layer, false); }
else if (layer_state_is(RSE)) { oled_write_P(raise_layer, false); }
else if (layer_state_is(LWR)) { oled_write_P(lower_layer, false); }
else { oled_write_P(default_layer, false); }
```
Layer state enumerates sequentially—replacing them with `switch` condition will save 22 bytes:
```c
switch (get_highest_layer(state)) {
	case ADJ:
		oled_write_P(adjust_layer, false);
		break;
	case RSE:
		oled_write_P(raise_layer, false);
		break;
	case LWR:
		oled_write_P(lower_layer, false);
		break;
	default:
		oled_write_P(default_layer, false);
}
```
## Replace function inside conditional statements
Function calls inside conditional `if` statements can contribute to code bloat. This is my Bongocat code using timer instead of absolute WPM:
```c
void render_bongocat(void) {
	// WPM triggered typing timer
	static uint_fast8_t prev_wpm = 0;
	static uint_fast32_t tap_timer = 0;

	if (get_current_wpm() >prev_wpm) { tap_timer = timer_read32(); }
	prev_wpm = get_current_wpm();

	static uint_fast16_t anim_timer = 0;

	void animation_phase(void) {
		oled_clear();
		if (timer_elapsed32(tap_timer) <FRAME_DURATION*3) { render_cat_tap(); }
		else if (timer_elapsed32(tap_timer) <FRAME_DURATION*10) { render_cat_prep(); }
		else { render_cat_idle(); }
	}

	if (timer_elapsed32(tap_timer) >OLED_TIMEOUT) {
		oled_off();
	} else if (timer_elapsed(anim_timer) >FRAME_DURATION) {
		anim_timer = timer_read();
		animation_phase();
	}
}
```
External function `timer_elapsed32(tap_timer)` is called multiple times to evaluate elapsed time. Replacing all 3 with an integer `keystroke` reduced firmware size by 80 bytes:
```c
void render_bongocat(void) {
	// WPM triggered typing timer
	static uint_fast8_t prev_wpm = 0;
	static uint_fast32_t tap_timer = 0;

	if (get_current_wpm() >prev_wpm) { tap_timer = timer_read32(); }
	prev_wpm = get_current_wpm();

	static uint_fast16_t anim_timer = 0;
	uint_fast32_t keystroke = timer_elapsed32(tap_timer);

	void animation_phase(void) {
		oled_clear();
		if (keystroke <FRAME_DURATION*3) { render_cat_tap(); }
		else if (keystroke <FRAME_DURATION*10) { render_cat_prep(); }
		else { render_cat_idle(); }
	}

	if (keystroke >OLED_TIMEOUT) {
		oled_off();
	} else if (timer_elapsed(anim_timer) >FRAME_DURATION) {
		anim_timer = timer_read();
		animation_phase();
	}
}
```

"It is said that programmers should write code for other programmers to understand and leave compilers to write for code for machines."
