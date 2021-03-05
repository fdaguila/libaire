**************
The adc module
**************

Introduction
============

The module adc contains an abstract interface to the AVR
analogic-to-digital conversion hardware.

An inherently complex issue in the AD conversor is the reliability of
the values obtained. Such values are influenced by a number of
details: the waiting time after changing the channel or the reference
source. This module hides to some extent the procedures of channel
change or reference source change while trying to guarantee reliable
enough read values. Note that some of the factors that induce
unreliable reading come from the electrical characteristics of the
signal source being converted. These kind of factors cannot be
alleviated by the module mechanisms.



From this module standpoint, the AVR ADC services are based on:

1. A set of physical analogic channels attached to physical pins (some
   of them shared with digital ports);
2. A set of reference sources that are used to
   compare with the analogic signals beig converted.
3. A hardware that reads a physical channel, compares its value to a
   reference source and returns the corresponding digital value.

The center of this module is the adc_channel. An adc_channel is a
logical file-like object that abstracts a physical analogic channel
together with its reference source.

The functions of this module are operations on the adc_channel. A
basic use to sample N times a single channel would follow this
pattern:

.. uml::

   participant User
   participant adc
   participant adc_channel

   User ->  adc             : bind
   User <-- adc             : adc_channel

   User -> adc_channel ++      : prepare
   User <-- adc_channel

   loop for each sample
      User -> adc_channel   : start_conversion
      activate adc_channel
      User <-- adc_channel

      loop while true
         User -> adc_channel   : converting
	 User <-- adc_channel  : true or false
      end

      deactivate adc_channel
      User ->  adc_channel --  : get sample
      User <-- adc_channel     : sampled value
   end

   User ->  adc_channel !!  : unbind


   
1. Bind the logical channel to a physical channel and to a reference
   source.
2. Prepare the logical channel to be sampled.
3. Request the channel to begin a sampling.
4. When sampled: get the value obtained.
5. Unbound the channel.

Some functions are canned combinations of basic operations.


Operation Modes
===============

Es poden definir 2 modes d'operació. El primer mode, que es pot
anomenar **Only**, està implementat i definit a la Introducció i a la
API. Aquest mode està basat en els modes propis del perifèric
*Single* i *Free Running*. En aquest mode es genera una única
mostra. Alternativament aquesta única mostra pot ser el promig de
N_SAMPLES mostres obtingudes de forma continuada. Per aconseguir
aquestes mostres, es fa servir el mode *Free Running*, que
consisteix en realitzar la nova conversió, tot just s'acabat
l'anterior. En aquest cas, el conversor obté les mostres a la màxima
velocitat possible.

Un segon mode anomenat **Periodic** genera mostres de forma continuada
amb 2 opcions. Una primera opció és que les generi a la màxima
velocitat que pot el convertidor, per tant estarà basat en el mode
bàsic *Free Running* o bé que les generi en base a un senyal extern de
dispar i per tant basat en el mode *Triggered*. En aquest últim cas
cal coordinar-se amb els mòduls que generen la font de Trigger.

La gestió dels canals de conversió posterior a la inicialització del
sistema es fa molt complicada i potser no té gaire sentit. Redefinir
freqüències de mostreig quan un canal s'incorpora o es treu pot donar
lloc a situacions on els temps de conversió, combinat amb el període
de mostreig per canal generi una situació inviable. Per aquest motiu
la idea és fer un planificador de mostreig comú a tots els
protothreads que incorpori tots els canals que es volen mostrejar
definit a la inicialització global del sistema. El disseny d'aquest
planificador pot venir donat per una eina exterior
(feta p. ex. python) que determini els paràmetres d'inicialització.

Les dades d'entrada a aquesta eina planificadora hauria de ser els
canals que es volen mostrejar i la seva freqüència de mostreig. La
sortida hauria de donar:

1. La configuració amb la freqüència de funcionament del senyal de
   *Trigger*.
2. Una llista amb la seqüència de canals sobre els que s'anirà
   generant cada mostra. A cada senyal de *Trigger*, el convertidor
   escollirà el següent canal de la llista seguint un estratègia tipus
   Round-Robin. Si un dels canals es vol mostrejar més sovint es pot
   repetir dins aquesta llista. L'eina planificadora haurà de vigilar
   que es mantingui la periodicitat. Els valors dels possibles canals
   estan dins el rang [0:10]. Un exemple de planificador per 3 canals
   podria ser [0, 1, 0, 2]. En aquest cas el canal 0 es mostreja el
   doble de vegades que els canals 1 i 2.
3. Per a cada canal existent al planificador, ha de donar el nombre de
   mostres sobre les que es farà un filtrat abans de lliurar la mostra
   final com a resultat de la conversió. Si no es realitza cap
   filtrat, aquest nombre ha de ser 1. Aprofitant el cas anterior
   {c0:1, c1:2, c2:3} vol dir que al canal 0 no se li farà cap
   filtrat, al canal 1 es farà un filtrat de 2 mostres i el canal 3 un
   filtrat de 3 mostres.

En conclusió, aprofitant els exemples anteriors, si considerem que la
freqüència del senyal de Trigger és *Ft*, definint la mida de la
llista del planificador com *M*, les freqüències de mostreig de
cadascun dels 3 canals serà:

1. c0: Ft/M*2/1 On 2 és el nombre de vegades que c0 surt a la llista
   del planificador i 1 és el nombre de mostres del filtre.
   
2. c1: Ft/M*1/2
   
3. c3: Ft/M*1/3


L'estructura d'utilització del mòdul podria ser:
   
   1. Inicialització general del mòdul (abans d'interrupcions i 1 sola
      vegada) amb 'adc_setup()'.

   2. Definir els canals a mostrejar. Això es fa via un seguit
      d'ordres 'adc_bind()'. S'afegeix com a paràmetre el nombre de
      mostres sobre el que es farà un filtrat tipus CMA Cummulative
      Moving Average.

   3. Definir la planificació de mostreig. Aquesta planificació
      s'inicialitza amb 'adc_start_planning()'. Segueix amb succesives
      crides a 'adc_add_planning()' passant com a paràmetre un dels
      canal als que s'ha fet el binding anteriorment. Finalitza amb
      'adc_stop_planning()'. 'adc_add_planning()' accepta repeticions
      d'un canal concret dins del planificador. Serà responsabilitat
      del programador assegurar que es mantingui la periodicitat de
      mostreig d'un canal si 'adc_add_planning()' s'ha cridat més
      d'una vegada amb el mateix canal.

   4. Engegar **totes** les conversions periòdiques amb
      'adc_start_conversion()'.

   5. Consultar si la conversió ha acabat o no amb una funció
      'adc_converting()' que hauria de tenir com a paràmetre el
      'adc_channel' retornat per 'adc_bind()'. O bé cridar a un
      call-back cada vegada que es té una nova mostra. Aquest
      call-back s'hauria de passar també com a paràmetre a
      'adc_bind()'

   6. Llegir el valor del conversor amb 'adc_get()' passant com a
      paràmetre el 'adc_channel'.






Module API
==========


Sampled values range
--------------------

The sampling values obtained are always defined between zero and this
constant:

.. doxygendefine:: ADC_MAX

Additionally, a utility macro is defined that allows to map any sample
value obtained to a given range:

.. doxygendefine:: ADC_VALUE


Reference sources
-----------------

A logical channel is bound to a specific reference source. The signal
of this logical channel will be compared to the channel reference
source to get a digital value at sampling time. Several reference
sources are available.

The "external reference source" is a mode that allows for a external
voltage source to be used as a reference. The AVR A/D conversor poses
some contraints on the use of this reference: XX The module will
enforce this constraint.

.. doxygenenum:: adc_ref
		   

Basic operations
----------------

.. doxygenfunction:: adc_bind
.. doxygenfunction:: adc_unbind
.. doxygenfunction:: adc_prepare
.. doxygenfunction:: adc_start_conversion
.. doxygenfunction:: adc_converting
.. doxygenfunction:: adc_get
		     
Canned operations
-----------------

These are utility functions that include several basic operations on a
single channel frequently used together. The aim is to simplify the
usage of this module on simple cases.

.. doxygenfunction:: adc_prepare_start

.. doxygenfunction:: adc_prep_start_get
   

Examples
========

Example 1
---------


A basic example working with a single channel. We sample four times
the channel.

.. code-block:: c

   #include <adc.h>

   int main() {
     adc_channel c;

     /* bind the logical channel channel */
     c = adc_bind(3, Int11);
     /* prepare channel `c` to be sampled */
     adc_prepare(c);
     /* start conversion on prepared channel */
     for(int i=0; i<4; ++) {
       adc_start_conversion();
       /* wait sampling and conversion ends */
       while (adc_converting());
       /* output the result */
       put(adc_get());
     }

     return 0;
   }




   
Example 2
---------

In the following example, we practice round robin sampling on two
channels. Note how the slow `put()` operation is executed while
waiting for the next conversion done. This allows for a faster
sampling rate. The example uses canned operations when possible.

The time diagram of the central part of the algorithm is as follows:

.. uml::
   :scale: 70%
	   
   concise "analog signal 2" as s2
   concise "adc channel 2" as adc2
   concise "analog signal 1" as s1
   concise "adc channel 1" as adc1
   concise "main program" as main

   hide time-axis

   main is Run

   @0
   
   @10
   main -> adc1 : "prepare and start"
   adc1 is Prepare
   main is "Processing last s2 sample"
   
   @+6
   adc1 -> s1 : "sample"
   adc1 is Converting

   @18
   main is Run

   @20
   adc1 -> main : "sample value"
   adc1 is {hidden}

   @30
   main -> adc2 : "prepare and start"
   adc2 is Prepare
   main is "Processing last s1 sample"   
   
   @+6
   adc2 -> s2 : "sample"
   adc2 is Converting

   @38
   main is Run
   
   @40
   adc2 -> main : "sample value"
   adc2 is {hidden}
   
   highlight 8 to 22  #yellow:"Sampling channel 1"
   highlight 28 to 42 #yellow:"Sampling channel 2"

   @45


.. code-block:: c

   #include <adc.h>

   int main() {
     adc_channel c1, c2;
     uint8_t s1, s2;

     /* bind the logical channel channel */
     c1 = adc_bind(3, Int11);
     c2 = adc_bind(4, Int11);
     /* do sampling */
     s2 = adc_prepare_start_get(c2);
     for(;;) {
       adc_prepare_start(c1);
       put(s2);
       while (adc_converting());
       s1 = adc_get();
       adc_prepare_start(c2);
       put(s1);
       while (adc_converting());
       s2 = adc_get();       
     }

     return 0;
   }
   


