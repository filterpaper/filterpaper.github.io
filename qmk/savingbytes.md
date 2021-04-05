# Saving Bytes with QMK

It is said that programmers should write code for other programmers to understand and leave compilers to write for code for machines.

However easy to understand codes may not be the most efficient with execution and space. The latter is a premium for 28672 bytes controllers like the Elite-C MCU, especially when you try to squeeze in RGB features and OLED animation.

If you are over the firmware limit and looking to shave a few bytes off the QMK compiled output, there are a couple of C code tricks that can employed if `LTO_ENABLE = yes` didn't help.

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
However `oled_set_cursor()` is costly in byte usage. If it is used to just skip a few lines, that can replaced by rendering carriage return inside the initial `oled_write_P()`, saving 34 bytes:
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
## Avoid repeated function with variables

## Multiple calls in if statements

## Decreasing for Loop
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
You can save 10 bytes by decrementing to 0 because machine language handles zero state in lesser code:
```c
for (uint_fast8_t i = DRIVER_LED_TOTAL; i !=0; --i) {
	if (HAS_FLAGS(g_led_config.flags[i-1], LED_FLAG_MODIFIER)) {
		rgb_matrix_set_color(i-1, RGB_MODS);
	}
}
```


## use IF istead of switch for non sequential

## use switch if sequential
