# PN532 NFC Library

This library allows for integration of NXP's [PN532/C106 NFC Module](http://www.nxp.com/documents/short_data_sheet/PN532_SDS.pdf) in an Electric Imp project.  This device supports a wide range of NFC technologies, including read, write, and emulation support for MIFARE, FeliCa, and ISO/IEC 14443A tags.  It also supports peer-to-peer communication and has power-saving options.

This library currently contains the base code needed to initialize the PN532, communicate with it, read from generic NFC tags, and enable power-saving features.

For a library exposing the PN532's MIFARE Classic support, see the [PN532MifareClassic library](https://github.com/electricimp/PN532MifareClassic).

### Examples

For an example extension class supporting PN532 card emulation mode, see the [`PN532CardEmulator`](examples/cardEmulator) class.  It serves as an example of how to build on the `PN532` class to interface with the many other protocols and features that the PN532 supports.

The [PN532MifareClassic library](https://github.com/electricimp/PN532MifareClassic) also serves as an example of how to build on the `PN532` class.

## Constructor: PN532(*spi, ncs, rstpd_l, irq, callback*)

Creates and initializes an object representing the PN532 NFC device.

### Parameters

- *spi*: A [SPI object](https://developer.electricimp.com/api/hardware/spi) pre-configured with the flags `LSB_FIRST | CLOCK_IDLE_HIGH` and a clock rate.  The PN532 supports clock rates up to 5 MHz.
- *ncs*: A pin connected to the PN532's not-chip-select line.
- *rstpd_l*: A pin connected to the PN532's RSTPDN (Reset/Power-Down) pin.  This can be null if this the RSTPDN pin will not be under software control.
- *irq*: A pin connected to the PN532's P70_IRQ pin.
- *callback*: A function that will be called when object instantiation is complete.  It takes one *error* parameter that is null upon successful instantiation.

### Usage

```squirrel
#require "PN532.class.nut:1.0.1"

local spi = hardware.spi257;
local ncs = hardware.pin1;
local rstpd_l = hardware.pin3;
local irq = hardware.pin4;

spi.configure(LSB_FIRST | CLOCK_IDLE_HIGH, 2000);

function constructorCallback(error) {
    if(error != null) {
        server.log("Error constructing PN532: " + error);
        return;
    }
    // It's now safe to use the PN532
}

reader <- PN532(spi, ncs, rstpd_l, irq, constructorCallback);
```

## init(*callback*)

Configures the PN532 with settings for later use.  The *callback* is called upon completion and takes one *error* parameter that is null on success.

This method must be called whenever the PN532 is power-cycled, but is automatically called by the constructor.

### Usage

```squirrel
function initCallback(error) {
    if(error != null) {
        server.log("Error re-initializing PN532: " + error);
        return;
    }
    // It's now safe to use the PN532
}

// Power cycle the device
rstpd_l.write(0);
rstpd_l.write(1);

// Reinitialize
reader.init(initCallback);
```

## setHardPowerDown(*poweredDown, callback*)

Enables or disables power to the PN532 using the *rstpdn_l* pin passed to the constructor.

This produces a greater power savings than using [enablePowerSaveMode(*shouldEnable [, callback]*)](#enablepowersavemodeshouldenable--callback), but wipes all device state and must be explicitly turned off before the PN532 can be used again.  It will automatically call [init(*callback*)](#initcallback) upon power-up.

The *callback* is called upon completion and takes one *error* parameter that is null on success.

If the *rstpdn_l* pin was not passed to the constructor, the callback will return a `PN532.ERROR_NO_RSTPDN` error.

### Usage

```squirrel
reader.setHardPowerDown(true, function(error) {
    if(error != null) {
        server.log(error);
        return;
    }

    // Spend a long time doing things that don't require the PN532...

    reader.setHardPowerDown(false, function(error) {
        if(error != null) {
            server.log(error);
            return;
        }

        // Continue using the PN532 from here
    });

});
```

## enablePowerSaveMode(*shouldEnable [, callback]*)

Enables or disables a power-saving mode on the PN532.  This mode will apply to all future commands on the PN532 until it is disabled with another call to this method.  Use of power-save mode will add a 1 ms latency to all commands sent, but significantly decreases power consumption.

Note that even if this is set, power save mode will not be entered after certain commands (such as [pollNearbyTags(*tagType, pollAttempts, pollPeriod, callback*)](#pollnearbytagstagtype-pollattempts-pollperiod-callback)) that require state to be stored on the PN532.  This is automatically handled by the class and power save mode will be re-entered when a compatible command is run.

The *callback* must take the following parameters:

- *error*: A string that is null on success.
- *wasEnabled*: A boolean that matches *shouldEnable*.

### Usage

```squirrel
function powerSaveCallback(error, succeeded) {
        // Continue using PN532 with lower power usage
}

reader.enablePowerSaveMode(true, powerSaveCallback);
```

## getFirmwareVersion(*callback*)

Queries the PN532 for its internal firmware version number.

Takes a *callback* with two parameters:

- *error*: A string that is null on success.
- *version*: A table that contains fields for *IC*, *Ver*, *Rev*, and *Support* as documented in the `GetFirmwareVersion` command from the [PN532 datasheet](http://www.nxp.com/documents/user_manual/141520.pdf).

### Usage

```squirrel
function firmwareVersionCallback(error, version) {
    if(error != null) {
        server.log(error);
        return;
    }

    server.log(format("Firmware version: %X:%X.%X-%X", version.ic, version.ver, version.rev, version.support));
}

reader.getFirmwareVersion(firmwareVersionCallback);
```

## pollNearbyTags(*tagType, pollAttempts, pollPeriod, callback*)

Repeatedly searches for nearby NFC tags of type *tagType* and passes scan results to the *callback* upon completion.

### Parameters

- *tagType*: An integer representing the baud rate and initialization protocol used during the scan.  It must be taken from the following static class members:

 - `PN532.TAG_TYPE_106_A`: 106 kbps ISO/IEC14443 Type A
 - `PN532.TAG_TYPE_212`: Generic 212 kbps
 - `PN532.TAG_TYPE_424`: Generic 424 kbps
 - `PN532.TAG_TYPE_106_B`: 106 kbps ISO/IEC14443-3B
 - `PN532.TAG_TYPE_106_JEWEL`: 106 kbps Innovision Jewel

    Any of the above members can also be combined with the flag `PN532.TAG_FLAG_MIFARE_FELICA` where appropriate to specify that only cards with MIFARE or FeliCa support should be polled.
- *pollAttempts*: An integer representing how many times the PN532 should search for the specified card type. Any value between 1 and 254 will initiate the corresponding number of polls.  The static class member `PN532.POLL_INDEFINITELY` will cause this command to poll forever until a card is found.
- *pollPeriod*: An number controlling the time in between poll attempts.  It should be given in seconds, but will round up to the nearest multiple of of 150 ms.
- *callback*: A function with the following parameters:
 - *error*: A string that is null on success.
 - *numTagsFounds*: The number of tags found during the scan (currently either 1 or 0)
 - *tagData*: A table that is non-null only when *numTagsFound* is 1 and contains the following fields:
     - *type*: The type of tag found.  This corresponds to the tag type specified above except when the `TAG_FLAG_MIFARE_FELICA` flag has not been used, when it will be added if MIFARE or FeliCa support has been detected.
     - *SENS_RES*: A 2-byte blob representing the SENS_RES/ATQA field of the tag
     - *SEL_RES*: 1 byte representing the SEL_RES/SAK field of the tag
     - *NFCID*: A blob representing the semi-unique UID of the tag (usually either 4 or 7 bytes)
     - *ATS*: The ATS response, if generated by the tag

### Usage

```squirrel
function scanCallback(error, numTagsFound, tagData) {
    if(error != null) {
        server.log(error);
        return;
    }

    if(numTagsFound > 0) {
        server.log("Found a tag:");
        server.log(format("SENS_RES: %X %X", tagData.SENS_RES[0], tagData.SENS_RES[1]));
        server.log("NFCID:");
        server.log(tagData.NFCID);
    } else {
        server.log("No tags found");
    }
}

// Poll 10 times around once a second
reader.pollNearbyTags(PN532.TAG_TYPE_106_A | PN532.TAG_FLAG_MIFARE_FELICA, 0x0A, 6, scanCallback);
```

**The following methods are exposed for use in creating extensions of the PN532 class to support extra protocols and commands.**

## PN532.makeDataExchangeFrame(*tagNumber, data*)

Constructs a data exchange frame for use in [PN532.sendRequest(*requestFrame, responseCallback, shouldRespectPowerSave [, numRetries]*)](#sendrequestrequestframe-responsecallback-shouldrespectpowersave--numretries).

### Parameters

- *tagNumber*: The index number of the tag in the current field.  Currently this is always 1.
- *data*: The payload blob for this frame.

See the [PN532 datasheet](http://www.nxp.com/documents/user_manual/141520.pdf) for detailed use of the data exchange frame.

### Usage

```squirrel
function makeMifareReadFrame(address) {
    local frame = blob(2);

    frame.writen(0x30, 'b');
    frame.writen(address, 'b');

    return PN532.makeDataExchangeFrame(1, frame);
}
```

## PN532.makeCommandFrame(*command [, data]*)

Constructs a command frame for use in [PN532.sendRequest(*requestFrame, responseCallback, shouldRespectPowerSave [, numRetries]*)](#sendrequestrequestframe-responsecallback-shouldrespectpowersave--numretries).

### Parameters

- *command*: The integer corresponding to a PN532 command.
- *data*: An optional blob containing the payload/arguments for the command.

See the [PN532 datasheet](http://www.nxp.com/documents/user_manual/141520.pdf) for detailed use of the command frame.

### Usage

```squirrel
function getFirmwareVersion(callback) {
    local frame = PN532.makeCommandFrame(0x02);

    PN532.sendRequest(frame, callback, true);
}
```

## sendRequest(*requestFrame, responseCallback, shouldRespectPowerSave [, numRetries]*)

Sends the specified *requestFrame* to the PN532 and associates a *responseCallback* to handle the response.  Optionally allows for a maximum number of retries due to transmission failures.

### Errors

- If a previously sent request is still waiting for a response, the callback will return a `PN532.ERROR_CONCURRENT_COMMANDS` error.
- If the request fails to transmit more than *numRetries* times, the callback will return a `PN532.ERROR_TRANSMIT_FAILURE` error.
- If the request contains an application-level error (e.g. incorrect arguments to a command), the callback will return a `PN532.ERROR_DEVICE_APPLICATION` error.

### Parameters

- *requestFrame*: A frame generated by [PN532.makeDataExchangeFrame(*tagNumber, data*)](#pn532makedataexchangeframetagnumber-data) or [PN532.makeCommandFrame(*command [, data]*)](#pn532makecommandframecommand--data).
- *responseCallback*: A function that takes the following arguments:
    - *error*: A string that is null on success.
    - *responseData*: A blob containing the raw response from the PN532 to the request.
- *shouldRespectPowerSave*: A boolean representing whether this request should respect the power save mode state set in [enablePowerSaveMode(*shouldEnable [, callback]*)](#enablepowersavemodeshouldenable--callback). If false, this request will not initiate a power-down after the request. Most calls should try to respect the state, but calls that require state to be stored in the PN532 between commands cannot use it.
- *numRetries*: An optional integer defaulting to 3.

### Usage

```squirrel
function responseCallback(error, responseData) {
    if(error != null) {
        imp.wakeup(0, function() {
            userCallback(error, null);
        });
        return;
    }

    // Process responseData...
    server.log(responseData.tostring());
}

local frame = makeMifareReadFrame(0x3);
reader.sendRequest(frame, responseCallback, false);
```

# License

The PN532 library is licensed under the [MIT License](./LICENSE).
