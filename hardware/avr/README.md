# AVR

> We found this old terminal with access to some top secret data, but it's secured by passwords. Can you break in anyway?

This challenge is about abusing certain conditions in the AVR hardware timer to sniff the terminal login password through a timing
attack and further to abuse a race condition that lets us query the flag.

## Introduction

Upon downloading the challenge from the server, you obtain a mysterious file called
`8bfc40e205d0793678e76f2610fc0a9f58159fcdcbbf3424b0538b0b019bfd50c0ddffcaeca391379f260390c90b1e4d5633acb2c334bd5f5663c4072354bb13`.

With the help of the `file` command, it can be identified as a zip archive:

```sh
❯ file 8bfc40e205d0793678e76f2610fc0a9f58159fcdcbbf3424b0538b0b019bfd50c0ddffcaeca391379f260390c90b1e4d5633acb2c334bd5f5663c4072354bb13
8bfc40e205d0793678e76f2610fc0a9f58159fcdcbbf3424b0538b0b019bfd50c0ddffcaeca391379f260390c90b1e4d5633acb2c334bd5f5663c4072354bb13: Zip archive data, at least v2.0 to extract

❯ unzip 8bfc40e205d0793678e76f2610fc0a9f58159fcdcbbf3424b0538b0b019bfd50c0ddffcaeca391379f260390c90b1e4d5633acb2c334bd5f5663c4072354bb13
Archive:  8bfc40e205d0793678e76f2610fc0a9f58159fcdcbbf3424b0538b0b019bfd50c0ddffcaeca391379f260390c90b1e4d5633acb2c334bd5f5663c4072354bb13
 extracting: Makefile
 extracting: code.c
 extracting: code.hex
 extracting: simavr_diff
 extracting: simduino.elf
```

This reveals a small project setup, so let's look at the individual files:

* [`code.c`](./code.c): Obviously the source code of the application

* [`code.hex`](./code.hex): The "compiled" binary that will be flashed onto the Arduino, or, in our case, emulated in simduino (see below)

* [`Makefile`](./Makefile): A build setup for this AVR project

* [`simavr_diff`](./simavr_diff): As the Makefile suggests, it's a path to the [simavr](https://github.com/buserror/simavr) project
that will be applied to it before building it

* [`simduino.elf`](./simduino.elf): Seemingly the [simavr](https://github.com/buserror/simavr) for Arduino hardware emulation

So apparently we have some AVR binary that is being run inside an emulator.

## Analyzing the code

After some googling, I learned that `hex` files, which contain lines that look like a nicely lined up hexdump, belong to the
[Intel HEX](https://en.wikipedia.org/wiki/Intel_HEX) file format.

We can use the following command to disassemble the hex lines:

```sh
avr-objdump -j .sec1 -d -m avr5 code.hex > code.dis
```

So as I started to read the C code, I'd been following along the disassembly to get a picture of the inner workings of some AVR APIs.
Keep in mind that we're doing a hardware CTF, without any prior knowledge to the challenge, I wasn't expecting something that is obvious on
the eye without any insight in how the AVR architecture and the provided abstractions from the `avr` library work.

For the same reason, I also followed along the [`atmega328p` datasheet](https://ww1.microchip.com/downloads/en/DeviceDoc/Atmel-7810-Automotive-Microcontrollers-ATmega328P_Datasheet.pdf) since we know from the `avr-gcc` invocation in the Makefile that we're dealing
with exactly this MCU.

Equipped with all the knowledge from the holy datasheet and the disassembly, we can tackle the contents of `code.c`.

```c
#undef F_CPU
#define F_CPU 1000000UL
```

I've noticed this as I've been flying over the `simavr_diff` patch already, apparently Google purposefully underclocks the MCU from its original
frequency of `16000000` to `1000000`, which means the processor will run 16 (!) times slower than usual. My guess here was that the `F_CPU` macro
is utilized by the AVR APIs, hence the re-declaration before the includes.

```c
#ifndef PASS1
#define PASS1 "PASSWORD_REDACTED_XYZ"
#endif

#ifndef PASS2
#define PASS2 "TOPSECRET_PASSWORD_ALSO_REDACTED_TOPSECRET"
#endif

#ifndef FLAG
#define FLAG "CTF{_REAL_FLAG_IS_ON_THE_SERVER_}"
#endif

const char* correctpass = PASS1;
const char* top_secret_password = PASS2;
const char* top_secret_data = 
	"INTELLIGENCE REPORT:\n"
	"FLAG CAPTURED FROM ENEMY.\n"
	"FLAG IS " FLAG ".";

char buf[512];
char secret[256] = 
	"Operation SIERRA TANGO ROMEO:\n"
	"Radio frequency: 13.37MHz\n"
	"Received message: ATTACK AT DAWN\n";
char timer_status[16] = "off";
```

Breakdown of what we have here:

* Three macros, seemingly two for passwords and the third one is our desired flag

* `PASS*` macros are bound to variables `correctpass` and `top_secret_password`

* `top_secret_data` is a string buffer that contains the flag

* An empty buffer of 512 bytes is allocated in `buf`

* Another string buffer allocated in `secret`, which contains seemingly uninteresting data

* `timer_status` is another string buffer holding a value of `"off"` which we can't interpret yet

```c
volatile char uart_ready;
ISR(USART_RX_vect) {
	uart_ready = 1;
}
```

My good friend Google has brought me to the conclusion that `ISR(USART_RX_vect)` defines a new interrupt
handler for us that sets `uart_ready` to `1` whenever we finished receiving data over the integrated
[UART](https://de.wikipedia.org/wiki/Universal_Asynchronous_Receiver_Transmitter) bus.

```c
void uart_init(void) {
    UBRR0H = UBRRH_VALUE;
    UBRR0L = UBRRL_VALUE;

    UCSR0C = (1<<UCSZ01) | (1<<UCSZ00);
    UCSR0B = (1<<RXEN0) | (1<<TXEN0) | (1<<RXCIE0);
}
```

This piece of code initializes, as the function name suggests, the
[UART](https://de.wikipedia.org/wiki/Universal_Asynchronous_Receiver_Transmitter) bus on hardware with a baud rate of `125000`.

It initializes both TX and RX, but RX with full interrupts. Further, it configures transfers for 8 data bits and even parity,
although that's irrelevant for our case as we're not going to interface with physical hardware directly.

```c
static int uart_getchar(FILE* stream) {
	while (1) {
		cli();
		if (!uart_ready) {
			sleep_enable();
			sei();
			sleep_cpu();
			sleep_disable();
		}
		cli();
		if (uart_ready) {
			uart_ready = 0;
			unsigned int c = UDR0;
			sei();
			return c;
		}
		sei();
	}
}

static int uart_putchar(char c, FILE* stream) {
	loop_until_bit_is_set(UCSR0A, UDRE0);
	UDR0 = c;
	return 0;
}
static FILE uart = FDEV_SETUP_STREAM(uart_putchar, uart_getchar, _FDEV_SETUP_RW);
```

Here, we have implementations of two functions inspired by `getchar` and `putchar`, except that they're
meant to work over [UART](https://de.wikipedia.org/wiki/Universal_Asynchronous_Receiver_Transmitter) MMIO access.

And then below, a virtual file device called `uart` is being defined with the previously implemented functions. Later
this is used to redirect all I/O to the UART driver interface that was just implemented.

```c
void quit() {
	printf("Quitting...\n");
	_delay_ms(100);
	cli();
	sleep_enable();
	sleep_cpu();
	while (1);
}
```

A `quit` function which halts the CPU and essentially causes a shutdown of the hardware over MMIO.

```c
volatile uint32_t overflow_count;
uint32_t get_time() {
	uint32_t t;
	cli();
	t = (overflow_count << 16) + TCNT1;
	sei();
	return t;
}

void timer_on_off(char enable) {
	overflow_count = 0;
	strcpy(timer_status, enable ? "on" : "off");
	if (enable) {
		TCCR1B = (1<<CS10);
		sei();
	}
	else {
		TCCR1B = 0;
	}
}

ISR(TIMER1_OVF_vect) {
	if (!logged_in) {
		overflow_count++;
		// Allow ten seconds.
		if (overflow_count >= ((10*F_CPU)>>16)) {
			printf("Timed out logging in.\n");
			quit();
		}
	}
	else {
		// If logged in, timer is used to securely copy top secret data.
		secret[top_secret_index] = top_secret_data[top_secret_index];
		timer_on_off(top_secret_data[top_secret_index]);
		top_secret_index++;
	}
}
```

The next bit of code is responsible for configuring a hardware timer of the MCU, namely Counter/Timer 1.

To understand all the code and its purpose, we need to take a look at how AVR timers work. Luckily, there's a really
good article at https://exploreembedded.com/wiki/AVR_Timer_programming.

We learn that Counter 1 is a 16-bit timer that generates an overflow upon hitting `65536` and starts again from `0`.
This explains the `overflow_count` variable! It essentially tracks how many overflow the timer has had so far and with
this information, we can derive for how long the timer's been running already.

Let's do some quick mafs with the CPU frequency and the prescale from the `timer_on_off` code:

![Rendered by quicklatex.com](./counter_calc.png)

Now we know that every overflow in the timer means that ~65.5 milliseconds have ellapsed since the start of it!

It becomes obvious now that `get_time` also uses `overflow_count` to derive the time that has ellapsed so far. Next,
there is `timer_on_off` which uses the MMIO to turn the timer on off (who would've guessed it), based on the value of `enable`
and also `strcpy`s the current status into the `timer_status` variable we discovered earlier.

And finally, `ISR(TIMER1_OVF_vect)` declares another interrupt handler for, timer overflow events. But what it actually does is pretty
interesting. If the variable `logged_in` is not set, it will increment `overflow_count` by one and compare its value against `152`,
which evaluates to ~9.6 seconds rounded up to 10 seconds (as the comment also suggests, but we don't trust anything we haven't verified
ourselves here, of course). If `logged_in` is set on the other hand though, one byte from string buffer with our flag will be copied into
useless string buffer with the funky message. The timer will be either turned on or off, depending on the char that has been copied and
`top_secret_index` is incremented by one.

We'll get back to this later.

```c
void read_data(char* buf) {
	scanf("%200s", buf);
}

void print_timer_status() {
	printf("Timer: %s.\n", timer_status);
}
```

These two functions are intended as helpers. `read_data` will read at most 200 characters into the `buf` argument. `print_timer_status`
just prints out the contents of the `timer_status` variable, which is either `"on"` or `"off"`, depending on how the timer is configured.

For the `main` function that is actually pretty big, we break it down into smaller chunks.

```c
	uart_init();
	stdout = &uart;
	stdin = &uart;
```

First, we initialize the UART bus on hardware and overwrite `stdout` and `stdin` to our previously crafted UART device so that all the I/O
functions from `stdout.h` header will drive the UART.

```c
	TCCR1A = 0;
	TIMSK1 = (1<<TOIE1);
```

As explained by the datasheet, these MMIO writes are used to initialize the timer. Setting the `TOIE1` bit in the `TIMSK1` register will enable
Counter 1 overflow interrupts that we previously set up an interrupt handler for.

```c
	printf("Initialized.\n");
	printf("Welcome to secret military database. Press ENTER to continue.\n");
	char enter = uart_getchar(0);
	if (enter != '\n') {
		quit();
	}
```

We're greeted and prompted to enter a `'\n'` character through ENTER, otherwise the program will quit.

```c
	timer_on_off(1);
```

The timer is being brought up!

```c
	while (1) {
		print_timer_status();
		printf("Uptime: %ldus\n", get_time());
		printf("Login: ");
		read_data(buf);
		printf("Password: ");
		read_data(buf+256);
		if (strcmp(buf, "agent") == 0 && strcmp(buf+256, correctpass) == 0) {
			printf("Access granted.\n");
			break;
		}
		printf("Wrong user/password.\n");
	}
```

The first rock in our way. We get the status of the timer and the uptime thrown into our way
and are prompted for login data then. The `buf` of 512 bytes is being used to store our input,
whereas the first 256 bytes are meant for our username and the last 256 bytes for our password.
If we remember back, `read_data` uses a `"%200s"` format string, so sadly no memory corruption
possible here.

Then our data is being validated using `strcmp` calls. For the username, we can se a hardcoded value
of `"agent"`, so that's an easy one. For the password, we need to look out for `correctpass` which is
defined to...well...`"PASSWORD_REDACTED_XYZ"`. So we have, in fact, no idea what password the program
expects here. Anyway, if we pass this thing, we'll be granted access and the program breaks out of the
infinite loop, whereas it just starts from the beginning of the block if we fail to pass the check.

We'll get back to this later as well, I promise!

```c
	cli();
	timer_on_off(0);
	sei();

	logged_in = 1;
```

So now that we are logged in with correct credentials, the `logged_in` variable will be set accordingly and
the timer will be disabled. But what on earth are the `cli` and `sei` function calls for?

Let's look at the documentation of the `I` bit in the `SREG` register:

![The status register](./sreg.png)

What we learn from here is that in AVR, a status bit `I` exists, which globally controls the interrupt delivery.
With the `cli()` function or the `CLI` instruction in assembly, we can prevent interrupt delivery whereas `sei()` or
the `SEI` instruction turn on active interrupt delivery.

```c
	while (1) {
		print_timer_status();
		printf("Menu:\n");
		printf("1. Store secret data.\n");
		printf("2. Read secret data.\n");
		printf("3. Copy top secret data.\n");
		printf("4. Exit.\n");
		printf("Choice: ");
		read_data(buf);
		switch (buf[0]) {
			case '1':
			{
				printf("Secret: ");
				read_data(secret);
				break;
			}
			case '2':
			{
				printf("Stored secret:\n---\n%s\n---\n", secret);
				break;
			}
			case '3':
			{
				printf("Enter top secret data access code: ");
				read_data(buf);
				char pw_bad = 0;
				for (int i = 0; top_secret_password[i]; i++) {
					pw_bad |= top_secret_password[i]^buf[i];
				}
				if (pw_bad) {
					printf("Access denied.\n");
					break;
				}
				printf("Access granted.\nCopying top secret data...\n");
				timer_on_off(1);
				while (TCCR1B);
				printf("Done.\n");
				break;
			}
			case '4':
			{
				quit();
				break;
			}
			default:
			{
				printf("Invalid option.\n");
				break;
			}
		}
	}
	quit();
```

The last block of the `main` function throws us into another infinite loop. This time, we're given the
timer status and some instructions on commands that we can use by entering the respective number. With
`1`, we can override the contents of `secret` - unfortunately `read_data` on a buffer of 256 bytes, so
yet again no stack smashing possibility. `2` lets us dump the contents of `secret`, although that's
useless without the contents of `top_secet_data` being copied over to `secret` first. `3` allows for
copying "top secret data" by prompting us for a second password first. If we pass that, we'll end up
in the Counter 1 overflow interrupt handler until all the bytes have been copied. All the other command
variants just end up either prompting for other input or just a call to `quit` that halts the CPU.

And prolly here's the next problem. Even if we figure out the "top secret data access code" and data is
copied, a `break` followed by `quit()` is hit immediately after that, rendering the effort completely
useless...?!

Drowning in questions and problems here in the late hours of the night, let's break them down one by one
after I got some sleep. :)
