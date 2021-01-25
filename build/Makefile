##### Configuration zone #############################################

# library release
RELEASE=1.0

# user space local install root directory
USRLOCALROOT=$(HOME)/.avrlocal

# testing platform
PLATFORM=ArduinoONE

# device file to upload tests
DEVICE=/dev/ttyACM0

# private library headers (not needed by the library end user)
PRIVATE_HEADERS = queue.h i2cq.h i2cr.h

# public library headers (required by the library end user)
PUBLIC_HEADERS  = adc.h ticker.h pin.h switch.h timer.h alert.h serial.h i2c.h

# library modules (object files in the library; file suffix not needed)
SRC_MODS = queue i2cq adc ticker pin switch timer alert serial i2c

# library tests/examples
SRC_TESTS = test_timer test_timer2 \
            test_pin \
            test_ticker \
	    test_adc test_adc_2 \
            test_switch_1 test_switch_2 test_switch_3 \
            test_alert_1 \
	    test_serial_1 test_serial_2 test_serial_3 test_serial_4 \
            test_serial_5

# Link rules for tests/examples (may have specific platform requirements to run)
test_adc: adc.o ticker.o test_fixture.o alert.o pin.o
test_adc_2: adc.o ticker.o test_fixture.o alert.o pin.o
test_ticker: ticker.o test_fixture.o
test_timer: pin.o timer.o
test_timer2: pin.o timer.o
test_pin: pin.o
test_switch_1: switch.o test_fixture.o
test_switch_2: switch.o test_fixture.o
test_switch_3: switch.o test_fixture.o ticker.o
test_switch_4: switch.o test_fixture.o ticker.o adc.o
test_alert_1: alert.o pin.o
test_serial_1: serial.o queue.o
test_serial_2: serial.o queue.o
test_serial_3: serial.o queue.o adc.o ticker.o alert.o pin.o
test_serial_4: serial.o queue.o adc.o ticker.o alert.o pin.o
test_serial_5: serial.o queue.o adc.o ticker.o alert.o pin.o


##### Internal configs ##########################################


# processing tools and processing flags

CC=avr-gcc
CPPFLAGS=
CFLAGS=-Wall -std=gnu99 -Os -fshort-enums
LDFLAGS=-std=c99  # only to link examples
AR=avr-ar
STRIP=avr-strip

# Non modificable names

SRC_DIST=libaire-$(RELEASE)
SRC_LIB = libaire.a


# Platform and shield dependent parameters

ifeq ($(PLATFORM),ArduinoONE)
  MCU = atmega328p
  PROGRAMMER = arduino
  DUDEOPTS =
  FREQ = 16000000UL
else ifeq ($(PLATFORM),ArduinoMEGA)
  MCU = atmega2560
  PROGRAMMER = wiring
  DUDEOPTS = -D
  FREQ = 16000000UL
endif


# Processing parameters

CFLAGS   +=  -mmcu=$(MCU)
LDFLAGS  +=  -mmcu=$(MCU)
CPPFLAGS +=  -D$(PLATFORM) -DF_CPU=$(FREQ) -I$(SRCDIR)


# Project structure

DEPSDIR = .deps
SRCDIR  = ../src
TESTDIR = ../test
BUILDDIR = .

# Sets of files

SOURCES = $(SRC_MODS) $(SRC_TESTS)
DEPFILES = $(addprefix $(DEPSDIR)/, $(addsuffix .d, $(SOURCES)))

# Search path

VPATH = $(SRCDIR):$(TESTDIR)


##### Main targets #######################################################

.PHONY: lib tests install dist clean veryclean

.DEFAULT_GOAL := lib

# Library

$(SRC_LIB): $(SRC_LIB)($(addsuffix .o, $(SRC_MODS)))
lib:	    $(SRC_LIB)

# tests

tests:  $(SRC_TESTS)

# Package distribution

$(SRC_DIST).tar.gz: $(SRC_LIB) $(PUBLIC_HEADERS)
	install -D -m 444 -t tmpdist/lib $(SRC_LIB)
	for f in $(PUBLIC_HEADERS); \
           do install -D -m 444  -t tmpdist/include $(SRCDIR)/$$f; \
        done
	tar zcf $(SRC_DIST).tar.gz -C tmpdist .
	rm -rf tmpdist


dist:   $(SRC_DIST).tar.gz

# Install in user local space

install: lib
	install -D -m 444 -t $(USRLOCALROOT)/lib $(SRC_LIB)
	for f in $(PUBLIC_HEADERS); \
           do install -D -m 444  -t $(USRLOCALROOT)/include $(SRCDIR)/$$f; \
        done

# Utility targets

clean:
	@\rm -f *~ *.o *.s *.hex

veryclean: clean
	@\rm -f  \#*\# $(SRC_LIB) $(SRC_TESTS) $(SRC_DIST).tar.gz
	@\rm -rf $(DEPSDIR)



##### Generic rules ########################################

%.hex: %
	avr-objcopy -Oihex  $< $@

upload_%: %.hex
	avrdude $(DUDEOPTS) -c $(PROGRAMMER) -p $(MCU) -P $(DEVICE) -U $<

%.s: %.c
	$(CC) $(CPPFLAGS) $(CFLAGS) -S $<

$(DEPSDIR)/%.d: %.c | $(DEPSDIR)
	@set -e; rm -f $@; \
        $(CC) -MM $(CPPFLAGS) $< > $@.$$$$; \
        sed 's,\($*\)\.o[ :]*,\1.o $@ : ,g' < $@.$$$$ > $@; \
	rm -f $@.$$$$

$(DEPSDIR):
	 @mkdir -p $@


##### Automagic dependencies ###############################

-include $(DEPFILES)
