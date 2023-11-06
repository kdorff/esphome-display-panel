# esphome-display-panel
`DisplayPanel` class for [ESPHome](https://esphome.io/) [display:](https://esphome.io/components/display/index.html) objects. 

DisplayPanel provides a rectangular panel to be displayed on the LCD/OLED display that we can  write one or more lines of centered text to. One can also use DisplayPanel to easily detect if a panel is within range of a touch event.

# Sample uses

Sample ESPHome projects that use the DisplayPanel class:

**Clock that uses the ill9341 TFT display + touchscreen, connected to Home Assistant to display time, temperature, etc.**
Supports touch to adjust brightness.
* Model and pics https://www.thingiverse.com/thing:5586412
* Code https://github.com/kdorff/esphome/tree/main/guest-time-temp

**Clock that uses an 2.42" SSD1306/SSD1309 OLED, connected to Home Assistant to display time, temperature, etc.**
* Model and pics https://www.thingiverse.com/thing:5607245
* Code https://github.com/kdorff/esphome/tree/main/tft-office

# How to use esphome-display-panel

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
    esphome-display-panel=https://github.com/kdorff/esphome-display-panel.git#v0.0.14
```

The `includes:` stanza is including the `tft-room-time-temp.h` file where we will be doing data initialization, state updates, drawing, etc.

The `libraries:` stanza is include this `esphome-display-panel project`, version `v0.0.14`.

## The .h file, adding the esphome-display-panel library

The first thing you'll find in the `tft-room-time-temp.h` file is 

```C++
#include <display-panel.h>
```

This enables the use of the DisplayPanel C++ class.

## The display: lambda: loop in the .yaml file

The way one updates a `display:` component in ESPHome is by providing code in its `lambda:` stanza. The actual **initialization**, **state update**, and **drawing** code will be found in the `.h` file, but will be called by the code we define the `display:`'s `lambda:` such as

```yaml
display:
  - ...
    # How often to update the display
    update_interval: 1s
    # Code to update the display
    lambda: |-
      static bool panelsInitialized = 0;
      if (!panelsInitialized) {
        initializePanels(it);
        panelsInitialized = 1;
      }
      updatePanelStates();
      drawPanels();
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
#define CONT_WIDTH PW(15)
...
#define DAY_WIDTH PW(70)
#define DAY_HEIGHT PH(13)
#define DATE_WIDTH PW(70)
#define DATE_HEIGHT PH(12)
...
```

Define a few important variables and create a DisplayPanel for each element we want in the UI. 

```C++
...
// Current page number
int pageNumber = 0;

// Last touched page
DisplayPanel* lastTouchedPanel = NULL;

// The display/lcd we are working with. Defined in initializePanels().
esphome::display::Display* lcd;


// The constructor arguments are X, Y, Width, Height
DisplayPanel datePanel(CONT_WIDTH, 0, DATE_WIDTH, DATE_HEIGHT);
DisplayPanel dayPanel(CONT_WIDTH, DATE_HEIGHT, DAY_WIDTH, DAY_HEIGHT);

// The pages of the application.
std::vector<std::vector<DisplayPanel*>> pages = {
    {
        // Page 0.
        &datePanel,
        &dayPanel
    }
};
...
```

## initializePanels(...), only at startup

The method `initializePanels(...)` should only be called once. It should define the initial font, color, text color, etc. for each of the DisplayPanels. Anything that doesn't change when the display is redrawn should be set here. 

```C++
void initializePanels(esphome::display::Display &display) {
    lcd = &display;

    dayPanel.font = font_day;
    dayPanel.color = Color::BLACK;
    dayPanel.textColor = color_text_white;

    datePanel.font = font_date;
    datePanel.color = Color::BLACK;
    datePanel.textColor = color_text_white;
}
```

## updatePanelStates(...), once per display update

Every time the display is updated / needs to be redrawn, the method `updatePanelStates(...)` should be used to define any changes to the states of any DisplayPanels. If your program is a clock, it should update one of the DisplayPanels with the current time each time `updatePanelStates(...)` is called.

At this time, the **one or more** strings of `.text` will always be printed in the **center** of the DisplayPanel.

```C++
void updatePanelStates() {
    auto now = esptime->now();

    // Day of the week
    std::string dayName = now.strftime("%A");
    std::vector<std::string> dayText = { dayName };
    dayPanel.text = dayText;

    // Date
    std::string monthName = now.strftime("%b");
    sprintf(buffer, "%s. %d", monthName.c_str(), now.day_of_month);
    std::vector<std::string> dateText = { buffer };
    datePanel.text = dateText;
}
```

## drawPanels(...), once per display update

This is where the panel is actually drawn to the display.

This should call `.draw(...)` for each DisplayPanel, and/or use the more optimized `DisplayPanel::drawAllPanels(...)` to more than one DisplayPanel in a single call.

```C++
void drawPanels(esphome::display::Display &display) {
    DisplayPanel::drawAllPanels(*lcd, pages[pageNumber]);
}
```

## Detecting touch, in the .yaml file

We need to create a method to check the enabled, toucable
panels on the current page to see if a touch is the target
of the touch.

```C++
// See if one of the enabled, touchable panels on the
// current page has been touched.
// lastTouchedPanel will be set to a pointer to the
// touched panel (or NULL of no panel was found for the coordinates).
boolean isPanelTouched(int tpX, int tpY) {
    lastTouchedPanel = DisplayPanel::touchedPanel(pages[pageNumber], tpX, tpY);
    return lastTouchedPanel != NULL;
}
```

The `on_touch:` stanza will check if a touch has occured and then perform the desired logic. In `tft-office.yaml`s `touch:` `on_touch:` stanza, we see where it checks for touch on the `contUpPanel` and `contDownPanel` DisplayPanels, adjust `brightness` accordingly, and sets the TFTs `backlight` to the value stored of `brightness`.

```yaml
  on_touch:
    then:
      - if:
          condition:
            lambda: |-
              return isPanelTouched((id(touch)).x, (id(touch)).y);
          then:
            - lambda: |-
                ESP_LOGD("yaml", "touched name=%s", (lastTouchedPanel->name.c_str()));
            - if:
                condition:
                  lambda: |-
                    return (lastTouchedPanel == &contUpPanel);
                then:
                  - lambda: |-
                      // Increase brightness 1%
                      id(brightness) = id(brightness) + 0.01 > 1 ? 1.0 : id(brightness) + 0.01;
                      id(backlight).set_level(id(brightness));
                      sprintf(buffer, "Increased to %.0f%%", id(brightness)*100);
                      enableFlash({"Brightness", buffer});
            - if:
                condition:
                  lambda: |-
                    return (lastTouchedPanel == &contDownPanel);
                then:
                  - lambda: |-
                      // Decrease brightness 1%
                      id(brightness) = id(brightness) - 0.01 < 0 ? 0.0 : id(brightness) - 0.01;
                      id(backlight).set_level(id(brightness));
                      sprintf(buffer, "Decreased to %.0f%%", id(brightness)*100);
                      enableFlash({"Brightness", buffer});
```
