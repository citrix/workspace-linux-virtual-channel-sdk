# Programming Guide

Virtual channels are referred to by a seven-character (or shorter) ASCII
name. In several previous versions of the ICA protocol, virtual channels
were numbered; the numbers are now assigned dynamically based on the
ASCII name, making implementation easier.

When developing virtual channel code for internal use only, you can use
any seven-character name that does not conflict with existing virtual
channels. Use only upper and lowercase ASCII letters and numbers. Follow
the existing naming convention when adding your own virtual channels.

The predefined channels, which begin with the OEM identifier CTX, are
for use only by Citrix.


## Design Suggestions

Follow these suggestions to make your virtual channels easier to design
and enhance:

-   When you design your own virtual channel protocol, allow for the
    flexibility to add features. Virtual channels have version numbers
    that are exchanged during initialization so that both the client and
    the server detect the maximum level of functionality that can
    be used. For example, if the client is at Version 3 and the server
    is at Version 5, the server does not send any packets with
    functionality beyond Version 3 because the client does not know how
    to interpret the newer packets.

-   Because the server side of a virtual channel protocol can be
    implemented as a separate process, it is easier to write code that
    interfaces with the Citrix-provided virtual channel support on the
    server than on the client (where the code must fit into an existing
    code structure). The server side of a virtual channel simply opens
    the channel, reads from and writes to it, and closes it when done.

Writing code for the server-side is similar to writing an application,
which uses services exported by the system. It is easier to write an
application to handle the virtual channel communication because it can
then be run once for each ICA connection supporting the virtual
channel.

Writing for the client-side is similar to writing a driver, which must
provide services to the system in addition to using system services.
If a service is written, it must manage multiple connections.

-   If you are designing new hardware for use with new virtual channels
    (for example, an improved compressed video format), make sure the
    hardware can be detected so that the client can determine whether or
    not it is installed. Then the client can communicate to the server
    if the hardware is available before the server uses the new
    data format. Optionally, you could have the virtual driver translate
    the new data format for use with older hardware.

-   There might be limitations preventing your new virtual channel from
    performing at an optimum level. If the client is connecting to the
    server running XenApp through a low-speed connection, the bandwidth
    might not be great enough to properly support audio or video data.
    You can make your protocol adaptive, so that as bandwidth decreases,
    performance degrades gracefully, possibly by sending sound normally
    but reducing the frame rate of the video to fit the
    available bandwidth.

-   To identify where problems are occurring (connection,
    implementation, or protocol), first get the connection and
    communication working. Then, after the virtual channel is complete
    and debugged, do some time trials and record the results. These
    results establish a baseline for measuring further optimizations
    such as compression and other enhancements so that the channel
    requires less bandwidth.

-   The time stamp in the pVdPoll variable can be helpful for resolving
    timing issues in your virtual driver. It is a ULONG containing the
    current time in milliseconds. The pVdPoll variable is a pointer to a
    DLLPOLL structure. See dllapi.h (in base/inc/) for definitions of
    these structures.


## Client-Side Functions Overview

The client software is built on a modular configurable architecture that
allows replaceable, configurable modules (such as virtual channel
drivers) to handle various aspects of an ICA connection. These modules
are specially formatted and dynamically loadable. To accomplish this
modular capability, each module (including virtual channel drivers)
implements a fixed set of function entry points.

There are six groups of functions:
user-defined, virtual driver helper, memory INI, Workspace app for Linux
sub-window interface, Workspace app for Linux event interface, and Workspace app
for Linux timer interface.

###  User-Defined Functions

To make writing virtual channels easier, dynamic loading is handled by
the WinStation driver, which in turn calls user-defined functions. This
simplifies creating the virtual channel because all you have to do is
fill in the functions and link your virtual channel driver with vdapi.a
(provided with this SDK).

 | Function | Description |
 |----------|-------------|
 | DriverClose | Frees private driver data. Called before unloading a virtual driver (generally upon client exit). |
 | DriverGetLastError | Returns the last error set by the virtual driver. Not used; links with the common front end, VDAPI. |
 | DriverInfo | Retrieves information about the virtual driver. |
 | DriverOpen | Performs all initialization for the virtual driver. Called once when the client loads the virtual driver (at startup).|
 | DriverPoll | Allows driver to check timers and other state information, sends queued data to the server, and performs any other required processing. Called periodically to see if the virtual driver has any data to write. |
 | DriverQueryInformation | Retrieves run-time information from the virtual driver. |
 | DriverSetInformation | Sets run-time information in the virtual driver. |
 | ICADataArrival | Indicates that data was delivered. Called when data arrives on the virtual channel. |

### Virtual Driver Helper Functions

The virtual driver uses helper functions to send data and manage the
virtual channel. When the WinStation driver initializes the virtual
driver, the WinStation driver passes pointers to helper functions and
the virtual driver passes pointers to the user-defined functions. Newer
API functions QueueVirtualWrite, MM_\*, Evt_\*, and Tmr_\* helper
functions are callable directly by the virtual driver.

VdCallWd is linked in as part of VDAPI and is available in all
user-implemented functions. The others are obtained during DriverOpen
when VdCallWd is called with the WDxSETINFORMATION parameter.

 | Function | Description |
 |----------|-------------|
 | QueueVirtualWrite | Queues a virtual write and stimulates packet output if required allowing the data to be sent without waiting for the poll. This must be used to send data to the server in all newly written virtual drivers. This replaces the deprecated functions below. |
 | AppendVdHeader (**Deprecated**) |  Appends a virtual driver header to a buffer.|
 | OutBufAppend (**Deprecated**)  |   Appends data to a buffer. |
 | OutBufReserve (**Deprecated**)  |  Checks for available output buffer space. |
 | OutBufWrite (**Deprecated**)    |  Sends the buffer to the server. |
 | VdCallWd     | Used to query and set information from the WinStation driver (WD). |
                   
### Memory INI Functions

Memory INI functions read data from the client engine configuration
files stored in both the client installation directory for system wide
settings and \$HOME/.ICAClient for user specific settings.

For each entry in appsrv.ini and wfclient.ini, there must be a
corresponding entry in All_Regions.ini for the setting to take effect.
For more information, refer to All_Regions.ini file in the
\$ICAROOT/config directory.

 | Function | Description |
 |----------|-------------|
 | miGetPrivateProfileBool  |   Returns a boolean value. |
|  miGetPrivateProfileInt   |   Returns an integer value. |
 | miGetPrivateProfileLong  |   Returns a long value. |
 | miGetPrivateProfileString  | Returns a string value. |
 
### Workspace app for Linux Sub-Window Interface

Workspace app for Linux sub-window interface allows a virtual channel to gain
access to a sub-window of the client session in order to draw within a
session. The sub-window interface is not designed to take keyboard and
mouse input. It is simply for rendering graphics.

 | Function | Description |
 |----------|-------------|
 | MM_clip | Sets the shape of the window. |
 | MM_destroy_window | Destroys a window created by MM_get_window. |
 | MM_get_window  | Creates an operating system window that is a sub-window of an existing session window. |
| MM_set_geometry | Sets the size and position of a session sub-window. |
| MM_show_window |  Makes a window visible. |
 | MM_TWI_clear_new_window_function | Clears the callback for seamless window creation. |
 | MM_TWI_set_new_window_function| Adds a callback for seamless window creation. |

### Workspace app for Linux Event (Evt) Interface

Workspace app for Linux event interface allows a virtual channel to select on
a given file descriptor in the Workspace app for Linux event loop and receive
a callback from the Workspace app for Linux event loop when the given
conditions are met.

| Function | Description |
|----------|-------------|
| Evt_create | Allocates an event structure that can be used to fire a callback on an event.|
| Evt_destroy | Destroys previously created event structure. |
| Evt_remove_triggers | Removes any previously added file descriptor selections on a given file descriptor. |
| Evt_remove_triggers | Removes any previously added file descriptor selections on a given file descriptor. |
| Evt_signal | Calls the function stored in the event structure. |
| Evt_trigger_for_input | Connects the callback of an event structure to be triggered on the given file descriptor satisfying the input conditions. |
| Evt_trigger_for_output | Connects the callback of an event structure to be triggered on the given file descriptor satisfying the output conditions. |
         
### Workspace app for Linux Timer (Tmr) Interface

Workspace app for Linux timer interface allows a virtual channel to set up a
recurrent timer that invokes a given callback. The timer is attached to
the event loop of the Workspace app for Linux and is called from the event
loop when the timer fires.

| Function | Description |
|----------|-------------|
| Tmr_create | Creates a timer object and returns its handle.|
| Tmr_destroy | Destroys a timer object given a printer to its handle and sets the handle to NULL. |
| Tmr_setEnabled | Enables or disables a timer object. |
| Tmr_setPeriod  |  Sets the timeout period for a timer. |
  

