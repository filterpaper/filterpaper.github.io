# Pseudorandom Number Generators
A simple suggestion on randomising RGB lights in the QMK discord led me down the rabbit hole of pseudorandom number generators. The original goal was replacing C library's `rand()` function with a smaller function to generate random unsigned 8-bit numbers for RGB. However generating good randomness within 8-bit space is not that easy. Compiled here are interesting codes found on the interwebs.

# Evaluating PRNGs
## Visually with bitmap
While pseudorandom number generators are not of cryptography quality, we do want an algorithm produces randomness that do not repeat themselves when are are used for animation and visual lighting. Rendering the PRNG output as bitmap image is a simple way to check for patterns with [this code](https://stackoverflow.com/questions/50090500/create-simple-bitmap-in-c-without-external-libraries), replacing `rngfunction()` with the PRNG function:

```c
void generate_image(uint_fast64_t (*rngfunction)(), char filename[]) {
	static uint_fast32_t const width  = WIDTH;
	static uint_fast32_t const height = HEIGHT;
	static uint_fast16_t const bitcount = 24;

	int width_in_bytes = ((width * bitcount + 31) / 32) * 4;
	uint_fast32_t imagesize = width_in_bytes * height;
	static uint_fast32_t const biSize = 40;
	static uint_fast32_t const bfOffBits = 54;
	uint_fast32_t filesize = 54 + imagesize;
	static uint_fast16_t const biPlanes = 1;

	unsigned char header[54] = { 0 };
	memcpy(header, "BM", 2);
	memcpy(header + 2 , &filesize, 4);
	memcpy(header + 10, &bfOffBits, 4);
	memcpy(header + 14, &biSize, 4);
	memcpy(header + 18, &width, 4);
	memcpy(header + 22, &height, 4);
	memcpy(header + 26, &biPlanes, 2);
	memcpy(header + 28, &bitcount, 2);
	memcpy(header + 34, &imagesize, 4);

	unsigned char* buf = malloc(imagesize);
	for(int row = height - 1; row >= 0; row--) {
		for(int col = 0; col < width; col++) {

			uint_fast8_t blue  = (uint8_t)rngfunction();
			uint_fast8_t green = (uint8_t)rngfunction();
			uint_fast8_t red   = (uint8_t)rngfunction();
#ifdef GRAYSCALE
			uint_fast8_t gray = 0.3*red + 0.59*green + 0.11*blue;
			blue = green = red = gray;
#endif
			buf[row * width_in_bytes + col * 3 + 0] = blue;
			buf[row * width_in_bytes + col * 3 + 1] = green;
			buf[row * width_in_bytes + col * 3 + 2] = red;
		}
	}
	FILE *fout = fopen("24bit.bmp", "wb");
	fwrite(header, 1, 54, fout);
	fwrite((char*)buf, 1, imagesize, fout);
	fclose(fout);
	free(buf);

	return;
}
```

## Empirically with PractRand
For more serious use of PRNG output, the simple [PractRand tool](http://pracrand.sourceforge.net/) tool can be used to evaluate output quality. See this post for more details on [setting up PractRand tests](https://www.pcg-random.org/posts/how-to-test-with-practrand.html).

# 32 and 64-bit PRNGs
32 and 64-bit realm is where one can find many PRNGs—large variables has sufficient space for mathematical operations. They are overkill for embedded systems like QMK that rarely need big random numbers and compiled code sizes will be larger than `rand()`. Nonetheless, listed in this section are interesting ones that passes PractRand tests.
## PCG32
Melissa O'Neill published her paper and PCG (permuted congruential generator) family of codes at [www.pcg-random.org](https://www.pcg-random.org/). Her most robust [PCG32 code](https://www.pcg-random.org/download.html) has many versions—the following is a seeded version of 64-bit MCG with XORshift and random-rotation:
```c
// pcg_mcg_64_xsh_rr_32_random_r
static uint_fast32_t pcg32(void) {
	static uint_fast64_t state = 0x406832dd910219e5;

	uint_fast64_t oldstate = state;
	state = state * 6364136223846793005ULL;

	uint_fast32_t value = ((oldstate >> 18U) ^ oldstate) >> 27U;
	uint_fast32_t rot = oldstate >> 59U;
	return (value >> rot) | (value << ((- rot) & 31));
}
```
## Xoroshiro++
Shift-register generator using [XORshift](https://en.wikipedia.org/wiki/Xorshift) was discovered by mathematician George Marsaglia. Weaknesses with earlier implementations were improved with XORshift and rotate versions–[dubbed xoshiro / xoroshiro](https://prng.di.unimi.it/). The following is the general purpose `xoroshiro128++`:
```c
static uint_fast64_t rol64(uint_fast64_t const x, int const k) {
	return (x << k) | (x >> (64 - k));
}

// xoroshiro128++
static uint_fast64_t xoroshiro128pp(void) {
	// Seed these 64 bit manually
	static uint_fast64_t s0 = 0xaafdbd4fce743b4d;
	static uint_fast64_t s1 = 0xcaee5c952c4ae6a8;

	uint_fast64_t const t0 = s0;
	uint_fast64_t t1 = s1;
	uint_fast64_t const result = rol64(t0 + t1, 17) + t0;

	t1 ^= t0;
	s0 = rol64(t0, 49) ^ t1 ^ (t1 << 21); // a, b
	s1 = rol64(t1, 28); // c

	return result;
}
```
# 16-bit PRNGs
16-bit is exponentially smaller and more manageable for embedded firmware–its output can be casted as `unint8_t` as needed.
## PCG16
The 16-bit version of PCG code is robust for mo† of PractRand's test and is x larger than `rand()`:
```c
// pcg_mcg_32_xsh_rr_16_random_r
static uint16_t pcg16(void) {
	static uint32_t state = 0x406832dd;

	uint32_t oldstate = state;
	state = state * 747796405U + 1U;

	uint16_t value = ((oldstate >> 10U) ^ oldstate) >> 12U;
	uint32_t rot = oldstate >> 28U;
	return (value >> rot) | (value << ((- rot) & 15));
}
```
## xorshift16
[Brad Forschinger](http://b2d-f9r.blogspot.com/2010/08/16-bit-xorshift-rng-now-with-more.html) shrank Marsaglia's XORshift to the following 2-register code function. Smaller than `rand()`.
```c
static uint_fast16_t rnd_xorshift_16(void) {
	static uint_fast16_t x = 1, y = 1;
	uint_fast16_t t = (x ^ (x << 5U));
	x = y * 3;
	return y = (y ^ (y >> 1U)) ^ (t ^ (t >> 3U));
}
```
# 8-bit PRNGs
8-bit space is where limitation of its size become apparently. Poorly implemented linear-feedback shift register (LFSR) codes will show render repeated patterns on bitmap. Almost all of them will fail most PractRand tests.
## Tzarc's XORshift
@tzarc's [version of XORshift](https://github.com/tzarc/qmk_build/blob/bebe5e5b21e99bdb8ff41500ade1eac2d8417d8c/users-tzarc/tzarc_common.c#L57-L63) was the start of this rabbit hole and his code follows:
```c
static uint8_t prng(void) {
	static uint8_t s = 0xAA, a = 0;
	s ^= s << 3;
	s ^= s >> 5;
	s ^= a++ >> 2;
	return s;
}
```
It is a modified XORshift using two 8-bit variables and its small code base is 220 bytes smaller than `rand()`. The output fails all PractRand's test and there are visual patterns to its bitmap output:

![tzarc_prng](images/tzarc_prng.bmp)

## PCG8
An 8-bit implementation of PCG can be found in the [pcg-c-basic library](https://github.com/imneme/pcg-c-basic). Adapted to a single seeded function below is the 16-bit MCG with 8-bit output using xorshift and random-rotation:
```c
// pcg_mcg_16_xsh_rr_8_random_r
uint8_t pcg8(void) {
	static uint16_t state = 0x6835;

	uint16_t oldstate = state;
	state = state * 12829U;

	uint8_t value = ((oldstate >> 5U) ^ oldstate) >> 5U;
	uint32_t rot = oldstate >> 13U;
	return (value >> rot) | (value << ((- rot) & 7));
}
```
Unlike its larger cousins, the 8-bit version doesn't work as well—strong visual pattern suggest that the output repeat itself with small samples.
![pcg8](images/pcg8.bmp)

## JSF8
Finally there's [Bob Jenkin's PRNG](http://burtleburtle.net/bob/rand/smallprng.html), with a detailed [description by the author](http://burtleburtle.net/bob/rand/talksmall.html). His code is dubbed Jenkins Fast Small or JSF, and [was adapted](https://www.pcg-random.org/posts/bob-jenkins-small-prng-passes-practrand.html) into 16 and this 8-bit version:
```c
uint8_t jsf8(void) {
	static uint8_t a = 0xf1;
	static uint8_t b = 0xee, c = 0xee, d = 0xee;

	uint8_t e = a - ((b << 1)|(b >> 7));
	a = b ^ ((c << 4)|(c >> 4));
	b = c + d;
	c = d + e;
	d = e + a;
	return d;
}
```
And behold, this tiny 8-bit algorithm passes the visual bitmap test, PractRand and is smaller than `rand()`, making it the perfect PRNG for embedded code!
![jsf8](images/jsf8.bmp)

rand() 10582 bytes free
tzard() 10802 bytes free
