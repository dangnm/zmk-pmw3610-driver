PMW3610 driver implementation for ZMK with at least Zephyr 3.5

This work is based on [ufan's implementation](https://github.com/ufan/zmk/tree/support-trackpad) of the driver.

## Installation

Only GitHub actions builds are covered here. Local builds are different for each user, therefore it's not possible to cover all cases.

Include this project on your ZMK's west manifest in `config/west.yml`:

```yml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/petejohanson
    - name: inorichi
      url-base: https://github.com/inorichi
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: feat/pointers-move-scroll
      import: app/west.yml
    - name: zmk-pmw3610-driver
      remote: inorichi
      revision: main
  self:
    path: config
```

Then, edit your `build.yml` to look like this, 3.5 is now on main:

```yml
on: [workflow_dispatch]

jobs:
  build:
    uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@main
```

Now, update your `board.overlay` adding the necessary bits (update the pins for your board accordingly):

```dts
&pinctrl {
    spi0_default: spi0_default {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 0, 8)>,
                <NRF_PSEL(SPIM_MOSI, 0, 17)>,
                <NRF_PSEL(SPIM_MISO, 0, 17)>;
        };
    };

    spi0_sleep: spi0_sleep {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 0, 8)>,
                <NRF_PSEL(SPIM_MOSI, 0, 17)>,
                <NRF_PSEL(SPIM_MISO, 0, 17)>;
            low-power-enable;
        };
    };
};

&spi0 {
    status = "okay";
    compatible = "nordic,nrf-spim";
    pinctrl-0 = <&spi0_default>;
    pinctrl-1 = <&spi0_sleep>;
    pinctrl-names = "default", "sleep";
    cs-gpios = <&gpio0 20 GPIO_ACTIVE_LOW>;

    trackball: trackball@0 {
        status = "okay";
        compatible = "pixart,pmw3610";
        reg = <0>;
        spi-max-frequency = <2000000>;
        irq-gpios = <&gpio0 6 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;

        /*   optional features   */
        // snipe-layers = <1>;
        // scroll-layers = <2 3>;
        // automouse-layer = <4>;

        /*   optional: ball action on specific layers  */
        // arrows {
        //     layers = <3>;
        //     bindings =
        //         <&kp RIGHT_ARROW>,
        //         <&kp LEFT_ARROW>,
        //         <&kp UP_ARROW>,
        //         <&kp DOWN_ARROW>;
        //
        //     /*   optional: ball action configuration  */
        //     tick = <10>;
        //     // wait-ms = <5>;
        //     // tap-ms = <5>;
        // };
    };
};

/ {
  trackball_listener {
    compatible = "zmk,input-listener";
    device = <&trackball>;

  };
};
```

Now enable the driver config in your `board.config` file (read the Kconfig file to find out all possible options):

```conf
CONFIG_SPI=y
CONFIG_INPUT=y
CONFIG_ZMK_MOUSE=y
CONFIG_PMW3610=y
```

## Runtime Configuration

### Automouse Timeout

The automouse layer timeout can now be configured at runtime using the provided API functions. This allows you to dynamically adjust how long the mouse layer stays active after trackball movement.

**API Functions:**

```c
#include "pmw3610.h"

// Get the current timeout value
uint32_t timeout = pmw3610_get_automouse_timeout_ms(device);

// Set a new timeout value (in milliseconds, must be > 0)
int result = pmw3610_set_automouse_timeout_ms(device, 1000);  // 1 second
```

**Example Usage in Custom Behavior:**

You can create a custom ZMK behavior to adjust the timeout on the fly. The default timeout is set via `CONFIG_PMW3610_AUTOMOUSE_TIMEOUT_MS` in your config, but can be changed at runtime.

**Get Device Reference:**

To use these functions, you need to get a reference to the PMW3610 device. In most cases, you can use:

```c
const struct device *pmw_dev = DEVICE_DT_GET(DT_NODELABEL(trackball));
```

Where `trackball` matches the label in your device tree overlay.
