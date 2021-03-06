SPI API
=======

The SPI API supports the Serial Peripheral Interface, a synchronous serial protocol that allows multiple slave chips to communicate with a master chip. A single SPI bus uses the following pins: SCLK for clock, MOSI (Master Out, Slave In) for write, MISO (Master In, Slave Out) for read, and one or more SS (Slave Select) for selecting the slave device.

For each clock signal one bit is written from the master to the selected slave and one bit is read by the master from the selected slave, so there is no separate read and write, but one transceive operation.

When a slave device's chip select is 0 (low), then it communicates with the master, otherwise it ignores the master. The master can select multiple slaves in a write-only configuration; in this case no slave is writing data, they only read.

Since the SS pins may be connected to slave chip select through a demultiplexer and thereby work as an address bus, slave devices are identified by an index in this API, rather than by SS pins. Also, since multiple SPI buses may be present on a board, these are identified by an index in this API. Implementations SHOULD encapsulate the mapping from SPI bus number and device number to the physical SPI pins.

This API uses a [`Buffer`](../README.md/#buffer) object for both read and write data.

<a name="apiobject"></a>
### The SPI API object
When requiring `"spi"`, the following steps are run:
- If there is no permission for using the functionality, throw `SecurityError`.
- If the SPI functionality is not supported on the board, throw `"NotSupportedError"`.
- Return an object that implements the following method.

| Method              | Description      |
| ---                 | ---              |
| [`open()`](#open)   | synchronous open |

See also the [Web IDL](./webidl.md) definition.

<a name="open"></a>
#### The `SPI open(options)` method
Configures an SPI bus using data provided by the `options` argument. It runs the following steps:
- Let `spi` be an [`SPI`](#spi) object.
- If `options` is a dictionary and the `options.bus` property is a number between 0 and 127, let `spi.bus` be `options.bus`, otherwise select the platform default value, and if that is not available, set the value to 0.
- If `options.speed` is not a number, let `spi.speed` be 10. Otherwise, set `spi.speed` to the closest matching value that is lower than `options.speed` and is supported by the platform.
- If `options.msbFirst` is `false`, set `spi.msbFirst` to `false`, otherwise set it to `true`.
- If `options.bits` is in the set {1, 2, 4, 8, 16 }, then set `spi.bits` to `option.bits`, otherwise set it to the value 4.
- If `options.polarity` is 0, then set `spi.polarity` to 0, otherwise set `spi.polarity` to 2.
- If `options.phase` is 0, then set `spi.phase` to 0, otherwise set `spi.phase` to 1.
- Request the underlying platform to initialize the SPI `spi.bus` with `spi.speed` on the board. The implementation will use the board mapping from the value of `bus` to the set of physical pins used for the bus.
- In case of failure in any of the steps above, return `null`.
- Set `spi.frameGap` to 0. Request the underlying platform to provide the SPI inter-frame delay value expressed in nanoseconds and if the request successfully completes, then set `spi.frameGap` to that value.
- Set `spi.topology` to `"full-duplex"`. Request the underlying platform to provide the SPI transfer mode and if the request successfully completes, then set `spi.topology` to the corresponding value.
- Return `spi`.

<a name="spi"></a>
### The `SPI` interface
Represents the properties and methods that expose SPI functionality. The `SPI` object has the following read-only properties:

| Property   | Type   | Optional | Default value | Represents |
| ---        | ---    | ---      | ---           | ---        |
| `bus`      | octet  | no       | 0             | SPI bus |
| `speed`    | long   | yes      | `undefined`   | SPI bus speed in MHz |
| `msbFirst` | boolean | yes     | `true`        | MSB (or LSB) first |
| `bits`     | number | yes      | 8             | number of data bits |
| `polarity` | long   | yes      | 0             | clock polarity, 0 or 2 |
| `phase`    | long   | yes      | 0             | clock phase, 0 or 1 |
| `frameGap` | unsigned long | yes | `undefined` | inter-frame gap in nanoseconds |
| `topology` | string | yes      | `undefined` | SPI master-slave transfer topology |

| Method signature  | Description            |
| ---               | ---                    |
| [`transceive(device, buffer)`](#transceive) | read or write device(s) |
| [`close()`](#close) | close the SPI bus |

The `bus` property denotes the SPI bus number between 0 and 127.

The `speed` property is a floating point number that denotes the SPI bus speed in MHz. Usually it is between 10 and 66 MHz.

The `msbFirst` property is a boolean denoting whether the most significant bit (MSB) is sent first, or the least significant bit (LSB). The default value is `true` (MSB first).

The `bits` property is a number denoting the number of data bits (word size). Usually it is 1, 2, 4, 8 or 16. The default value is 4.

The `polarity` property is a number denoting the clock polarity (CPOL). The value 0 means clock is active on high (rising edge), the value 2 means clock is active on low (falling edge).

The `phase` property is a number denoting the clock phase (CPHA), i.e. when the data is sampled on the MOSI (Master Out Slave In) and MISO (Master In Slave Out) pins of the SPI bus. When `phase` is 0, MISO and MOSI data is sampled on the leading clock edge, whether it is rising (polarity 0) or falling (polarity 2). When `phase` is 1, MISO and MOSI data is sampled on the trailing clock edge (whether it is rising or falling).

Often, SPI mode is being referred to as the combination of clock polarity and phase values:
- mode 0: polarity normal (value: 0), phase 0, data is sampled on leading (and rising) clock
- mode 1: polarity normal (value: 0), phase 1, data is sampled on trailing (and rising) clock
- mode 2: polarity inverse (value: 2), phase 0, data is sampled on leading (and falling) clock
- mode 3: polarity inverse (value: 2), phase 1, data is sampled on trailing (and falling) clock.

The sum of the `polarity` and `phase` property values provides the SPI mode (this is the reason `polarity` takes the values 0 or 2).

The `frameGap` property denotes the inter-frame gap in milliseconds on the platforms this is supported. The default value is 0.

The `topology` property describes the SPI master-slave connection type. This value may be provided by implementations that support the feature, but it is not mandatory. Applications can check whether the property is defined. The values can be the following:
- `"full-duplex"`: the slave devices are connected to the master via separate SS (Slave Select) lines that each activate one slave device. Prior to communication, the master must activate one device by driving the corresponding SS to 0, then data transfer is bidirectional. This is the default value.
- `"single-read"`: the slaves are connected to the master as above. When a slave is activated, the data is read by the master.
- `"single-write"`: the slaves are connected to the master as above. The master can activate any number of SS lines and the data is read by all activated slaves.
- `"multiplexed"`: 4 SS lines are connected to a decoder that can activate up to 15 slave devices. This works as full-duplex.
- `"daisy-chain"`: the master uses one SS and one SCLK (clock) line for all slaves. The MOSI line from the master goes to the first slave's MOSI pin, the MISO line of that slave goes to the MOSI pin of the next slave, and so forth. The last slave's MISO line is connected to the master's MISO pin.

<a name="transceive"></a>
#### The `transceive(device, buffer)` method
Writes a [`Buffer`](../README.md/#buffer) `buffer` using SPI to the slave identified by the `device` argument, and reads another [`Buffer`](../README.md/#buffer) from the slave device that is returned. The method runs the following steps:
- If `device` is not a number between 0 and 127, Throw `TypeError` and terminate these steps.
- Request the underlying platform to write the bytes specified in `buffer` to the specified device and read another [`Buffer`](../README.md/#buffer) `readBuffer`. The implementation maps the value of `device` to the physical SS (slave select) pins on the board. If the operation fails, throw `SystemError` and abort these steps.
- Return `readBuffer`.

<a name="close"></a>
#### The `close()` method
Closes the current [`SPI`](#spi) bus and cancels all pending operations.

### Examples

```javascript
try {
  var spi = require("spi").open();  // open the default SPI bus

  console.log("SPI bus " + spi.bus + " opened with bus speed " + spi.speed);
  console.log("SPI mode: " + spi.mode);
  console.log("Data bits: " + spi.bits);
  console.log("Speed [MHz]: " + spi.speed);
  console.log("MSB first: " + (spi.msbFirst ?  "true" : "false"));

  var buffer = spi.transceive(0, [1, 2, 3]);
  console.log("From SPI device 0: " + buffer.toString());

  spi.close();
} catch (err) {
  console.log("SPI error: " + err.message);
}
```

