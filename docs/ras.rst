RAS support in Trusted Firmware-A
=================================

.. section-numbering::
    :suffix: .

.. contents::
    :depth: 2

.. |EHF| replace:: Exception Handling Framework
.. |TF-A| replace:: Trusted Firmware-A

This document describes |TF-A| support for Arm Reliability, Availability, and
Serviceability (RAS) extensions. RAS is a mandatory extension for Armv8.2 and
later CPUs, and also an optional extension to the base v8.0 architecture.

In conjunction with the `|EHF|`__, support for RAS extension enables
firmware-first paradigm for handling platform errors, in which exceptions
resulting from errors—viz. Synchronous External Abort (SEA), Asynchronous
External Abort (signalled as SErrors), Fault Handling and Error Recovery
interrupts—are routed to and and handled in EL3. The |EHF| document mentions
various `error handling use-cases`__.

.. __: exception-handling.rst
.. __: exception-handling.rst#delegation-use-cases

For the description of Arm RAS extensions, Standard Error Records, and the
precise definition of RAS terminology, please refer to the Arm Architecture
Reference Manual. The rest of this document assumes familiarity with
architecture and terminology.

Overview
--------

As mentioned above, the RAS framework enables handling of exceptions resulting
from platform errors to be routed to and handled in EL3. The RAS framework in
|TF-A| allows the platform to define External Abort handler and register RAS
nodes with error records, and interrupts. The framework also provides
`helpers`__ for accessing Standard Error Records as introduced by the RAS
extensions.

.. __: `Standard Error Record helpers`_

The build option ``RAS_EXTENSION`` when set to ``1`` enables the RAS framework;
``EL3_EXCEPTION_HANDLING`` must also be set ``1``.

Platform APIs
-------------

The RAS framework allows the platform to define handlers for External Abort,
Uncontainable Errors, Double Fault, and errors rising from EL3 execution. Please
refer to the porting guide for the `RAS platform API descriptions`__.

.. __: porting-guide.rst#external-abort-handling-and-ras-support

Registering RAS error records
-----------------------------

RAS nodes are components in the system capable of signalling errors to PEs
through one one of the notification mechanisms—SEAs, SErrors, or interrupts. RAS
nodes contain one or more error records, which are registers through which the
nodes advertise various properties of the signalled error. Arm recommends that
error records are implemented in the Standard Error Record format. The RAS
architecture allows for error records to be accessible via. system or
memory-mapped registers.

The platform should enumerate the error records providing for each of them:

-  A handler to probe the error record for errors;
-  If probing identfies an error, a handler to handle it;
-  For memory-mapped error record, its base address and size in KB; for a system
   register-accessed record, the start index of the record, and number of
   continuous records from that index;
-  Any node-specific auxiliary data.

With this information supplied, when the run time firmware receives one of the
notification mechanisms, the RAS framework can iterate through and probe error
records for error, and invoke the appropriate handler to handle it.

The RAS framework provides the macros to populate error record information. The
macros are versioned, and the latest version as of this writing is 1. These
macros create a strcture of type ``err_record_info`` from its arguments, which
are later passed to probe and error handlers.

For memory-mapped error records:

.. code:: c

    ERR_RECORD_MEMMAP_V1(base_addr, size_num_k, probe, handler, aux)

And, for system register ones:

.. code:: c

    ERR_RECORD_SYSREG_V1(idx_start, num_idx, probe, handler, aux)

The probe handler must have the following prototype:

.. code:: c

    typedef int (*err_record_probe_t)(const struct err_record_info *info,
                    int *probe_data);

The ``probe_data`` output parameter can be used to pass any useful information
resulting from probe to the error handler (see below). For example, in case of
system register error records, it could be the index of the record.

The error handler must have the following prototype:

.. code:: c

    typedef int (*err_record_handler_t)(const struct err_record_info *info,
               int probe_data, const struct err_handler_data *const data);

The platform is expected populate an array with error record information, and
register the array with the RAS framework using the macro
``REGISTER_ERR_RECORD_INFO()``, passing it the name of the array describing the
records. Note that the macro must be used in the same file where the array is
defined.

Registering for RAS interrupts
------------------------------

Double-fault handling
---------------------

Interaction with |EHF|
----------------------

Standard Error Record helpers
-----------------------------
