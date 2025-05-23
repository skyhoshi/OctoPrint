.. _sec-api-printer:

******************
Printer operations
******************

.. contents::

Printer control is mostly achieved through the use of commands, issued to resources reflecting components of the
printer. OctoPrint currently knows the following components:

Print head
  Print head commands allow jogging and homing the print head in all three axes. Querying the resource is currently
  not supported.
  See :ref:`sec-api-printer-printheadcommand`.
Tool
  Tool commands allow setting the temperature and temperature offsets for the printer's hotends/tools. Querying the
  corresponding resource returns temperature information including an optional history.
  See :ref:`sec-api-printer-toolcommand`.
Bed
  Bed commands allow setting the temperature and temperature offset for the printer's heated bed. Querying the
  corresponding resource returns temperature information including an optional history. Note that Bed commands
  are only available if the currently selected printer profile has a heated bed.
  See :ref:`sec-api-printer-bedcommand`.
Chamber
  Chamber commands allow setting the temperature and temperature offset for the printer's heated chamber. Querying
  the corresponding resource returns temperature information including an option history. Note that Chamber commands
  are only available if the currently selected printer profile has a heated chamber.
  See :ref:`sec-api-printer-chambercommand`.
SD card
  SD commands allow initialization, refresh and release of the printer's SD card (if available). Querying the
  corresponding resource returns the current SD card state.
  See :ref:`sec-api-printer-sdcommand`.

Besides that, OctoPrint also provides a :ref:`full state report of the printer <sec-api-printer-state>`.

.. note::

   You might be wondering why all command responses below only return a ``204`` with an empty body instead of
   the output of the just sent commands. There are two reasons for this.

   OctoPrint's internal webserver is single threaded and can only handle one request at a time. This is
   not a problem generally since asynchronous programming allows to just have one request which is waiting for
   data from a long running backend operation to sleep while handling other requests. The internal framework
   used for providing the REST API though, Flask, is based on WSGI, which is synchronous in nature. This means
   that it is impossible to wait in a non blocking wait while handling a request on the REST API. So in order to
   return the response of a command sent to the printer, the single thread of the webserver would have to be blocked
   by the API while the response wasn't available yet. Which in turn would mean that the whole web server would
   stop responding while waiting for the printer to reply, which, depending on the command in question (e.g. homing)
   can take a long while. That would be a bad idea.

   The second reason is that thanks to a large number of firmwares out there having a `particular bug <https://github.com/MarlinFirmware/Marlin/commit/acc0e7527914948656ccabba35f7faedc94ef885>`_
   that makes it impossible to track the output of sent commands, identifying the correct response to a given
   sent command is pretty much hit-and-miss. That situation gets even worse considering that it's next to impossible
   to distinguish firmware versions which have that bug from firmware versions which don't have it.

   Hence OctoPrint currently doesn't offer any synchronous way of retrieving the output of responses from the printer.
   If you need the printer's serial communication, you'll need to subscribe to :ref:`push updates <sec-api-push>`.

.. _sec-api-printer-state:

Retrieve the current printer state
==================================

.. http:get:: /api/printer

   Retrieves the current state of the printer. Returned information includes:

   * temperature information (see also :ref:`Retrieve the current tool state <sec-api-printer-toolstate>` and
     :ref:`Retrieve the current bed state <sec-api-printer-bedstate>`)
   * sd state (if available, see also :ref:`Retrieve the current SD state <sec-api-printer-sdstate>`)
   * general printer state

   Temperature information can also be made to include the printer's temperature history by supplying the ``history``
   query parameter. The amount of data points to return here can be limited using the ``limit`` query parameter.

   Clients can specific a list of attributes to not return in the response (e.g. if they don't need it) via the
   ``exclude`` query parameter.

   Returns a :http:statuscode:`200` with a :ref:`Full State Response <sec-api-printer-datamodel-fullstate>` in the
   body upon success.

   Requires the ``STATUS`` permission.

   **Example 1**

   Include temperature history data, but limit it to two entries.

   .. sourcecode:: http

      GET /api/printer?history=true&limit=2 HTTP/1.1
      Host: example.com
      X-Api-Key: abcdef...

   .. sourcecode:: http

      HTTP/1.1 200 OK
      Content-Type: application/json

      {
        "temperature": {
          "tool0": {
            "actual": 214.8821,
            "target": 220.0,
            "offset": 0
          },
          "tool1": {
            "actual": 25.3,
            "target": null,
            "offset": 0
          },
          "bed": {
            "actual": 50.221,
            "target": 70.0,
            "offset": 5
          },
          "history": [
            {
              "time": 1395651928,
              "tool0": {
                "actual": 214.8821,
                "target": 220.0
              },
              "tool1": {
                "actual": 25.3,
                "target": null
              },
              "bed": {
                "actual": 50.221,
                "target": 70.0
              }
            },
            {
              "time": 1395651926,
              "tool0": {
                "actual": 212.32,
                "target": 220.0
              },
              "tool1": {
                "actual": 25.1,
                "target": null
              },
              "bed": {
                "actual": 49.1123,
                "target": 70.0
              }
            }
          ]
        },
        "sd": {
          "ready": true
        },
        "state": {
          "text": "Operational",
          "flags": {
            "operational": true,
            "paused": false,
            "printing": false,
            "cancelling": false,
            "pausing": false,
            "sdReady": true,
            "error": false,
            "ready": true,
            "closedOrError": false
          }
        }
      }

   **Example 2**

   Exclude temperature and sd data.

   .. sourcecode:: http

      GET /api/printer?exclude=temperature,sd HTTP/1.1
      Host: example.com
      X-Api-Key: abcdef...

   .. sourcecode:: http

      HTTP/1.1 200 OK
      Content-Type: application/json

      {
        "state": {
          "text": "Operational",
          "flags": {
            "operational": true,
            "paused": false,
            "printing": false,
            "cancelling": false,
            "pausing": false,
            "sdReady": true,
            "error": false,
            "ready": true,
            "closedOrError": false
          }
        }
      }

   :query exclude:  An optional comma-separated list of fields to exclude from the response (e.g. if not needed by
                    the client). Valid values to supply here are ``temperature``, ``sd`` and ``state``.
   :query history:  If set to ``true`` (or: ``yes``, ``y``, ``1``), history information will be included in the response
                    too. If no ``limit`` parameter is given, all available temperature history data will be returned.
   :query limit:    If set to an integer (``n``), only the last ``n`` data points from the printer's temperature history
                    will be returned. Will be ignored if ``history`` is not enabled.
   :statuscode 200: No error
   :statuscode 409: If the printer is not operational.

.. _sec-api-printer-printheadcommand:

Issue a print head command
==========================

.. http:post:: /api/printer/printhead

   Print head commands allow jogging and homing the print head in all three axes. Available commands are:

   jog
     Jogs the print head (relatively) by a defined amount in one or more axes. Additional parameters are:

     * ``x``: Optional. Amount/coordinate to jog print head on x axis, must be a valid number corresponding to the distance to travel in mm.
     * ``y``: Optional. Amount/coordinate to jog print head on y axis, must be a valid number corresponding to the distance to travel in mm.
     * ``z``: Optional. Amount/coordinate to jog print head on z axis, must be a valid number corresponding to the distance to travel in mm.
     * ``absolute``: Optional. Boolean value specifying whether to move relative to current position (provided
       axes values are relative amounts) or to absolute position (provided axes values are coordinates)
     * ``speed``: Optional. Speed at which to move. If not provided, minimum speed for all selected axes from printer
       profile will be used. If provided but ``false``, no speed parameter will be appended to the command. Otherwise
       interpreted as an integer signifying the speed in mm/min, to append to the command.

   home
     Homes the print head in all of the given axes. Additional parameters are:

     * ``axes``: A list of axes which to home, valid values are one or more of ``x``, ``y``, ``z``.

   feedrate
     Changes the feedrate factor to apply to the movements of the axes.

     * ``factor``: The new factor, percentage between 50 and 200% as integer (``50`` to ``200``) or float (``0.5`` to ``2.0``).

   All of these commands except ``feedrate`` may only be sent if the printer is currently operational and not printing.
   Otherwise a :http:statuscode:`409` is returned.

   Upon success, a status code of :http:statuscode:`204` and an empty body is returned.

   Requires the ``CONTROL`` permission.

   **Example Jog Request**

   Jog the print head by 10mm in X, -5mm in Y and 0.02mm in Z.

   .. sourcecode:: http

      POST /api/printer/printhead HTTP/1.1
      Host: example.com
      Content-Type: application/json
      X-Api-Key: abcdef...

      {
        "command": "jog",
        "x": 10,
        "y": -5,
        "z": 0.02
      }

   .. sourcecode:: http

      HTTP/1.1 204 No Content

   **Example Home Request**

   Home the X and Y axes.

   .. sourcecode:: http

      POST /api/printer/printhead HTTP/1.1
      Host: example.com
      Content-Type: application/json
      X-Api-Key: abcdef...

      {
        "command": "home",
        "axes": ["x", "y"]
      }

   .. sourcecode:: http

      HTTP/1.1 204 No Content

   **Example feed rate request (1/2)**

   Set the feed rate factor to 105% using an integer argument.

   .. sourcecode:: http

      POST /api/printer/printhead HTTP/1.1
      Host: example.com
      Content-Type: application/json
      X-Api-Key: abcdef...

      {
        "command": "feedrate",
        "factor": 105
      }

   .. sourcecode:: http

      HTTP/1.1 204 No Content

   **Example feed rate request (2/2)**

   Set the feed rate factor to 105% using a float argument.

   .. sourcecode:: http

      POST /api/printer/printhead HTTP/1.1
      Host: example.com
      Content-Type: application/json
      X-Api-Key: abcdef...

      {
        "command": "feedrate",
        "factor": 1.05
      }

   .. sourcecode:: http

      HTTP/1.1 204 No Content

   :json string command: The command to issue, either ``jog`` or ``home``.
   :json number x:       ``jog`` command: The amount to travel on the X axis in mm.
   :json number y:       ``jog`` command: The amount to travel on the Y axis in mm.
   :json number z:       ``jog`` command: The amount to travel on the Z axis in mm.
   :json array axes:     ``home`` command: The axes which to home, valid values are one or more of ``x``, ``y`` and ``z``.
   :json number factor:  ``feedrate`` command: The factor to apply to the feed rate, percentage between 50 and 200% as integer or float.
   :statuscode 204: No error
   :statuscode 400: Invalid axis specified, invalid value for travel amount for a jog command or factor for feed rate or otherwise invalid
                    request.
   :statuscode 409: If the printer is not operational or currently printing.

.. _sec-api-printer-toolcommand:

Issue a tool command
====================

.. http:post:: /api/printer/tool

   Tool commands allow setting the temperature and temperature offsets for the printer's tools (hotends), selecting
   the current tool and extruding/retracting from the currently selected tool. Available commands are:

   target
     Sets the given target temperature on the printer's tools. Additional parameters:

     * ``targets``: Target temperature(s) to set, properties must match the format ``tool`` or ``tool{n}`` with ``n`` being the
       tool's index starting with 0. ``tool`` signifies the currently active extruder. A target value of `0` will turn the heater off.

   offset
     Sets the given temperature offset on the printer's tools. Additional parameters:

     * ``offsets``: Offset(s) to set, properties must match the format ``tool{n}`` with ``n`` being the tool's index
       starting with 0.

   select
     Selects the printer's current tool. Additional parameters:

     * ``tool``: Tool to select, format ``tool{n}`` with ``n`` being the tool's index starting with 0.

   extrude
     Extrudes the given amount of filament from the currently selected tool. Additional parameters:

     * ``amount``: The amount of filament to extrude in mm. May be negative to retract.
     * ``speed``: Optional. Speed at which to extrude. If not provided, maximum speed for E axis from printer
       profile will be used. Otherwise interpreted as an integer signifying the speed in mm/min, to append to the command.

   flowrate
     Changes the flow rate factor to apply to extrusion of the tool.

     * ``factor``: The new factor, percentage between 75 and 125% as integer (``75`` to ``125``) or float (``0.75`` to ``1.25``).

   All of these commands may only be sent if the printer is currently operational and -- in case of ``select`` and
   ``extrude`` -- not printing. Otherwise a :http:statuscode:`409` is returned.

   Upon success, a status code of :http:statuscode:`204` and an empty body is returned.

   Requires the ``CONTROL`` permission.

   **Example Target Temperature Request**

   Set the target temperature for the printer's first hotend to 220°C and the printer's second extruder to 205°C.

   .. sourcecode:: http

      POST /api/printer/tool HTTP/1.1
      Host: example.com
      Content-Type: application/json
      X-Api-Key: abcdef...

      {
        "command": "target",
        "targets": {
          "tool0": 220,
          "tool1": 205
        }
      }

   .. sourcecode:: http

      HTTP/1.1 204 No Content

   **Example Target Temperature Request (Current Tool Only)**

   Set the target temperature for the printer's currently active hotend to 220°C.

   .. sourcecode:: http

      POST /api/printer/tool HTTP/1.1
      Host: example.com
      Content-Type: application/json
      X-Api-Key: abcdef...

      {
        "command": "target",
        "targets": {
          "tool": 220
        }
      }

   .. sourcecode:: http

      HTTP/1.1 204 No Content

   **Example Offset Temperature Request**

   Set the offset for temperatures on ``tool0`` to +10°C and on ``tool1`` to -5°C.

   .. sourcecode:: http

      POST /api/printer/tool HTTP/1.1
      Host: example.com
      Content-Type: application/json
      X-Api-Key: abcdef...

      {
        "command": "offset",
        "offsets": {
          "tool0": 10,
          "tool1": -5
        }
      }

   .. sourcecode:: http

      HTTP/1.1 204 No Content

   **Example Tool Select Request**

   Select the second hotend of the printer for any following ``extrude`` commands.

   .. sourcecode:: http

      POST /api/printer/tool HTTP/1.1
      Host: example.com
      Content-Type: application/json
      X-Api-Key: abcdef...

      {
        "command": "select",
        "tool": "tool1"
      }

   .. sourcecode:: http

      HTTP/1.1 204 No Content

   **Example Extrude Request**

   Extrude 5mm on the currently selected tool.

   .. sourcecode:: http

      POST /api/printer/tool HTTP/1.1
      Host: example.com
      Content-Type: application/json
      X-Api-Key: abcdef...

      {
        "command": "extrude",
        "amount": 5
      }

   .. sourcecode:: http

      HTTP/1.1 204 No Content

   **Example Retract Request**

   Retract 3mm of filament on the currently selected tool.

   .. sourcecode:: http

      POST /api/printer/tool HTTP/1.1
      Host: example.com
      Content-Type: application/json
      X-Api-Key: abcdef...

      {
        "command": "extrude",
        "amount": -3
      }

   .. sourcecode:: http

      HTTP/1.1 204 No Content

   **Example flow rate request (1/2)**

   Set the flow rate factor to 95% using an integer attribute.

   .. sourcecode:: http

      POST /api/printer/tool HTTP/1.1
      Host: example.com
      Content-Type: application/json
      X-Api-Key: abcdef...

      {
        "command": "flowrate",
        "factor": 95
      }

   .. sourcecode:: http

      HTTP/1.1 204 No Content

   **Example flow rate request (2/2)**

   Set the flow rate factor to 95% using a float attribute.

   .. sourcecode:: http

      POST /api/printer/tool HTTP/1.1
      Host: example.com
      Content-Type: application/json
      X-Api-Key: abcdef...

      {
        "command": "flowrate",
        "factor": 0.95
      }

   .. sourcecode:: http

      HTTP/1.1 204 No Content

   :json string command: The command to issue, either ``target``, ``offset``, ``select`` or ``extrude``.
   :json object targets: ``target`` command: The target temperatures to set. Valid properties have to match the format ``tool{n}``.
   :json object offsets: ``offset`` command: The offset temperature to set. Valid properties have to match the format ``tool{n}``.
   :json object tool:    ``select`` command: The tool to select, value has to match the format ``tool{n}``.
   :json object amount:  ``extrude`` command: The amount of filament to extrude from the currently selected tool.
   :json number factor:  ``flowrate`` command: The factor to apply to the flow rate, percentage between 75 and 125% as integer or float.
   :statuscode 204: No error
   :statuscode 400: If ``targets`` or ``offsets`` contains a property or ``tool`` contains a value not matching the format
                    ``tool{n}``, the target/offset temperature, extrusion amount or flow rate factor is not a valid number or outside of
                    the supported range, or if the request is otherwise invalid.
   :statuscode 409: If the printer is not operational or -- in case of ``select`` or ``extrude`` -- currently printing.

.. _sec-api-printer-toolstate:

Retrieve the current tool state
===============================

.. http:get:: /api/printer/tool

   Retrieves the current temperature data (actual, target and offset) plus optionally a (limited) history (actual, target,
   timestamp) for all of the printer's available tools.

   It's also possible to retrieve the temperature history by supplying the ``history`` query parameter set to ``true``. The
   amount of returned history data points can be limited using the ``limit`` query parameter.

   Returns a :http:statuscode:`200` with a Temperature Response in the body upon success.

   Requires the ``STATUS`` permission.

   .. note::
      If you want both tool and bed temperature information at the same time, take a look at
      :ref:`Retrieve the current printer state <sec-api-printer-state>`.

   **Example**

   Query the tool temperature data and also include the temperature history but limit it to two entries.

   .. sourcecode:: http

      GET /api/printer/tool?history=true&limit=2 HTTP/1.1
      Host: example.com
      X-Api-Key: abcdef...

   .. sourcecode:: http

      HTTP/1.1 200 OK
      Content-Type: application/json

      {
        "tool0": {
          "actual": 214.8821,
          "target": 220.0,
          "offset": 0
        },
        "tool1": {
          "actual": 25.3,
          "target": null,
          "offset": 0
        },
        "history": [
          {
            "time": 1395651928,
            "tool0": {
              "actual": 214.8821,
              "target": 220.0
            },
            "tool1": {
              "actual": 25.3,
              "target": null
            }
          },
          {
            "time": 1395651926,
            "tool0": {
              "actual": 212.32,
              "target": 220.0
            },
            "tool1": {
              "actual": 25.1
            }
          }
        ]
      }

   :query history:  If set to ``true`` (or: ``yes``, ``y``, ``1``), history information will be included in the response
                    too. If no ``limit`` parameter is given, all available temperature history data will be returned.
   :query limit:    If set to an integer (``n``), only the last ``n`` data points from the printer's temperature history
                    will be returned. Will be ignored if ``history`` is not enabled.
   :statuscode 200: No error
   :statuscode 409: If the printer is not operational.

.. _sec-api-printer-bedcommand:

Issue a bed command
===================

.. http:post:: /api/printer/bed

   Bed commands allow setting the temperature and temperature offsets for the printer's heated bed. Available commands
   are:

   target
     Sets the given target temperature on the printer's bed. Additional parameters:

     * ``target``: Target temperature to set. A value of `0` will turn the heater off.

   offset
     Sets the given temperature offset on the printer's bed. Additional parameters:

     * ``offset``: Offset to set.

   All of these commands may only be sent if the printer is currently operational. Otherwise a :http:statuscode:`409`
   is returned.

   Upon success, a status code of :http:statuscode:`204` and an empty body is returned.

   If no heated bed is configured for the currently selected printer profile, the resource will return
   an :http:statuscode:`409`.

   Requires the ``CONTROL`` permission.

   **Example Target Temperature Request**

   Set the target temperature for the printer's heated bed to 75°C.

   .. sourcecode:: http

      POST /api/printer/bed HTTP/1.1
      Host: example.com
      Content-Type: application/json
      X-Api-Key: abcdef...

      {
        "command": "target",
        "target": 75
      }

   .. sourcecode:: http

      HTTP/1.1 204 No Content

   **Example Offset Temperature Request**

   Set the temperature offset for the heated bed to -5°C.

   .. sourcecode:: http

      POST /api/printer/bed HTTP/1.1
      Host: example.com
      Content-Type: application/json
      X-Api-Key: abcdef...

      {
        "command": "offset",
        "offset": -5
      }

   .. sourcecode:: http

      HTTP/1.1 204 No Content

   :json string command: The command to issue, either ``target`` or ``offset``.
   :json object target: ``target`` command: The target temperature to set.
   :json object offset: ``offset`` command: The offset temperature to set.
   :statuscode 204: No error
   :statuscode 400: If ``target`` or ``offset`` is not a valid number or outside of the supported range, or if the
                    request is otherwise invalid.
   :statuscode 409: If the printer is not operational or the selected printer profile
                    does not have a heated bed.

.. _sec-api-printer-bedstate:

Retrieve the current bed state
==============================

.. http:get:: /api/printer/bed

   Retrieves the current temperature data (actual, target and offset) plus optionally a (limited) history (actual, target,
   timestamp) for the printer's heated bed.

   It's also possible to retrieve the temperature history by supplying the ``history`` query parameter set to ``true``. The
   amount of returned history data points can be limited using the ``limit`` query parameter.

   Returns a :http:statuscode:`200` with a Temperature Response in the body upon success.

   If no heated bed is configured for the currently selected printer profile, the resource will return
   an :http:statuscode:`409`.

   Requires the ``STATUS`` permission.

   .. note::
      If you want tool, bed and chamber temperature information at the same time, take a look at
      :ref:`Retrieve the current printer state <sec-api-printer-state>`.

   **Example**

   Query the bed temperature data and also include the temperature history but limit it to two entries.

   .. sourcecode:: http

      GET /api/printer/bed?history=true&limit=2 HTTP/1.1
      Host: example.com
      X-Api-Key: abcdef...

   .. sourcecode:: http

      HTTP/1.1 200 OK
      Content-Type: application/json

      {
        "bed": {
          "actual": 50.221,
          "target": 70.0,
          "offset": 5
        },
        "history": [
          {
            "time": 1395651928,
            "bed": {
              "actual": 50.221,
              "target": 70.0
            }
          },
          {
            "time": 1395651926,
            "bed": {
              "actual": 49.1123,
              "target": 70.0
            }
          }
        ]
      }

   :query history:  If set to ``true`` (or: ``yes``, ``y``, ``1``), history information will be included in the response
                    too. If no ``limit`` parameter is given, all available temperature history data will be returned.
   :query limit:    If set to an integer (``n``), only the last ``n`` data points from the printer's temperature history
                    will be returned. Will be ignored if ``history`` is not enabled.
   :statuscode 200: No error
   :statuscode 409: If the printer is not operational or the selected printer profile
                    does not have a heated bed.

.. _sec-api-printer-chambercommand:

Issue a chamber command
=======================

.. http:post:: /api/printer/chamber

   Chamber commands allow setting the temperature and temperature offsets for the printer's heated chamber. Available commands
   are:

   target
     Sets the given target temperature on the printer's chamber. Additional parameters:

     * ``target``: Target temperature to set. A value of `0` will turn the heater off.

   offset
     Sets the given temperature offset on the printer's chamber. Additional parameters:

     * ``offset``: Offset to set.

   All of these commands may only be sent if the printer is currently operational. Otherwise a :http:statuscode:`409`
   is returned.

   Upon success, a status code of :http:statuscode:`204` and an empty body is returned.

   If no heated chamber is configured for the currently selected printer profile, the resource will return
   an :http:statuscode:`409`.

   Requires the ``CONTROL`` permission.

   **Example Target Temperature Request**

   Set the target temperature for the printer's heated chamber to 50°C.

   .. sourcecode:: http

      POST /api/printer/chamber HTTP/1.1
      Host: example.com
      Content-Type: application/json
      X-Api-Key: abcdef...

      {
        "command": "target",
        "target": 50
      }

   .. sourcecode:: http

      HTTP/1.1 204 No Content

   **Example Offset Temperature Request**

   Set the temperature offset for the heated chamber to -5°C.

   .. sourcecode:: http

      POST /api/printer/chamber HTTP/1.1
      Host: example.com
      Content-Type: application/json
      X-Api-Key: abcdef...

      {
        "command": "offset",
        "offset": -5
      }

   .. sourcecode:: http

      HTTP/1.1 204 No Content

   :json string command: The command to issue, either ``target`` or ``offset``.
   :json object target: ``target`` command: The target temperature to set.
   :json object offset: ``offset`` command: The offset temperature to set.
   :statuscode 204: No error
   :statuscode 400: If ``target`` or ``offset`` is not a valid number or outside of the supported range, or if the
                    request is otherwise invalid.
   :statuscode 409: If the printer is not operational or the selected printer profile
                    does not have a heated chamber.

.. _sec-api-printer-chamberstate:

Retrieve the current chamber state
==================================

.. http:get:: /api/printer/chamber

   Retrieves the current temperature data (actual, target and offset) plus optionally a (limited) history (actual, target,
   timestamp) for the printer's heated chamber.

   It's also possible to retrieve the temperature history by supplying the ``history`` query parameter set to ``true``. The
   amount of returned history data points can be limited using the ``limit`` query parameter.

   Returns a :http:statuscode:`200` with a Temperature Response in the body upon success.

   If no heated chamber is configured for the currently selected printer profile, the resource will return
   an :http:statuscode:`409`.

   Requires the ``STATUS`` permission.

   .. note::
      If you want tool, bed and chamber temperature information at the same time, take a look at
      :ref:`Retrieve the current printer state <sec-api-printer-state>`.

   **Example**

   Query the chamber temperature data and also include the temperature history but limit it to two entries.

   .. sourcecode:: http

      GET /api/printer/chamber?history=true&limit=2 HTTP/1.1
      Host: example.com
      X-Api-Key: abcdef...

   .. sourcecode:: http

      HTTP/1.1 200 OK
      Content-Type: application/json

      {
        "chamber": {
          "actual": 50.221,
          "target": 70.0,
          "offset": 5
        },
        "history": [
          {
            "time": 1395651928,
            "chamber": {
              "actual": 50.221,
              "target": 70.0
            }
          },
          {
            "time": 1395651926,
            "chamber": {
              "actual": 49.1123,
              "target": 70.0
            }
          }
        ]
      }

   :query history:  If set to ``true`` (or: ``yes``, ``y``, ``1``), history information will be included in the response
                    too. If no ``limit`` parameter is given, all available temperature history data will be returned.
   :query limit:    If set to an integer (``n``), only the last ``n`` data points from the printer's temperature history
                    will be returned. Will be ignored if ``history`` is not enabled.
   :statuscode 200: No error
   :statuscode 409: If the printer is not operational or the selected printer profile
                    does not have a heated chamber.

.. _sec-api-printer-sdcommand:

Issue an SD command
===================

.. http:post:: /api/printer/sd

   SD commands allow initialization, refresh and release of the printer's SD card (if available).

   Available commands are:

   init
     Initializes the printer's SD card, making it available for use. This also includes an initial retrieval of the
     list of files currently stored on the SD card, so after issuing that command a :ref:`retrieval of the files
     on SD card <sec-api-fileops-retrievelocation>` will return a successful result.

     .. note::
        If OctoPrint detects the availability of a SD card on the printer during connection, it will automatically attempt
        to initialize it.

   refresh
     Refreshes the list of files stored on the printer's SD card. Will return a :http:statuscode:`409` if the card
     has not been initialized yet (see the ``init`` command and :ref:`SD state <sec-api-printer-sdstate>`).

   release
     Releases the SD card from the printer. The reverse operation to ``init``. After issuing this command, the SD
     card won't be available anymore, hence and operations targeting files stored on it will fail. Will return a :http:statuscode:`409`
     if the card has not been initialized yet (see the ``init`` command and :ref:`SD state <sec-api-printer-sdstate>`).

   Upon success, a status code of :http:statuscode:`204` and an empty body is returned.

   Requires the ``CONTROL`` permission.

   **Example Init Request**

   Initialize the SD card.

   .. sourcecode:: http

      POST /api/printer/sd HTTP/1.1
      Host: example.com
      Content-Type: application/json
      X-Api-Key: abcdef...

      {
        "command": "init"
      }

   .. sourcecode:: http

      HTTP/1.1 204 No Content

   **Example Refresh Request**

   Refresh the file list of the SD card

   .. sourcecode:: http

      POST /api/printer/sd HTTP/1.1
      Host: example.com
      Content-Type: application/json
      X-Api-Key: abcdef...

      {
        "command": "refresh"
      }

   .. sourcecode:: http

      HTTP/1.1 204 No Content

   **Example Release Request**

   Release the SD card

   .. sourcecode:: http

      POST /api/printer/sd HTTP/1.1
      Host: example.com
      Content-Type: application/json
      X-Api-Key: abcdef...

      {
        "command": "release"
      }

   .. sourcecode:: http

      HTTP/1.1 204 No Content

   :json string command: The command to issue, either ``init``, ``refresh`` or ``release``.
   :statuscode 204:      No error
   :statuscode 409:      If a ``refresh`` or ``release`` command is issued but the SD card has not been initialized (e.g.
                         via ``init``.

.. _sec-api-printer-sdstate:

Retrieve the current SD state
=============================

.. http:get:: /api/printer/sd

   Retrieves the current state of the printer's SD card. For this request no authentication is needed.

   If SD support has been disabled in OctoPrint's settings, a :http:statuscode:`404` is returned.

   Returns a :http:statuscode:`200` with an :ref:`SD State Response <sec-api-printer-datamodel-sdstate>` in the body
   upon success.

   Requires the ``STATUS`` permission.

   **Example**

   Read the current state of the SD card.

   .. sourcecode:: http

      GET /api/printer/sd HTTP/1.1
      Host: example.com
      X-Api-Key: abcdef...

   .. sourcecode:: http

      HTTP/1.1 200 OK
      Content-Type: application/json

      {
        "ready": true
      }

   :statuscode 200: No error
   :statuscode 404: If SD support has been disabled in OctoPrint's config.

.. _sec-api-printer-error:

Retrieve information about the last error
=========================================

.. http:get:: /api/printer/error

   Retrieves information about the last error that occurred on the printer.

   Returns a :http:statuscode:`200` with an :ref:`Error Information <sec-api-printer-datamodel-errorinfo>` in the body
   upon success.

   Requires the ``STATUS`` permission and ``MONITOR_TERMINAL`` to also get access to the last terminal lines associated with the error.

   **Example**

   Read the last error that occurred on the printer.

   .. sourcecode:: http

      GET /api/printer/error HTTP/1.1
      Host: example.com
      X-Api-Key: abcdef...

   .. sourcecode:: http

      HTTP/1.1 200 OK
      Content-Type: application/json

      {
        "consequence": "emergency",
        "error": "MINTEMP triggered, OMG, kill() called - Not SD printing",
        "faq": "firmware-mintemp",
        "logs": [
          "Send: N2685 G1 X147.748 Y108.411 E627.83763*85",
          "Recv: ok",
          "Send: N2686 G1 X148.522 Y108.286 E627.8963*98",
          "Recv: ok",
          "Send: N2687 G1 X148.866 Y108.174 E627.92338*87",
          "Recv: ok",
          "Send: N2688 G1 X149.494 Y107.868 E627.97566*91",
          "Recv: ok",
          "Send: N2689 G1 X149.731 Y107.779 E627.9946*96",
          "Recv: ok",
          "Send: N2690 G1 X149.69 Y108.032 E628.01378*101",
          "Recv: ok",
          "Send: N2691 G1 X147.252 Y112.252 E628.3785*107",
          "Recv: ok",
          "Send: N2692 G1 X145.082 Y112.253 E628.54089*93",
          "Recv: ok",
          "Recv: Error: MINTEMP triggered, kill() called",
          "Recv: Not SD printing",
          "Changing monitoring state from \"Printing\" to \"Error\""
        ],
        "reason": "firmware"
      }

   :statuscode 200: No error

.. _sec-api-printer-arbcommand:

Send an arbitrary command to the printer
========================================

.. http:post:: /api/printer/command

   Sends any command to the printer via the serial interface. Should be used with some care as some commands can interfere with
   or even stop a running print job.

   Expects a :ref:`Arbitrary Command Request <sec-api-printer-datamodel-arbcommand>` as the request's body.

   If successful returns a :http:statuscode:`204` and an empty body.

   Requires the ``CONTROL`` permission.

   **Example for sending a single command**

   .. sourcecode:: http

      POST /api/printer/command HTTP/1.1
      Host: example.com
      Content-Type: application/json
      X-Api-Key: abcdef...

      {
        "command": "M106"
      }

   .. sourcecode:: http

      HTTP/1.1 204 No Content

   **Example for sending multiple commands**

   .. sourcecode:: http

      POST /api/printer/command HTTP/1.1
      Host: example.com
      Content-Type: application/json
      X-Api-Key: abcdef...

      {
        "commands": [
          "M18",
          "M106 S0"
        ]
      }

   .. sourcecode:: http

      HTTP/1.1 204 No Content

   :json string command:  Single command to send to the printer, mutually exclusive with ``commands``.
   :json string commands: List of commands to send to the printer, mutually exclusive with ``command``.
   :statuscode 204:       No error

.. _sec-api-printer-customcontrols:

Retrieve custom controls
========================

.. http:get:: /api/printer/command/custom

   Retrieves the :ref:`custom controls <sec-features-custom_controls>` as configured in
   :ref:`config.yaml <sec-configuration-config_yaml>`.

   Please refer to the documentation of :ref:`custom controls <sec-features-custom_controls>` on what
   data structure to expect here.

   Returns a :http:statuscode:`200` with an :ref:`Custom Controls Response <sec-api-printer-datamodel-customcontrols>`
   in the body upon success.

   Requires the ``CONTROL`` permission.

   :statuscode 200: No error

.. _sec-api-printer-datamodel:

Data model
==========

.. _sec-api-printer-datamodel-fullstate:

Full State Response
-------------------

.. list-table::
   :widths: 15 5 10 30
   :header-rows: 1

   * - Name
     - Multiplicity
     - Type
     - Description
   * - ``temperature``
     - 0..1
     - :ref:`Temperature State <sec-api-printer-datamodel-temps>`
     - The printer's temperature state data
   * - ``sd``
     - 0..1
     - :ref:`SD State <sec-api-printer-datamodel-sdstate>`
     - The printer's sd state data
   * - ``state``
     - 0..1
     - :ref:`Printer State <sec-api-datamodel-printer-state>`
     - The printer's general state

.. _sec-api-printer-datamodel-temps:

Temperature State
-----------------

.. list-table::
   :widths: 15 5 10 30
   :header-rows: 1

   * - Name
     - Multiplicity
     - Type
     - Description
   * - ``tool{n}``
     - 0..*
     - :ref:`Temperature Data <sec-api-datamodel-printer-tempdata>`
     - Current temperature stats for tool *n*. Enumeration starts at 0 for the first tool. Not included if querying
       only bed state.
   * - ``bed``
     - 0..1
     - :ref:`Temperature Data <sec-api-datamodel-printer-tempdata>`
     - Current temperature stats for the printer's heated bed. Not included if querying only tool state or if
       the currently selected printer profile does not have a heated bed.
   * - ``history``
     - 0..1
     - List of :ref:`Historic Temperature Datapoint <sec-api-datamodel-printer-temphistory>`
     - Temperature history

.. _sec-api-printer-datamodel-sdstate:

SD State
--------

.. list-table::
   :widths: 15 5 10 30
   :header-rows: 1

   * - Name
     - Multiplicity
     - Type
     - Description
   * - ``ready``
     - 1
     - Boolean
     - Whether the SD card has been initialized (``true``) or not (``false``).

.. _sec-api-printer-datamodel-arbcommand:

Arbitrary Command Request
-------------------------

.. list-table::
   :widths: 15 5 10 30
   :header-rows: 1

   * - Name
     - Multiplicity
     - Type
     - Description
   * - ``command``
     - 0..1
     - String
     - Single command to send to the printer, mutually exclusive with ``commands`` and ``script``.
   * - ``commands``
     - 0..*
     - Array of String
     - Multiple commands to send to the printer (in the given order), mutually exclusive with ``command`` and ``script``.
   * - ``script``
     - 0..*
     - String
     - Name of the GCODE script template to send to the printer, mutually exclusive with ``command`` and ``commands``.
   * - ``parameters``
     - 0..1
     - Map of key value pairs
     - Key value pairs of parameters to replace in sent commands/provide to the script renderer
   * - ``context``
     - 0..1
     - Map of key value pairs
     - (only if ``script`` is set) additional template variables to provide to the script renderer

.. _sec-api-printer-datamodel-customcontrols:

Custom Controls Response
------------------------

.. list-table::
   :widths: 15 5 10 30
   :header-rows: 1

   * - Name
     - Multiplicity
     - Type
     - Description
   * - ``controls``
     - 0..n
     - List of :ref:`custom controls <sec-features-custom_controls>`
     - A list of custom control definitions as defined in ``config.yaml``.

.. _sec-api-printer-datamodel-errorinfo:

Error Information
-----------------

.. list-table::
   :widths: 15 5 10 30
   :header-rows: 1

   * - Name
     - Multiplicity
     - Type
     - Description
   * - ``error``
     - 1
     - String
     - The error message, will be empty if no error information is currently present.
   * - ``reason``
     - 1
     - String
     - The reason for the error, e.g. ``firmware`` in case of errors triggered by the firmware. Will be empty if no
       error information is currently present.
   * - ``consequence``
     - 0..1
     - String
     - The consequence of the error, e.g. ``emergency`` if an emergency stop and disconnect was triggered, ``disconnect``
       if just a disconnect was triggered, ``cancel`` if the ongoing print job was cancelled. Will be unset if
       no consequence was triggered or no error information is currently present.
   * - ``faq``
     - 0..1
     - String
     - An FAQ id for the error, to be added to ``https://faq.octoprint.org/`` for a link to the FAQ entry. Will be unset
       if no FAQ entry is available for the error or no error information is currently present.
   * - ``logs``
     - 0..*
     - List of String
     - A list of log lines that might be relevant for the error. Will be unset if no error information is currently
       present or if the requesting user lacks the ``MONITOR_TERMINAL`` permission.
