.. _mpsl_fem:

Front-End Module feature
########################

The Front-End Module feature allows the application to interface with several types of Front-End Modules (FEMs).
This allows to increase the transmitted power through a Power Amplifier (PA) or increase the sensitivity through a Low-Noise Amplifier (LNA).
Any increase in power and sensitivity results in an increased communication range.
The exact PA and LNA gains are dependent on the specific FEM used.

Implementation
**************

Two FEM implementations are provided:

* *nRF21540 GPIO* - compatible with nRF21540 FEM, implementing a 3-pin interface.
* *Simple GPIO* - simplified version for supporting other Front-End Modules, implementing a 2-pin interface.

Both implementations use PA and LNA pins for controlling the FEM.
Additionally, the nRF21540 GPIO implementation uses the PDN pin for powering down FEM internal circuits to reduce energy consumption.

Configurable timings
********************

In both implementations, two timings can be configured:

* ``LNA time gap`` - the time between LNA activation and the start of radio reception.
* ``PA time gap`` - the time between PA activation and the start of radio transmission.

For nRF21540, two additional timings can also be configured:

* ``TRX hold time`` - the time interval for which the FEM is kept powered up after the PDN deactivation event occurs.
* ``PDN settle time`` - the time interval before the PA or LNA activation reserved for the FEM ramp-up.

General usage
*************

The Power Amplifier and the Low-Noise Amplifier are responsible for, respectively, transmission and reception and are configured and activated independently.
The two functionalities can't be configured and set to operate simultaneously as they share some resources.
As such, after operating with a Power Amplifier, the PA configuration must be cleared to be able to configure and use a Low-Noise Amplifier afterward, and vice versa.

Both amplifiers are controlled through activation and deactivation events.
Two types of events are supported:

* *timer event* - ``COMPARE`` event of a hardware timer, can be used for both PA/LNA activation and deactivation.
* *generic event* - any other event type, can only be used for PA/LNA deactivation.

Preparing a generic event only requires the application to provide the event register.
Preparing a timer event requires the application to provide:

* The instance of the timer, which the protocol has to start by itself.
* The *Compare Channels* mask, which tells the Front End Module which Compare Channels of the provided Timer are free to use.
* The Start time, at which the Front End Module can start preparing the PA or LNA.
* The End time, at which the Front End Module must be ready for the RF procedure.

The module will then configure the timer so that the FEM is activated or deactivated accordingly, taking into account the FEM ramp-up time.

See below for an example of activating LNA for Rx operation, using the following parameters:

* Time scheduled by the application (RX ramp-up) - 40 us
* LNA ramp-up time - ``13 us``
* LNA deactivation event - ``rx_end``
* LNA activation timer - ``TIMER0``

See below for the steps needed to properly configure LNA in this example:

* The application configures the LNA to be activated by the timer event, with the start time set to 0 us and the end time set to 40 us.
* The application provides the ``rx_end`` event as the LNA deactivation event.
* The FEM module reads the scheduled time and sets ``TIMER0`` compare channel to 27 us, as a result of the RX ramp-up time (40 us) minus the LNA ramp-up time (13 us).
* The application starts the RX operation.
* The application starts ``TIMER0``.

The following picture illustrates the timings in this scenario:

.. figure:: pic/FEM_timing_simple.svg
   :alt: Timing of LNA pin for reception

   Timing of LNA pin for reception

The following picture illustrates the calls between the application, the FEM module, and the hardware in this scenario:

.. figure:: pic/FEM_sequence_simple.svg
   :alt: Sequence diagram of LNA control for reception

   Sequence diagram of LNA control for reception

nRF21540 usage
**************

In the nRF21540 implementation, the PDN pin is used to power down the FEM internal circuits.
The FEM can be powered down on application request, configuring the activation timer event, similarly to PA and LNA pins.
The FEM is powered back up automatically before PA or LNA are activated.

See below for an example of controlling LNA and PDN during Rx operation, using the following parameters:

* Time scheduled by application - 40 us
* LNA ramp-up time - 13 us
* PDN settle time - 18 us
* TRX hold time - 5 us
* LNA deactivation event - ``rx_end``
* PDN deactivation event - ``rx_end``
* PDN activation timer - ``TIMER0``
* LNA activation timer - ``TIMER1``

See below for the steps needed to properly configure LNA and PDN in this example:

* The application configures the power-down by passing ``rx_end`` as the activation event.
* The FEM module connects the activation event with the ``TIMER0`` start task through PPI and sets TRX hold time to 5 us.
* The application configures LNA to be activated by the timer event, with the start time set to 0 us and end time set to 40 us.
* The application provides the ``rx_end`` event as the LNA deactivation event.
* The FEM module reads the scheduled time and sets ``TIMER1`` compare channels to 27 us (40-13) and 9 us (27-18).
* The application starts Rx operation.
* The application starts ``TIMER1``.

The following picture illustrates the timing in this scenario:

.. figure:: pic/FEM_timing_nRF21540.svg
   :alt: Timing of LNA and PDN pins for reception

   Timing of LNA and PDN pins for reception

The following picture presents the calls between the application, the FEM module, and the hardware in this scenario:

.. figure:: pic/FEM_sequence_nRF21540.svg
   :alt: Sequence diagram of LNA and PDN control for reception

   Sequence diagram of LNA and PDN control for reception