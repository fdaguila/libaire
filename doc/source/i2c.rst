**************
The i2c module
**************

Introduction
============

The module adc contains an abstract interface to the AVR i2c bus. This
interface leverages the user of the protocol details making easy and
reliable to use the i2c bus. Implementation is based on interruptions
and only master-send and master-receive modes are available. It is
assumed that there is single master.

The module assumes that a single i2c bus exists. Operations are
classified in the following groups:

* Management operations

  Operations devoted to manage the module (initialization, etc.).

* Block transfer operations

  Operations devoted to send and receive a bunch of bytes.

* Byte transfer operations

  Operations devoted to send or receive single bytes.

* Canned operations

  Utility operations that combine several basic operations into a
  single one.

  


Module API
==========


Module management operations
----------------------------

The module must be initialized once by calling i2c_setup(). No other
module operation can be called before.

.. doxygenfunction:: i2c_setup

i2c bus use should be enclosed by a open-close parenthesis following
the well known semantics of files. No send or receive operation should
be used while i2c in closed state. i2c can be reopened after being
closed if needed. 

.. doxygenfunction:: i2c_open

.. doxygenfunction:: i2c_close


Given the asyncronous nature of this module operations, them are
non-blocking. User could relay on this when designing
software. However, the number of pending requests is finite. If a new
operation is requested when no space left for requests, the operation
blocks until issuing the request is feasible. Next operation lets to
foresee this condition.

.. doxygenfunction:: i2c_swamped



Block transfer operations
-------------------------

These are operations intended to transfer blocks of several bytes
to/from a given slave in the bus. The operations require a buffer
where data to be sent resides. This buffer:

1. Cannot be reused or unallocated until operation finished. This
   should be accounted given the asyncronous nature of the
   operations.

2. Two operations O1 and O2 cannot share the same buffer unless
   provisions are made that guarantee that O1 finished before O2 is
   executed.

.. doxygenfunction:: i2c_send

.. doxygenfunction:: i2c_receive


Byte transfer operations
------------------------

These are operations intended to transfer one or two bytes to/from a
given slave in the bus.

The send operations doesn't require a buffer and data to be sent is
internally stored by the module. This allows for a smoother use.

.. doxygenfunction:: i2c_send_uint8

.. doxygenfunction:: i2c_send_2uint8

The i2c_send_uint16() function sends a word. A provision is made to
allow the user to choose the byte order in which the word is sent.

.. doxygenfunction:: i2c_send_uint16

The module also provides a function to receive a byte as defined below.		     

.. doxygenfunction:: i2c_receive_uint8
		     


Canned operations
-----------------
		     
The module offers an operation to cope with the usual pattern of "send
an address and then receive content".

.. doxygenfunction:: i2c_sandr

.. warning::
   i2c_sandr() do not checks the status of the send part. Thus, if the
   send part fails, the consequent receive part would return
   unexpected data.

		     

Status value
------------

Transfer operations in this module have an out parameter of type
i2c_status_t that encodes the current state of the transfer. The
value of this should be considered volatile as it is asyncronouly
changed while the operation is alive.

.. doxygenenum:: i2c_status_t
   :outline:

The distinct values should be interpreted following this table:

.. list-table:: Status values
   :header-rows: 1

   * - Value
     - Operation ended?
     - All is ok?
     - Meaning
   * - Running
     - no
     - yes
     - Operation not finished
   * - Success
     - yes
     - yes
     - Operation was ok
   * - ReceivedMessageLenError
     - yes
     - no
     - The message received was shorter than expected
   * - SlaveRejected
     - yes
     - no
     - The destination slave rejected the data transfer
   * - SlaveDiscardedData
     - yes
     - no
     - The slave rejected to receive more data. Not all data was sent.
   * - InternalError
     - yes
     - no
     - An unexpected error arose. Usually due to an unexpected event.
   
The pointer to the status object could be eventually NULL. In this
case operations understand that user is not interested in how and when
the operation finished.

By similar reasons that the data buffers, a great care should be put
when trying to share the status object between distinct operations.






Implementation notes
====================

START/RESTART/STOP policy
-------------------------

The implementation tries to make most of its transfer without freeing
the bus control. Then, while there are requests to be served into the
queue a RESTART condition is signaled when a new request begins to be
served. Only when there are no more requests to be served a STOP is
signaled to the bus.


Non responding slave
--------------------

When the master tries to talk a slave by sending a SLA+R or SLA+W to
the bus and the slave do not responds by any reason, an error is
returned and the request is discarded.





Examples
========

Example 1
---------

An example where an RTC DS1307 is queried every 500 ms and the results
printed over the serial port.

.. code-block:: c

   #include <avr/interrupt.h>
   #include <util/delay.h>
   #include <stdio.h>
   #include "serial.h"
   #include "i2c.h"


   #define RTC_ADDRESS (0x68)



   static int write(char s, FILE *stream) {
     if (s == '\n'){
       serial_write('\r');
       serial_write('\n');
     } else serial_write(s);
     return 0;
   }

   static FILE mystdout = FDEV_SETUP_STREAM(write, NULL,
					    _FDEV_SETUP_WRITE);



   static uint8_t bcd2dec(uint8_t num)
   {
     return ((num/16 * 10) + (num % 16));
   }


   int main(){
       uint8_t buf[10];
       volatile i2c_status_t st;

       serial_setup();
       i2c_setup();
       sei();

       stdout = &mystdout;
       serial_open();
       i2c_open();

       puts("== slaves found in bus:");
       for (uint8_t sla=1; sla<0x7f; sla++) {  // SLA=0 broadcast reserved
	 i2c_send_uint8(sla, 0, &st);
	 while(!st);
	 if (st == Success)
	   printf(" -> %x\n", sla);
       }

       puts("\n== begin test");

       // reset RTC and start it (use a literal buffer)
       i2c_send(RTC_ADDRESS, (uint8_t[]){0,0x05}, 2, NULL);

       for(;;) {
	 // set address register to 0 and get time registers
	 i2c_sandr(RTC_ADDRESS,
		   (uint8_t[]){0}, 1,
		   buf, 7,
		   &st); 
	 while(!st);

	 // show registers in a nice way
	 printf("%02d:%02d:%02d ",
		bcd2dec(buf[2]),
		bcd2dec(buf[1]),
		bcd2dec(buf[0]));
	 printf("%02d/%02d/20%02d\n",
		bcd2dec(buf[4]),
		bcd2dec(buf[5]),
		bcd2dec(buf[6]));

	 _delay_ms(500);
       }

       i2c_close();
       serial_close();

       return 0;
   }

