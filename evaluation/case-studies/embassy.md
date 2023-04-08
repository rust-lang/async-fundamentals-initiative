# Use of AFIT in Embaassy

*The following are rough notes on the usage of Async Function in Traits from the [Embassy][] runtime. They are derived from a conversation between dirbaio and nikomatsakis.*

[Embassy]: https://github.com/embassy-rs/embassy

Links to uses of async functions in traits within Embassy:

* most popular ones are [embedded-hal-async](https://github.com/rust-embedded/embedded-hal/tree/master/embedded-hal-async/src)
* HAL crates provide impls for particular microcontrollers (e.g., [gpiote](https://github.com/embassy-rs/embassy/blob/master/embassy-nrf/src/gpiote.rs#L518), [spim](https://github.com/embassy-rs/embassy/blob/master/embassy-nrf/src/spim.rs#L523), [i2c](https://github.com/embassy-rs/embassy/blob/master/embassy-stm32/src/i2c/v2.rs#L1061))
* driver crates use the traits to [implement a driver for some chip that works on top of any HAL:
    * [nrf70](https://github.com/embassy-rs/nrf70/blob/main/src/main.rs#L811) (that one is interesting because it defines another async Bus trait on top, because that chip can be used with either SPI or QSPI)
    * [es-wifi-driver](https://github.com/drogue-iot/es-wifi-driver/blob/main/src/lib.rs#L132)
    * [hts221](https://github.com/drogue-iot/hts221-async/blob/main/src/lib.rs#L40)
    * [sx127x](https://github.com/embassy-rs/embassy/blob/master/embassy-lora/src/sx127x/sx127x_lora/mod.rs#L50)
* there's also embedded-io, which is [std::io traits adapted for embedded](https://github.com/embassy-rs/embedded-io/blob/master/src/asynch.rs)
* HALs have [impls for serial ports](https://github.com/embassy-rs/embassy/blob/master/embassy-nrf/src/buffered_uarte.rs#L600)
* embassy-net has an [impl for TCP sockets](https://github.com/embassy-rs/embassy/blob/master/embassy-net/src/tcp.rs#L431)
* here's some [driver using it](https://github.com/drogue-iot/esp8266-at-driver/blob/main/src/lib.rs#L74)
* embassy-usb has a [Driver trait](https://github.com/embassy-rs/embassy/blob/master/embassy-usb-driver/src/lib.rs); that one is probably the most complex, it's an async trait with associated types with more async traits, and interesting lifetimes
* HALs [impl these traits for one particular chip](https://github.com/embassy-rs/embassy/blob/master/embassy-nrf/src/usb/mod.rs#L178) and embassy-usb uses them to [implement a chip-independent USB stack](https://github.com/embassy-rs/embassy/blob/master/embassy-usb/src/lib.rs#L188)


most of these are "abstract over hardware", and when you build a firmware for some product/board you know which actual hardware you have, so you use static generics, no need for dyn

the few instances I've wished for dyn is:
* with embedded-io it does sometimes happen. For example, running the same terminal ui over a physical serial port and over telnet at the same time. Without dyn that code gets monomorphized two times, which is somewhat wasteful.
* this trait https://github.com/embassy-rs/embassy/blob/master/embassy-usb/src/lib.rs#L89 . That one MUST use dyn because you want to register multiple handlers that might be different types. Sometimes it'd have been handy to be able to do async things within these callbacks. Workaround is to fire off a notification to some other async task, it's not been that bad.
    * niko: how is this used?
    * handlers are added [here](https://github.com/embassy-rs/embassy/blob/master/embassy-usb/src/builder.rs#L262), passed into UsbDevice [here](https://github.com/embassy-rs/embassy/blob/master/embassy-usb/src/lib.rs#L225), and then called when handling some bus-related stuff, for example [here](https://github.com/embassy-rs/embassy/blob/master/embassy-usb/src/lib.rs#L714-L715).
    * the tldr of what it's used for is you might have a "composite" usb device, which can have multiple "classes" at the same time (say, an Ethernet adapter and a serial port). Each class gets its own "endpoints" for data, so each launches its own independent async tasks reading/writing to these endpoints. 
    * But there's also a "control pipe" endpoint that carries "control" requests that can be for any class for example for ethernet there's control requests for "bring the ethernet interface up/down", so each class registers a handler with callbacks to handle their own control requests, there's a "control pipe" task that dispatches them.
    * Sometimes when handling them, you want to do async stuff. For example for "bring the ethernet interface up" you might want to do some async SPI transfer to the ethernet chip, but currently you can't.
    * niko: would all methods be async if you could?
    * not sure if all methods, but probably `control_in`/`control_out` yes. and about where to store the future for the dyn... not sure. That crate is no-alloc so Box is out it'd probably be inline in the stack, like with StackFuture. Would need configuring the max size, probably some compile-time setting, or a const-generic in UsbDevice.
