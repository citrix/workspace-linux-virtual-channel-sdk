# Programming Reference

For function summaries, see:

-   [Client-Side Functions Overview](./programming-guide/#client-side-functions-overview)

## AppendVdHeader (Deprecated)

!!!tip "Note"
		This function is deprecated. QueueVirtualWrite must be used in all
new virtual drivers.

Places an ICA virtual channel prefix on the output buffer prior to
assembling and sending the buffer.

### Calling Convention

```
INT WFCAPI AppendVdHeader (
	PWD pWd,
	USHORT Channel,
	USHORT ByteCount);
```

### Parameters

**pWD**

Pointer to a WinStation driver control structure.

**Channel**

Virtual channel number.

**ByteCount**

Actual size in bytes of the virtual channel packet data to be sent. Do
not include additional bytes reservered for the buffer overhead.

### Return Values

If the function succeeds, the return value is CLIENT_STATUS_SUCCESS.

If the function fails, the return value is the error code associated
with the failure; use GetLastError to get the extended error
information.

### Remarks

Call this function to prefix the virtual channel packet with the
appropriate header information. Normally the virtual driver sees only
the private packet data. However, when a virtual driver sends a virtual
channel packet to a server application, it must use this function to
prefix the data with the ICA header.

Use OutBufReserve to reserve a buffer prior to making this call. The
virtual driver must use this function immediately after a successful
OutBufReserve and before any other data is placed in the packet. This
action uses the additional four bytes requested in OutBufReserve, so do
not include this overhead in *ByteCount*.

If an ICA header or virtual channel data is appended to the buffer, the
buffer must be sent to the server before the control leaves the virtual
driver.

A pointer to this function is obtained from the VDWRITEHOOK structure
after hook registration in DriverOpen. The VDWRITEHOOK structure also
provides *pWd*.

## DriverClose

The WinStation driver calls this function prior to unloading the
virtual driver, when the ICA connection is being terminated.

### Calling Convention

```
INT Driverclose(
	PVD pVD,
	PDLLCLOSE pVdClose,
	PUINT16 puiSize);
```

### Parameters

**pVD**

Pointer to a virtual driver control structure.

**pVdClose**

Pointer to a standard driver close information structure.

**puiSize**

Pointer to the size of the driver close information structure. This is
an input parameter.

### Return Values

If the function succeeds the return value is CLIENT_STATUS_SUCCESS.

If the function fails, the return value is the CLIENT_ERROR_\* value
corresponding to the error condition; see clterr.h (in src/inc/) for a
list of error values beginning with CLIENT_ERROR.

### Remarks

When DriverClose is called, all private driver data is freed. The
virtual driver does not need to deallocate the virtual channel or write
hooks.

The pVdClose structure currently contains one element – NotUsed. This
structure can be ignored.

## DriverGetLastError

This function is not used but is available for linking with the common
front end, VDAPI.

### Calling Convention
```
INT DriverGetLastError(
	PVD pVD,
	PVDLASSTERROR pVdLastError);
```
### Parameters

**pVD**

Pointer to a virtual driver control structure.

**pVdLastError**

Pointer to a structure that receives the last error information.

### Return Value

The driver returns CLIENT_STATUS_SUCCESS.

### Remarks

This function currently has no practical significance for virtual
drivers; it is provided for compatibility with the loadable module
interface.

## DriverInfo

Gets information about the virtual driver, such as the version level of
the driver.

### Calling Convention

```
INT DriverInfo(
	PVD pVD,
	PDLLINFO pVdInfo,
	PUINT16 puiSize);
```

### Parameters

**pVD**

Pointer to a virtual driver control structure.

**pVdInfo**

Pointer to a standard driver information structure.

**puiSize**

Pointer to the size of the driver information structure. This is an
output parameter.

### Return Value

If the function succeeds, it returns CLIENT_STATUS_SUCCESS.

If the function fails because the buffer pointed to by pVdInfo is too
small, it returns CLIENT_ERROR_BUFFER_TOO_SMALL. Normally, when a
CLIENT_ERROR_\* result code is returned, the ICA session is
disconnected. CLIENT_ERROR_BUFFER_ TOO_SMALL is an exception and
does not result in the ICA session being disconnected. Instead, the
WinStation driver attempts to call DriverInfo again with the ByteCount
of pVdInfo returned by the failed call.

### Remarks

When the client starts, it calls this function to retrieve
module-specific information for transmission to the host. This
information is returned to the server side of the virtual channel by
WFVirtualChannelQuery.

The virtual driver must support this call by returning a structure in
the pVdInfo buffer. This structure can be a developer-defined virtual
channel-specific structure, but it must begin with a VD_C2H structure,
which in turn begins with a MODULE_C2H structure. All fields of the
VD_C2H structure must be filled in except for the ChannelMask field.
See ica-c2h.h (in src/inc/) for definitions of these structures.

The virtual driver must first check the size of the information buffer
given against the size that the virtual driver requires (the VD_C2H
structure). The size of the input buffer is given in
pVdInfo-&gt;ByteCount.

If the buffer is too small to store the information that the driver
needs to send, the correct size is filled into the ByteCount field and
the driver returns CLIENT_ERROR_BUFFER_TOO_SMALL.

If the buffer is large enough, the driver must fill it with a
module-defined structure. At a minimum, this structure must contain a
VD_C2H structure. The VD_C2H structure must be the first data in the
buffer; additional channel-specific data can follow. All relevant fields
of this structure are filled in by this function. The flow control
method is specified in the VDFLOW structure (an element of the VD_C2H
structure). The Ping example contains a flow control selection.

The WinStation driver calls this function twice at initialization, after
calling DriverOpen. The first call contains a NULL information buffer
and a buffer size of zero. The driver is expected to fill in
pVdInfo-&gt;ByteCount with the required buffer size and return
CLIENT_ERROR_BUFFER_TOO_SMALL. The WinStation driver allocates a
buffer of that size and retries the operation.

The data buffer pointed to by pVdinfo-&gt;pBuffer must not be changed by
the virtual driver. The WinStation driver stores byte swap information
in this buffer.

The parameter puiSize must be initialized to the size of the driver
information structure.

## DriverOpen

Initializes the virtual driver. The client engine calls this
user-written function once when the client is loaded.

### Calling Convention
```
INT DriverOpen(
	PVD pVD, PVDOPEN pVdOpen)
	PUINT16 puiSize);
```
### Parameters

**pVD**

Pointer to the virtual driver control structure. This pointer is passed
on every call to the virtual driver.

**pVdOpen**

Pointer to the virtual driver Open structure.

**puiSize**

Pointer to the size of the virtual driver Open structure. This is an
output parameter.

### Return Values

If the function succeeds, it returns CLIENT_STATUS_SUCCESS.

If the function fails, it returns the CLIENT_ERROR_\* value
corresponding to the error condition; see clterr.h (in src/inc/) for a
list of error values beginning with CLIENT_ERROR

### Remarks

The code fragments in this section are taken from the vdping example.

The DriverOpen function must:

&#49;.  Allocate a virtual channel.

Fill in a WDQUERYINFORMATION structure and call VdCallWd. The
WinStation driver fills in the OpenVirtualChannel structure (including
the channel number) and the data in pVd.

```
WDQUERYINFORMATION wdqi;
OPENVIRTUALCHANNEL OpenVirtualChannel;
UINT16 uiSize;
wdqi.WdInformationClass = WdOpenVirtualChannel;
wdqi.pWdInformation = &OpenVirtualChannel;
wdqi.WdInformationLength = sizeof(OPENVIRTUALCHANNEL);
OpenVirtualChannel.pVCName = CTXPING_VIRTUAL_CHANNEL_NAME;
uiSize = sizeof(WDQUERYINFORMATION);
rc = VdCallWd(pVd, WDxQUERYINFORMATION, &wdqi, &uiSize);
/* do error processing here */
```
After the call to VdCallWd, the channel number is assigned in the
OpenVirtualChannel structure's Channel element. Save the channel
number and set the channel mask to indicate which channel this driver
will handle.

For example:

```
g_usVirtualChannelNum = OpenVirtualChannel.Channel;
pVdOpen->ChannelMask = (1L << g_usVirtualChannelNum);
```

&#50;.  Optionally specify a pointer to a private data structure.

If you want the virtual driver to allocate memory for state data, it can
have a pointer to this data returned on each call by placing the pointer
in the virtual driver structure, as follows:
```
pVd->pPrivate = pMyStructure;
```

&#51;.  Exchange entry point data with the WinStation driver.

The virtual driver must register a write hook with the client WinStation
driver. The write hook is the entry point of the virtual driver to be
called when data is received for this virtual channel. The WinStation
driver returns pointers to functions that the driver must use to fill in
output buffers and sends data to the WinStation driver for transmission
to the server.

```
WDSETINFORMATION wdsi; VDWRITEHOOK vdwh;
// Fill in a write hook structure
vdwh.Type = g_usVirtualChannelNum; vdwh.pVdData = pVd;
vdwh.pProc = (PVDWRITEPROCEDURE) ICADataArrival;
// Fill in a set information structure
wdsi.WdInformationClass = WdVirtualWriteHook;
wdsi.pWdInformation = &vdwh;
wdsi.WdInformationLength = sizeof(VDWRITEHOOK);
uiSize = sizeof(WDSETINFORMATION);
rc = VdCallWd( pVd, WDxSETINFORMATION, &wdsi, &uiSize);
/* do error processing here */
```

During the registration of the write hook, the WinStation driver passes
entry points for the deprecated output buffer virtual driver helper
functions to the virtual driver in the VDWRITEHOOK structure. The
DriverOpen function saves these in global variables so helper functions
in the virtual driver can use them. The WinStation driver also passes a
pointer to the WinStation driver data area, which the DriverOpen
function also saves (because it is the first argument to the virtual
driver helper functions).

```
// Record pointers to functions used          
// for sending data to the host.              
pWd = vdwh.pWdData;                           
pOutBufReserve = vdwh.pOutBufReserveProc;     
pOutBufAppend = vdwh.pOutBufAppenProc;        
pOutBufWrite = vdwh.pOutBufWriteProc;         
pAppendVdHeader = vdwh.pAppendVdHeaderProc;   
```

&#52;.  Allocate all memory needed by the driver and do any initialization.
    You can obtain the maximum ICA buffer size from the MaximumWriteSize
    element in the VDWRITEHOOK structure that is returned.

!!!tip "Note"
		vdwh.MaximumWriteSize is one byte greater than the actual
maximum that you can use because it also includes the channel number.
```
g_usMaxDataSize = vdwh.MaxiumWriteSize - 1;
if(NULL == (pMyData = malloc( g_usMaxDataSize )))
{
	return(CLIENT_ERROR_NO_MEMORY);
}
```

&#53;.  Return the size of the VDOPEN structure in *puiSize*. This is used
    by the client engine to determine the version of the virtual
    channel driver.

## DriverPoll

Allows the virtual driver to get periodic control to perform any action
as required. With the Evt_* and Tmr_* APIs, a more event driven
implementation is possible so you may find that the DriverPoll is empty.

### Calling Convention

```
INT DriverPoll(
PVD pVD,
PVOID pVdPoll,
PUINT16 puiSize);
```

### Parameters

**pVD**

Pointer to a virtual driver control structure.

**pVdPoll**

Pointer to one of the driver poll information structures (DLLPOLL).

**puiSize**

Pointer to the size of the driver poll information structure. This is an
output parameter.

### Return Values

If the functionsucceeds, it returns CLIENT_STATUS_SUCCESS. If the driver has no data
on this polling pass, it returns CLIENT_STATUS_NO_DATA.

If the virtual driver cannot allocate an output buffer, it returns
CLIENT_STATUS_ERROR_RETRY so the WinStation driver does not slow
polling. The virtual driver then attempts to get an output buffer the
next time it is polled.

Return values that begin with CLIENT_ERROR_ are fatal errors; the ICA
session is disconnected.

### Remarks
Because the client engine is single threaded, a virtual driver is not allowed to block while waiting for a
desired result (such as the availability of an output buffer) because
this prevents the rest of the client from processing.

The Ping example includes examples of processing that can occur in
DriverPoll.


## DriverQueryInformation

Gets run-time information from the virtual driver.

### Calling Convention

```
INT DriverQueryInformation(
PVD pVD,
PVDQUERYINFORMATION pVdQueryInformation,
PUINT16 puiSize);
```
### Parameters

**pVD**

Pointer to a virtual driver control structure.

**pVdQueryInformation**

Pointer to a structure that specifies the information to query and the
results buffer.

**puiSize**

Pointer to the size of the query information and resolves structure.
This is an output parameter.

###Return Value

The function returns CLIENT_STATUS_SUCCESS.

### Remarks

This function currently has no practical significance for virtual
drivers; it is provided for compatibility with the loadable module
interface. There are no general purpose query functions at this time
other than LastError. The LastError query is accomplished through the
DriverGetLastError function.

## DriverSetInformation

Sets run-time information in the virtual driver.

### Calling Convention

```
INT DriverSetInformation(
PVD pVD,
PVDSETINFORMATION pVdSetInformation,
PUINT16 puiSize);
```
### Parameters

**pVD**

Pointer to a virtual driver control structure.

**pVdSetInformation**

Pointer to a structure that specifies the information class, a pointer
to any additional data, and the size in bytes of the additional data (if
any).

**puiSize**

Pointer to the size of the information structure. This is an input
parameter.

### Return Value

The function returns CLIENT_STATUS_SUCCESS.

### Remarks

This function can receive two information classes:

-   VdDisableModule: When the connection is being closed.

-   VdFlush: When WFPurgeInput or WFPurgeOutput is called by the
    server-side virtual channel application. The VdSetInformation
    structure contains a pointer to a VDFLUSH structure that specifies
    which purge function was called.

## Evt_create

Allocates an event structure containing a callback that can be
associated with the input or the output events of a particular file
descriptor.

### Calling Convention

```
VPSTATUS
Evt_create (
void \*hTC,
PFNDELIVER pDeliverFunc,
void \*pSubscriberId,
PEVT \*out);
```
### Parameters

**hTC**

Pass NULL value as a dummy.

**pDeliverFunc**

The callback to call.

**pSubscriberId**

Data passed as an argument to the callback.

**out**

The event structure returned.

### Return Value

The event structure created is returned with the out pointer argument.
If the function succeeds, the return value is EVT_SUCCESS.

If the function fails because of insufficient memory, the return value
is EVT_OBJ_CREATE_FAILED.

### Remarks

The first argument of the callback pSubscriberId is the same as the
pSubscriberId used to create the event structure.

The second argument nEvt is a pointer to the event structure responsible
for the callback.

## Evt_destroy

Destroys previously created event structure by freeing its memory and
nulling the given pointer.

### Calling Convention
```
VPSTATUS
Evt_destroy (
PEVT \*phEvt);
```
### Parameters

**phEvt**

Pointer to the event object to destroy.

### Return Value

If the function succeeds, the return value is EVT_SUCCESS.

### Remarks

The event object to destroy must be removed from the event loop using
Evt_remove_triggers, before Evt_destroy is called.


## Evt_remove_triggers

Removes the previously setup file descriptor selections from the given
file descriptor.

### Calling Convention

```
VPSTATUS
Evt_remove_triggers (
Int fd);
```
### Parameters

**fd**

The file descriptor to remove all selections from.

### Return Value

If the function succeeds, the return value is EVT_SUCCESS.

### Remarks

If both the input and output conditions are selected, both the
conditions are removed.

## Evt_signal

Calls the callback stored within the given event structure.

### Calling Convention

```
VPSTATUS
Evt_signal (
PEVT hEvt);
```
### Parameters

**hEvt**

The event structure containing the callback to call.

### Return Value

If the function succeeds, the return value is EVT_SUCCESS.

### Remarks

Calls the callback function directly. No conditions must be met prior to
this call.

## Evt_trigger_for_input

Connects the callback of an event structure to trigger on the given file
descriptor when it satisfies the input conditions.

### Calling Convention
```
VPSTATUS
Evt_trigger_for_input (
PEVT hEvt,
int fd);
```
### Parameters

**hEvt**

The event structure to associate with the input conditions of the given
file descriptor.

**fd**

The file descriptor.Workspace app

### Return Value

If the function succeeds, the return value is EVT_SUCCESS.

If the function fails because of insufficient memory, the return value
is EVT_OBJ_CREATE_FAILED.

### Remarks

The Glib implementation of the event loop used by Citrix Workspace app for Linux watches for the input conditions G_IO_IN and G_IO_HUP.

## Evt_trigger_for_output

Connects the callback of an event structure to trigger on the given file
descriptor when it satisfies the output conditions.

### Calling Convention
```
VPSTATUS
Evt_trigger_for_output (
PEVT hEvt,
int fd);
```
### Parameters

**hEvt**

The event structure to associate with the ouput conditions of the given
file descriptor.

**fd**

The file descriptor.Workspace app

### Return Value

If the function succeeds, the return value is EVT_SUCCESS.

If the function fails because of insufficient memory, the return value
is EVT_OBJ_CREATE_FAILED.

### Remarks

The Glib implementation of the event loop used by Citrix Workspace app for Linux
watches for the ouput conditions G_IO_OUT.

## ICADataArrival

The WinStation driver calls this function when data is received on a
virtual channel being monitored by the driver. The address of this
function is passed to the WinStation driver during DriverOpen.

### Calling Convention
```
INT wfcapi ICADataArrival(
PVD pVD,
USHORT uChan,
LPBYTE pBuf,
USHORT Length);
```
### Parameters

**pVD**

Pointer to a virtual driver control structure.

**uChan**

Virtual channel number.

**pBuf**

Pointer to the data buffer containing the virtual channel data as sent
by the server-side application.

**Length**

Length in bytes of the data in the buffer.

### Return Value

The driver returns CLIENT\_STATUS\_SUCCESS.

### Remarks

This function name is a placeholder for a user-defined function; the
actual function does not have to be called ICADataArrival, although it
does have to match the function signature (parameters and return type).
The address of this function is given to the WinStation driver during
DriverOpen. Although ICA prefixes packet control data to the virtual
channel data, this prefix is removed before this function is called.

After the virtual driver returns from this function, the WinStation
driver considers the data delivered. The virtual driver must save
whatever information it needs from this packet if later processing is
required.

Do not allow this function to block. Use your own thread or the
DriverPoll function (with polling enabled) for any required deferred
processing.

The virtual driver can send data to the server on receipt of this data
from within the ICADataArrival function, but be aware that the send
operation may return an immediate error when buffers are not available
to accommodate the send operation. The virtual driver may not block in
this function waiting for the sending operation to complete.

If the virtual driver is handling multiple virtual channels, use the
uChan parameter to determine the channel over which this data is to be
sent. See DriverOpen for more information.

## miGetPrivateProfileBool

Gets a Boolean value from a section of the Configuration Storage.

### Calling Convention

```
INT miGetPrivateProfileBool(
PCHAR lpszSection,
PCHAR lpszEntry,
BOOL bDefault);
```
### Parameters

**lpszSection**

Name of section to query.

**lpszEntry**

Name of entry to query.

**bDefault**

Default value to use.

### Return Values

If the requested entry is found, the entry value is returned; otherwise,
*bDefault* is returned.

### Remarks

A Boolean value of TRUE can be represented by on, yes, or true in the
configuration files. All other strings are interpreted as FALSE.

## miGetPrivateProfileInt

Gets an integer from a section of the Configuration Storage.

### Calling Convention

```
INT miGetPrivateProfileInt(
PCHAR lpszSection,
PCHAR lpszEntry,
INT iDefault);
```

### Parameters

**lpszSection**

Name of section to query.

**lpszEntry**

Name of entry to query.

**iDefault**

Default value to use.

### Return Values

If the requested entry is found, the entry value is returned; otherwise,
iDefault is returned.

## miGetPrivateProfileLong

Gets a long value from a section of the configuration files.

### Calling Convention

```
INT miGetPrivateProfileLong(
PCHAR lpszSection,
PCHAR lpszEntry,
LONG lDefault);
```

### Parameters

**lpszSection**

Name of section to query.

**lpszEntry**

Name of entry to query.

**lDefault**

Default value to use.

### Return Values

If the requested entry is found, the entry value is returned; otherwise,
lDefault is returned.

## miGetPrivateProfileString

Gets a string from a section of the configuration files.

### Calling Convention

```
INT miGetPrivateProfileString(
PCHAR lpszSection,
PCHAR lpszEntry,
PCHAR lpszDefault,
PCHAR lpszReturnBuffer, INT cbSize);
```

### Parameters

**lpszSection**

Name of section to query.

**lpszEntry**

Name of entry to query.

**lpszDefault**

Default value to use.

**lpszReturnBuffer**

Pointer to a buffer to hold results.

**cbSize**

Size of lpszReturnBuffer in bytes.

### Return Values

This function returns the string length of the value returned in
lpszReturnBuffer (not including the trailing NULL).

If the requested entry is found and the size of the entry string is less
than or equal to cbSize, the entry value is copied to lpszReturnBuffer;
otherwise, iDefault is copied to lpszReturnBuffer.

### Remarks

lpszDefault must fit in lpszReturnBuffer. The caller is responsible for
allocating and deallocating lpszReturnBuffer.

lpszReturnBuffer must be large enough to hold the maximum length entry
string, plus a NULL termination character. If an entry string does not
fit in lpszReturnBuffer, the lpszDefault value is used.

## MM_clip

Sets the shape of the operating system window “xwin” from the list of
sorted rectangles.

### Calling Convention

```
void
MM_clip (
UINT32 xwin,
int count,
struct tagTWI_RECT \*rects,
BOOLEAN extended)
```
### Parameters

**xwin**

Operating system session sub-window.

**count**

Number of rectangles.

**rects**

Array of rectangles sorted by Y and X.

**extended**

TRUE for any extensions; otherwise, FALSE.

### Return Values

There are no return values.

### Remarks

The structure has four long integers for left, top, right, and bottom.
Rectangles are YXsorted.

The last argument must be FALSE to start a fresh clipping update, and
TRUE to add any clipping updates to the current clipping list.


## MM_destroy_window

Destroys a window created by MM_get_window().

### Calling Convention

```
void
MM_destroy_window (
UINT32 hwin,
UINT32 xwin,
```

### Parameters

**hwin**

Host (seamless) window identifiers, ignored for non-seamless sessions.

**xwin**

x sub-window of the session window.

### Return Values

There are no return values.

### Remarks

MM_destroy_window also removes any window deletion callbacks added
with the low level MM_TWI_set_deletion_call.

## MM_get_window

Creates an operating system window "xwinp" that is a sub-window of an
existing session window with a server handle "hwin".

### Calling Convention

```
BOOLEAN
MM_get_window (
UINT32 hwin,
UINT32 \*xwinp,
```
### Parameters

**hwin**

Host (seamless) window identifiers, ignored for non-seamless sessions.

**xwinp**

Local operating system window identifier. Returns the sub-window
identifier of the session window. In this case, the X Window System is
the operating system windowing system.

### Return Values

If the parent (hwin) exists, the return value is TRUE. If the parent
does not exist, the return value is FALSE.

If the return value is FALSE, the function, including window creation,
still works. The root window, however, is used as a temporary parent.

A call to MM_get_window() or MM_set_geometry() can be used to
reparent to any existing seamless window.

### Remarks

When "0" is passed as the server handle in a non-seamless (single
window) session, there can be an existing window, \*xwinp that is
reparented. The sub-window, however, is unmapped.

If the parent is seamless, \*xwinp is protected by unmapping and
reparenting it to the root before the parent is deleted.

## MM_set_geometry

Sets the size and position for an existing sub-window, "xwin" of a
session window with the server handle, "hwin".

### Calling Convention

```
BOOLEAN
MM_set_geometry (
UINT32 hwin,
UINT32 xwin,
CTXMM_RECT \*rt);
```
### Parameters

**hwin**

Host (seamless) window identifiers, ignored for non-seamless sessions.

**xwin**

Local operating system window identifier for the session sub-window. In
this case, the X Window System is the operating system windowing system.

**rt**

CTXMM_RECT that describes the new window position and geometry.

### Return Values

If the parent (hwin) exists, the return value is TRUE. If the parent
does not exist, the return value is FALSE.

If the return value is TRUE, the sub-window is mapped on return.

### Remarks

The CTXMM_RECT window rectangle is within the session coordinates which
are not window relative and consist of four unsigned 32-bit integers for
left, top, right, and bottom.


## MM_show_window

Makes a sub-window visible.

### Calling Convention

```
void
MM_show_window (
UINT32 xwin)
```
### Parameters

**xwin**

Local operating system window identifier for the session sub-window. In
this case, the X Window System is the operating system windowing system.

### Return Values

There are no return values.

### Remarks

This function is called when the parent seamless window arrives after
the geometry is set.

There must, however, be a successful call to MM_get_window()
initially.

The function can be called with exactly the same window identifiers as
the previous one. It cannot be used if MM_set_geometry() previously
returned TRUE.

## MM_TWI_clear_new_window_function

Clears the callback function set up using
MM_TWI_set_new_window_function.

### Calling Convention

```
void
MM_TWI_clear_new_window_function (
void (\*) (UINT32))
```

### Parameters

(*)(UINT32))

Callback function pointer to remove.

### Return Values

There are no return values.

### Remarks

Clears the callback for seamless window creation.

## MM_TWI_set_new_window_function

Sets a callback function for seamless window creation.

### Calling Convention

```
void
MM_TWI_set_new_window_function (
void (*) (UINT32));
```

### Parameters

(*)(UINT32)

Callback function pointer to remove.

### Return Values

There are no return values.

### Remarks

When MM_get_window() fails because the seamless window is not yet
created, MM_TWI_set_new_window_function can be used to watch the
creation. The handle must be established only when required and should
be removed immediately. The callback argument is the server window
handle of a newly created seamless window.

## OutBufAppend (Deprecated)

!!!tip "Note"
		This function is deprecated. QueueVirtualWrite must be used in all
new virtual drivers.

Adds virtual channel packet data to the current output buffer.

### Calling Convention

```
INT WFCAPI OutBufAppend(
PWD pWd,
LPBYTE pData,
USHORT ByteCount);
```

### Parameters

**pWd**

Pointer to a WinStation driver control structure.

**pData**

Pointer to the buffer containing the data to append.

**ByteCount**

Number of bytes to append to the buffer.

### Return Values

If the function succeeds, it returns CLIENT_STATUS_SUCCESS.

If the function fails, it returns error code associated with the
failure; use GetLastError to get the extended error information.

### Remarks

This function adds virtual channel packet data to the end of the current
output buffer. A buffer of appropriate size must be reserved before
calling this function.

The address for this function is obtained from the VDWRITEHOOK structure
after hook registration. The VDWRITEHOOK structure also provides *pWd*.

This function can be called multiple times to build up the content of
the buffer. It is not written until OutBufWrite is called. Attempts to
write more data than was specified in OutBufReserve cause unpredictable
results.

The packet header information must be filled in before this function is
called.

If an ICA header or virtual channel data is appended to the buffer, the
buffer must be sent to the server before the control leaves the virtual
driver.

## OutBufReserve (Deprecated)

!!!tip "Note" 
		This function is deprecated. QueueVirtualWrite must be used in all
new virtual drivers.

Checks if a buffer of the requested size is available. This function
does not allocate buffers because they are already allocated by the
WinStation driver.

### Calling Convention

```
INT WFCAPI OutBufReserve(
PWD pWd,
USHORT ByteCount);
```

### Parameters

**pWd**

Pointer to a WinStation driver control structure.

**ByteCount**

Size in bytes of the buffer needed. This must be four bytes larger than
the data to be sent.

### Return Values

If a buffer of the specified size is available, the return value is
CLIENT_STATUS_SUCCESS.

If a buffer of the specified size is not available, the return value is
CLIENT_ERROR_NO_OUTBUF.

### Remarks

After this function is called to reserve an output buffer, use the other
OutBuf* helper functions to append data and then send the buffer to the
server.

If a buffer of the specified size is not available, attempt the
operation in a later DriverPoll call.

The developer determines the *ByteCount*, which can be any length up to
the maximum size supported by the ICA connection. This size is
independent of size restrictions on the lower-layer transport.

-   If the server is running XenApp or a version of Presentation Server
    3.0 Feature Release 2 or later, the maximum packet size is 5000
    bytes (4996 bytes of data plus the 4-byte packet overhead generated
    by the ICA datastream manager)

-   If the server is running a version of Presentation Server earlier
    than 3.0 Feature Release 2, the maximum packet size is 2048 bytes
    (2044 bytes of data plus the 4- byte packet overhead generated by
    the ICA datastream manager)

The address for this function is obtained from the VDWRITEHOOK structure
after hook registration. The VDWRITEHOOK structure also provides the
*pWd* address.

## OutBufWrite (Deprecated)

!!! tip "Note"
		This function is deprecated. QueueVirtualWrite must be used in all
new virtual drivers.

Sends a virtual channel packet to XenApp or XenDesktop.

### Calling Convention

```
INT WFCAPI OutBufWrite(
PWD pWd);
```

### Parameters

**pWd**

Pointer to a WinStation driver control structure.

### Return Values

If the function succeeds, it returns CLIENT_STATUS_SUCCESS.

If the function fails, it returns the error code associated with the
failure; use GetLastError to get the extended error information.

### Remarks

This function sends the current output buffer to the host. If a buffer
was not reserved or no data was appended, this function does nothing.

If an ICA header or virtual channel data is appended to the buffer, the
buffer must be sent to the server before DriverPoll returns.

The address for this function is obtained from the VDWRITEHOOK structure
after hook registration. The VDWRITEHOOK structure also provides the
*pWd* address.

## QueueVirtualWrite

QueueVirtualWrite is an improved scatter gather interface. It queues a
virtual write and stimulates packet output if required allowing data to
be sent without having to wait for the poll.

### Calling Convention

```
int WFCAPI
QueueVirtualWrite (
PWD pWd,
SHORT Channel,
LPMEMORY_SECTION pMemorySections,
USHORT NrOfMemorySections,
USHORT Flag);
```

### Parameters

**pWd**

Pointer to a WinStation driver control structure.

**Channel**

The virtual channel number

**pMemorySections**

Pointer to an array memory sections.

**NrOfMemorySections**

The number of memory sections.

**Flag**

This can be FLUSH_IMMEDIATELY if the data is required to be sent
immediately or ! FLUSH_IMMEDIATELY for lower priority data.

### Return Values

If the function succeeds, that is queued successfully, the return value
is CLIENT_STATUS_SUCCESS.

If the function fails because of unsuccessful queue, the return value is
CLIENT_ERROR_NO_OUTBUF.

### Remarks

The interface is simpler as it reduces the call sequence OutBufReserve,
AppendVdHeader, OutBufAppend, and OutBufWrite down to a single
QueveVirtualWrite call.

The data to be written across the chosen virtual channel is described by
an array of MEMORY_SECTION structures, each of which contains a length
and data pointer pair. This allows multiple non-contiguous data segments
to be combined and written with a single QueueVirtualWrite.

## Tmr_create

Creates a timer object and returns its handle.

### Calling Convention

```
VPSTATUS
Tmr_create (
HND hTC,
UINT32 uiPeriod,
PVOID pvSubscriber,
PFNDELIVER pfnDeliver.
PTMR * phTimer);
```
### Parameters

**hTC**

The value is NULL.

**uiPeriod**

The timeout for the timer in milliseconds.

**pvSubscriber**

Data passed as an argument to the callback.

**pfnDeliver**

The callback to call.

**phTimer**

The returned timer structure.

### Return Values

If the function succeeds, the return value is TMR_SUCCESS.

If the function fails because of insufficient memory, the return value
is TMR_OBJ_CREATE_FAILED.

### Remarks

The default state of a newly created timer object is disabled. The
"deliver" function is called when the timer fires.

## Tmr_destroy

Destroys the timer object pointed to by the given handle and sets the
handle to NULL.

### Calling Convention

```
VPSTATUS
Tmr_destroy (
PTMR \* phTimer);
```

### Parameters

**phTimer**

The timer to destroy.

### Return Values

If the function succeeds, the return value is TMR_SUCCESS.

### Remarks

Tmr_destroy is called for all timer objects when they are not required.

## Tmr_setEnabled

Enables or disables a timer object.

### Calling Convention

```
VPSTATUS
Tmr_setEnabled (
PTMR \* hTimer);
BOOL fEnabled);
```

### Parameters

**hTimer**

The timer to enable or disable.

**fEnabled**

Enables or disables the timer.

### Return Values

If the function succeeds, the return value is TMR_SUCCESS.

### Remarks

Enabling a disabled timer restarts the timing period. Re-enabling an
enabled timer, however, does not perform any action.

## Tmr_setPeriod

Sets the timeout period for a timer.

### Calling Convention

```
VPSTATUS
Tmr_setPeriod (
PTMR \* hTimer);
UNIT32 uiPeriod);
```

### Parameters

**hTimer**

The timer to change the timeout period for.

**uiPeriod**

The new timeout period in milliseconds.

### Return Values

If the function succeeds, the return value is TMR_SUCCESS.

### Remarks

If the timer is already running, the timer is reset and fires after the
new period. If the timer is disabled, the timeout period is updated but
the timer remains disabled.


## VdCallWd

Calls the client WinStation driver to query and set information about
the virtual channel. This is the main method for the virtual driver to
access the WinStation driver. For general-purpose virtual channel
drivers, this sets the virtual write hook.

### Calling Convention

```
INT VdCallWd (
PVD pVd,
USHORT ProcIndex,
PVOID pParam,
PUINT16 puiSize);
```
### Parameters

**pVd**

Pointer to a virtual driver control structure.

**ProcIndex**

Index of the WinStation driver routine to call. For virtual drivers,
this can be either WDxQUERYINFORMATION or WDxSETINFORMATION.

**pParam**

Pointer to a parameter structure, used for both input and output.

**puiSize**

Size of parameter structure, used for both input and output.

### Return Values

If the function succeeds, it returns CLIENT_STATUS_SUCCESS.

If the function fails, it returns an error code associated with the
failure; use DriverGetLastError to get the extended error information.

### Remarks

This function is a general purpose mechanism to call routines in the
WinStation driver. The only valid uses of this function for a virtual
driver are:

-   To allocate the virtual channel using WDxQUERYINFORMATION

-   To exchange function pointers with the WinStation driver during
    DriverOpen using WDxSETINFORMATION

For more information, see DriverOpen or the Ping example.

On successful return, the VDWRITEHOOK structure contains pointers to the
output buffer virtual driver helper functions, and a pointer to the
WinStation driver control block (which is needed for buffer calls).
