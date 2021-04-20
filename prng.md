# Pseudorandom Number Generators
A simple suggestion on randomising RGB lights in the QMK discord led me down the rabbit hole of pseudorandom number generators. The original goal was replacing C library's `rand()` function with a smaller function to generate random unsigned 8-bit numbers for RGB. However generating good randomness within 8-bit space is not that easy.

# Evaluating PRNGs
While pseudorandom number generators are not of cryptography quality, we do want an algorithm produces randomness that do not repeat themselves when are are used for animation and visual lighting. Rendering the PRNG output as bitmap image is a simple way to check for patterns with [this code](https://stackoverflow.com/questions/50090500/create-simple-bitmap-in-c-without-external-libraries), replacing `rngfunction()` with the PRNG function:

{::options parse_block_html="true" /}
<details>
<summary>Show code</summary>

```c
void generate_image(uint_fast64_t (*rngfunction)(), char filename[]) {
	//width, height, and bitcount are the key factors:
	static uint_fast32_t const width  = WIDTH;
	static uint_fast32_t const height = HEIGHT;
	static uint_fast16_t const bitcount = 24;//<- 24-bit bitmap

	//take padding in to account
	int width_in_bytes = ((width * bitcount + 31) / 32) * 4;

	//total image size in bytes, not including header
	uint_fast32_t imagesize = width_in_bytes * height;

	//this value is always 40, it's the sizeof(BITMAPINFOHEADER)
	static uint_fast32_t const biSize = 40;

	//bitmap bits start after headerfile,
	//this is sizeof(BITMAPFILEHEADER) + sizeof(BITMAPINFOHEADER)
	static uint_fast32_t const bfOffBits = 54;

	//total file size:
	uint_fast32_t filesize = 54 + imagesize;

	//number of planes is usually 1
	static uint_fast16_t const biPlanes = 1;

	//create header:
	//copy to buffer instead of BITMAPFILEHEADER and BITMAPINFOHEADER
	//to avoid problems with structure packing
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

	//prepare pixel data:
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

</details>
{::options parse_block_html="false" /}

# 8-bit PRNG
## Tzarc's XORshift
Shift-register generator using [XORshift](https://en.wikipedia.org/wiki/Xorshift) was discovered by mathematician George Marsaglia. @tzarc proposed [his version](https://github.com/tzarc/qmk_build/blob/bebe5e5b21e99bdb8ff41500ade1eac2d8417d8c/users-tzarc/tzarc_common.c#L57-L63) that follows:
```c
static uint8_t prng(void) {
	static uint8_t s = 0xAA, a = 0;
	s ^= s << 3;
	s ^= s >> 5;
	s ^= a++ >> 2;
	return s;
}
```
His code is a modified XORshift using shifted two 8-bit variables. Its small code base is 220 bytes smaller than `rand()`. Visualising its output showed repeated patterns:

![tzarc_prng](images/tzarc_prng.bmp)

Repetition is not severe, but the output will immediately fail tests such as [PractRand](http://pracrand.sourceforge.net/).j

## PCG 8-bit
Melissa O'Neill published her paper and PCG family of codes at [www.pcg-random.org](https://www.pcg-random.org/). An 8-bit implementation is among the [pcg-c-basic librar](https://github.com/imneme/pcg-c-basic). Adapted to a single seeded function below is the 16-bit MCG with 8-bit output using xorshift and random-rotation:
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


rand() 10582 bytes free
tzard() 10802 bytes free
