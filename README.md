# esphome-display-panel
`DisplayPanel` class for [ESPHome](https://esphome.io/) [display:](https://esphome.io/components/display/index.html) objects.

DisplayPanel provides a rectangular panel to be displayed on the LCD/OLED display that we can  write one or more lines of centered text to. One can also use DisplayPanel to easily detect if a panel is within range of a touch event.

# Sample uses

Sample ESPHome projects that use the DisplayPanel class:

Clock that uses the ill9341 TFT display + touchscreen, connected to Home Assistant to display time, temperature, etc.
Supports touch to adjust brightness.
* Model and pics https://www.thingiverse.com/thing:5586412
* Code https://github.com/kdorff/esphome/tree/main/guest-time-temp

Clock that uses an 2.42" SSD1306/SSD1309 OLED, connected to Home Assistant to display time, temperature, etc.
* Model and pics https://www.thingiverse.com/thing:5607245
* Code https://github.com/kdorff/esphome/tree/main/tft-office

# How to use esphome-dislay-panel

I will highlight the important features of the [tft-office](https://github.com/kdorff/esphome/tree/main/tft-office)

## Getting started

ESPHome projects all start with a `.yaml` file. For this project, it is `tft-office.yaml`.

Unlike other ESPHome projects you might be used, we are going to put the bulk of the project's data initialization and per-loop update code in a `.h` file that lives in the  `esphome` folder next to your `.yaml` file. For this project, it is `tft-room-time-temp.h`. 

> **Note:** I named this differently from the ESPHome project name because it is used by multiple ESPHome projects.

Looking first at the `.yaml` file, we find

```yaml
esphome:
  name: tft-office
  includes:
    - tft-room-time-temp.h
  libraries:
    esphome-display-panel=https://github.com/kdorff/esphome-display-panel.git#v0.0.1
```

The `includes:` stanza is including the `tft-room-time-temp.h` file where we will be doing data initialization and per-update DisplayPanel state updates.

The `libraries:` stanza is include this `esphome-display-panel project`, version `v0.0.1`.

## The .h file, adding the esphome-display-panel library

The first thing you'll find in the `tft-room-time-temp.h` file is 

```C++
#include <display-panel.h>
```

This enables the use of the DisplayPanel C++ class.

## The display: lambda: loop in the .yaml file

The way one updates the `display:` component in ESPHome is by providing code in its `lambda` stanza. The actal initialization and update code will be found in the `.h` file. To call this code, we define the `display:`'s `lambda:` as such

```yaml
display:
  - ...
    # How often to update the displsay
    update_interval: 1s
    # Code to update the display
    lambda: |-
      static bool panelsInitialized = 0;
      if (!panelsInitialized) {
        initializePanels(it);
        panelsInitialized = 1;
      }
      updatePanelStates(it);
      drawPanels(it);
```

We can see from this, we must provide three methods in our `.h` file: `initializePanels(...)`, `updatePanelStates(...)`, and `drawPanels(...)`.

## Creating the DisplayPanel objects

Before looking at the method `initializePanels(...)`, we should look in `tft-room-time-temp.h` just above the `initializePanels(...)` method.

First we find `char buffer[25];`. This is a temporary buffer so we repeatedly can use with `sprintf` without continually needing new buffers to print to.

Next, we have a number of `#define`s. These are all related to configuring the sizes of the DisplayPanels that we will be creating.

The `PW` and `PH` macros convert a percentage of **Width** or **Height** to pixels for the current display, based on values in `WIDTH` and `HEIGHT`.

```C++
...
// Size of the actual display
#define WIDTH 320
#define HEIGHT 240
// Convert percentage width or height (0-100) to pixels
#define PW(PCT_WIDTH) (PCT_WIDTH * 0.01 * WIDTH)
#define PH(PCT_HEIGHT) (PCT_HEIGHT * 0.01 * HEIGHT)
...
```

The following `*_WIDTH` and `*_HEIGHT` `#define`s specify the **percentage** width and heights for the various DisplayPanels that will be created.

```C++
...
#define DAY_WIDTH PW(70)
#define DAY_HEIGHT PH(13)
#define DATE_WIDTH PW(70)
#define DATE_HEIGHT PH(12)
...
```

Finally, we define a DisplayPanel for each display element we want in the UI. 

```C++
// The constructor arguments are X, Y, Width, Height
DisplayPanel datePanel(CONT_WIDTH, 0, DATE_WIDTH, DATE_HEIGHT);
DisplayPanel dayPanel(CONT_WIDTH, DATE_HEIGHT, DAY_WIDTH, DAY_HEIGHT);
```

## initializePanels(...), only at startup

The method `initializePanels(...)` should only be called once. It should define the initial font, color, text color, etc. for each of the DisplayPanels. Anything that doesn't change from loop-to-loop can be set here. 

## updatePanelStates(...), once per display update

Every time the display is updated, the method `updatePanelStates(...)` should be used to define any changes to the states of any DisplayPanels. If your program is a clock, it should update one of the DisplayPanels with the time.

At this time, the **one or more** strings of `.text` will always be printed in the **center** of the DisplayPanel.

## drawPanels(...), once per display update

This is where the panel is actually drawn to the display.

This should call `.draw(...)` for each DisplayPanel, and/or use the more optimized `DisplayPanel::drawAllPanels(...)` to more than one DisplayPanel in a single call.

## Detecting touch, in the .yaml file

If the display in question also supports *touch*, calling `.isTouchOnPanel(x, y);` for any of the DisplayPanels will return `true` of the touch was on that panel.

In `tft-office.yaml`s `touch:` `on_touch:` stanza, we see where it checks for touch on the `contUpPanel` and `contDownPanel` DisplayPanels, adjust `brightness` accordingly, and sets the TFTs `backlight` to the value stored of `brightness`.

```yaml
touchscreen:
  on_touch:
    then:
      - if:
          condition:
            lambda: |-
              return contUpPanel.isTouchOnPanel((id(touch)).x, (id(touch)).y);
          then:
            - lambda: |-
                // Increase brightness 1%
                id(brightness) = id(brightness) + 0.01 > 1 ? 1.0 : id(brightness) + 0.01;
                id(backlight).set_level(id(brightness));
                // ...
        # ...
```
