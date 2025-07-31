# Introduction

We were presented with the issue that our MobileDemand T850's, a tablet used in production, was running slow due to running on outdated bare metal. The team was tasked with upgrading such tablets with a operating system that could take better advantage of its resources. 

# Objective

Installing a version of Linux that would ensure the tablets stay in longer service while keeping it as debloated as possible. 

# Process

### Requirements

Specific versions of software:

* Ubuntu LTS 22.04
* GNOME shell 42.9
* Gnome-Extensions-Manager

Packages that were installed via apt:

* vim (because vim > nano just deal with it)
* openssh-server
* slick-greeter
* x11vnc

Package that needs to be **uninstalled** via apt:

* iio-sensor-proxy

Software / Hardware needed:

* Usb 3.0 > 8GB
* Rufus
* Wireless keyboard
* Patience


### Workflow

1. Grab .iso from Ubuntu website: 
   
   `https://releases.ubuntu.com/jammy/ubuntu-22.04.5-desktop-amd64.iso`

2. Burn it into a USB with Rufus or USB burner of your choice.

3. After its done burning, plug the USB into the tablet while it is turned off. 

4. Turn on the tablet and spam the 'DEL' key to enter BIOS.

5. Enter BIOS and find the boot menu and put the USB as first boot in boot priority.

6.  Save changes and exit.

7. Install Ubuntu when it is prompted. MAKE SURE YOU INSTALL MINIMAL VERSION WHEN PROMPTED TO PREVENT BLOATWARE.

8. When loaded into the desktop to run the following commands in terminal:
``` 
sudo apt remove iio-sensor-proxy
```
(Might have to restart after this.)

After this, go to Settings > Display and make the orientation "Portrait Left".

Then go back to terminal and run these commands: 
```
sudo apt update

sudo apt upgrade -y 

sudo apt install vim openssh-server slick-greeter x11vnc
```

NOTE: when you install slick-greeter something will pop up saying "Default Display Manager gdm3 or lightdm" . CHOOSE lightdm!

10. Next, launch firefox and disable the following features:
    * go to Settings > Privacy & Security
    * Disable `Ask to save passwords`
    * Disable `Save and fill addresses` and `Save and fill payment methods`
    * Switch Firefox will `Remember History` -> `Never remember history`
	* Disable `Send technical and interaction data to Mozilla` and `Send daily usage ping to Mozilla`
	* go to General 
	* Disable `Check your spelling as you type`

11. Now were going to add the specific files. You can ssh into it if you want to make the process easier. Use `ip a` to find ip address.

12. Run the following commands , paste the content provided and then save for the next 4 files:
```
sudo vim /etc/lightdm/lightdm.conf
```

```
[SeatDefaults]  
autologin-user=user
autologin-user-timeout=0  
user-session=ubuntu  
greeter-session=unity-greeter

sudo vim /etc/lightdm/lightdm.conf.d/50-myconfig.conf
```

```
sudo vim /etc/lightdm/lightdm.conf.d/50-myconfig.conf
```

```
[SeatDefaults]  
autologin-user=user
```

```
sudo mkdir /home/user/.config/autostart
```

```
sudo vim /home/user/.config/autostart/user.desktop
```

```
[Desktop Entry]  
Type=Application  
Name=user 
Exec=/home/user/user.sh  
X-GNOME-Autostart-enabled=true
```

```
sudo vim /home/user/user.sh
```

```
#!/bin/bash  
  
# Runs in display 0  
export DISPLAY=:0  
  
# Hide mouse from the display uncomment if you want  
# unclutter &  
  
# start x11vnc for observer to connect to debug  
  
/usr/bin/x11vnc -create -xkb -noxrecord -noxfixes -noxdamage -display :0 -auth /var/run/lightdm/root/:0 -rfbauth /home/user/.x11vnc.pass -forever -rfbport 5905 &  
  
# run firefox and open tabs  
firefox --kiosk <website> & 
```

```
sudo chmod +x /home/user/user.sh
```

13. Set a VNC password running using:
```
x11vnc -storepasswd (password) .x11vnc.pass
```
NOTE:  To connect to x11vnc use port :5905

14. Run `sudo reboot` to ensure auto start works properly

(OPTIONAL BUT GOOD QOL):

Run the command to make GNOME text scale better:
```
gsettings set org.gnome.desktop.interface text-scaling-factor 1.5
```

For TTY to be on the side the tablet sits on and not vertical, add this line to the grub:
```
fbcon=rotate:1
```
To do this use the following commands:
```
sudo vim /etc/default/grub
```
Find the following line: 
```
GRUB_CMDLINE_LINUX_DEFAULT= '...'
```
Append fbcon=rotate:1 to end of that line.
```
GRUB_CMDLINE_LINUX_DEFAULT='quiet splash fbcon=rotate:1'
```
Save your changes then run:
```
sudo update-grub

sudo reboot
```



We will be upgrading the keyboard now.

1. Download the gnome extensions via the `Ubuntu Software` app. It will be listed as 'Extensions Manager' by Matthew Jakeman.
   
2. Once you are done downloading, open it and go to Browse section and look up 'Improved OSK' . Package will be handled by someone called `improvedosk@nick-shmyrev.dev` and install it.
   
3.  After that is done installing, go back to Extensions Manager and click on the settings wheel in 'Improved OSK' under User-Installed Extensions.

4.  The settings that we want to change here are:
	* Landscape Height in percent: 50 
    * Resize Desktop (Shell restart required) ACTIVE

5.  Now we are going to configure some files to better fit our needs.

6. Do these following commands:
```
cd ~/.local/share/gnome-shell/extensions/improvedosk@nick-shmyrev.dev
```
There will be a file called extension.js which we will edit.

Do these commands: 
```
touch ext.js

sudo vim ext.js
```
Then copy this script into `ext.js`. It is a modified version of the original extension.js to fit our needs.

```
"use strict";
const { Gio, St, Clutter, GObject } = imports.gi;
const Main = imports.ui.main;
const Keyboard = imports.ui.keyboard;
const PanelMenu = imports.ui.panelMenu;
const ExtensionUtils = imports.misc.extensionUtils;
const Me = ExtensionUtils.getCurrentExtension();
const InputSourceManager = imports.ui.status.keyboard;

const A11Y_APPLICATIONS_SCHEMA = "org.gnome.desktop.a11y.applications";
let _oskA11yApplicationsSettings;
let backup_lastDeviceIsTouchScreen;
let backup_relayout;
let backup_DefaultKeysForRow;
let backup_keyboardControllerConstructor;
let backup_keyvalPress;
let backup_keyvalRelease;
let backup_commitString;
let backup_loadDefaultKeys;
let _indicator;
let settings;

// Indicator
let OSKIndicator = GObject.registerClass(
  { GTypeName: "OSKIndicator" },
  class OSKIndicator extends PanelMenu.Button {
    _init() {
      super._init(0.0, `${Me.metadata.name} Indicator`, false);

      let icon = new St.Icon({
        icon_name: "input-keyboard-symbolic",
        style_class: "system-status-icon",
      });

      this.add_child(icon);

      this.connect("button-press-event", function (actor, event) {
        let button = event.get_button();

        if (button == 1) {
          if (Main.keyboard._keyboard._keyboardVisible) return Main.keyboard.close();

          Main.keyboard.open(Main.layoutManager.bottomIndex);
        }
        if (button == 3) {
          ExtensionUtils.openPrefs();
        }
      });

      this.connect("touch-event", function () {
        if (Main.keyboard._keyboard._keyboardVisible) return Main.keyboard.close();

        Main.keyboard.open(Main.layoutManager.bottomIndex);
      });
    }
  }
);

// Overrides
function override_lastDeviceIsTouchScreen() {
  if (!this._lastDevice) return false;

  let deviceType = this._lastDevice.get_device_type();

  return settings.get_boolean("ignore-touch-input")
    ? false
    : deviceType == Clutter.InputDeviceType.TOUCHSCREEN_DEVICE;
}

function override_relayout() {
  let monitor = Main.layoutManager.keyboardMonitor;

  if (!monitor) return;

  this.width = monitor.width;

  if (monitor.width > monitor.height) {
    this.height = (monitor.height * settings.get_int("landscape-height")) / 100;
  } else {
    this.height = (monitor.height * settings.get_int("portrait-height")) / 100;
  }
}


function override_keyvalPress(keyval) {
  // This allows manually releasing a latched ctrl/super/alt keys by tapping on them again
  if (keyval == Clutter.KEY_Control_L) {
    this._controlActive = !this._controlActive;

    if (this._controlActive) {
      this._virtualDevice.notify_keyval(
          Clutter.get_current_event_time(),
          Clutter.KEY_Control_L,
          Clutter.KeyState.PRESSED
      );
      Main.layoutManager.keyboardBox.add_style_class_name("control-key-latched");
    } else {
      this._virtualDevice.notify_keyval(
          Clutter.get_current_event_time(),
          Clutter.KEY_Control_L,
          Clutter.KeyState.RELEASED
      );
      Main.layoutManager.keyboardBox.remove_style_class_name("control-key-latched");
    }

    return;
  }

  if (keyval == Clutter.KEY_Super_L) {
    this._superActive = !this._superActive;

    if (this._superActive) {
      this._virtualDevice.notify_keyval(
          Clutter.get_current_event_time(),
          Clutter.KEY_Super_L,
          Clutter.KeyState.PRESSED
      );
      Main.layoutManager.keyboardBox.add_style_class_name("super-key-latched");
    } else {
      this._virtualDevice.notify_keyval(
          Clutter.get_current_event_time(),
          Clutter.KEY_Super_L,
          Clutter.KeyState.RELEASED
      );
      Main.layoutManager.keyboardBox.remove_style_class_name("super-key-latched");
    }

    return;
  }

  if (keyval == Clutter.KEY_Alt_L) {
    this._altActive = !this._altActive;

    if (this._altActive) {
      this._virtualDevice.notify_keyval(
          Clutter.get_current_event_time(),
          Clutter.KEY_Alt_L,
          Clutter.KeyState.PRESSED
      );
      Main.layoutManager.keyboardBox.add_style_class_name("alt-key-latched");
    } else {
      this._virtualDevice.notify_keyval(
          Clutter.get_current_event_time(),
          Clutter.KEY_Alt_L,
          Clutter.KeyState.RELEASED
      );
      Main.layoutManager.keyboardBox.remove_style_class_name("alt-key-latched");
    }

    return;
  }


  // Not a ctrl/super/alt key, continue down original execution path
  this._virtualDevice.notify_keyval(
      Clutter.get_current_event_time(),
      keyval,
      Clutter.KeyState.PRESSED
  );
}
 function override_keyvalRelease(keyval) {
  // By default each key is released immediately after being pressed.
  // Don't release ctrl/alt/super keys to allow them to be latched
  // and used in "ctrl/alt/super + key" combinations
  if (
      keyval == Clutter.KEY_Control_L
      || keyval == Clutter.KEY_Alt_L
      || keyval == Clutter.KEY_Super_L
  ) {
    return;
  }

  this._virtualDevice.notify_keyval(
      Clutter.get_current_event_time(),
      keyval,
      Clutter.KeyState.RELEASED
  );

  if (this._controlActive) {
    this._virtualDevice.notify_keyval(
        Clutter.get_current_event_time(),
        Clutter.KEY_Control_L,
        Clutter.KeyState.RELEASED
    );
    this._controlActive = false;
    Main.layoutManager.keyboardBox.remove_style_class_name("control-key-latched");
  }
  if (this._superActive) {
    this._virtualDevice.notify_keyval(
        Clutter.get_current_event_time(),
        Clutter.KEY_Super_L,
        Clutter.KeyState.RELEASED
    );
    this._superActive = false;
    Main.layoutManager.keyboardBox.remove_style_class_name("super-key-latched");
  }
  if (this._altActive) {
    this._virtualDevice.notify_keyval(
        Clutter.get_current_event_time(),
        Clutter.KEY_Alt_L,
        Clutter.KeyState.RELEASED
    );
    this._altActive = false;
    Main.layoutManager.keyboardBox.remove_style_class_name("alt-key-latched");
  }
}

function override_getDefaultKeysForRow(row, numRows, level) {
let defaultKeysPreMod = [
  [
    [],
    [],
    [],
    [],
  ],
  [
    [],
    [],
    [],
    [],
  ],
  [
    [],
    [],
    [],
    [],
  ],
  [
    [],
    [],
    [],
    [],
  ],
];

let defaultKeysPostMod = [
    [
      [
        { label: "⌫", width: 1, keyval: Clutter.KEY_BackSpace },
       // { label: "⌦", width: 1, keyval: Clutter.KEY_Delete },
        { label: "⇊", width: 1, action: "hide", extraClassName: "hide-key" },
      ],
      [
        {
          label: "⏎",
          width: 1.5,
          keyval: Clutter.KEY_Return,
          extraClassName: "enter-key",
        },
       /** {
          label: "🗺",
          width: 1.5,
          action: "languageMenu",
          extraClassName: "layout-key",
        }, **/
      ],
      [
        {
          label: "⇑",
          width: 1,
          level: 1,
          right: true,
          extraClassName: "shift-key-lowercase",
        },
        { label: "?123", width: 1, level: 2 },
      ],
      [
        //{ label: "←", width: 1, keyval: Clutter.KEY_Left },
        //{ label: "↑", width: 1, keyval: Clutter.KEY_Up },
        //{ label: "↓", width: 1, keyval: Clutter.KEY_Down },
        //{ label: "→", width: 1, keyval: Clutter.KEY_Right },
      ],
    ],
    [
      [
        { label: "⌫", width: 1.5, keyval: Clutter.KEY_BackSpace },
       // { label: "⌦", width: 1, keyval: Clutter.KEY_Delete },
        { label: "⇊", width: 1, action: "hide", extraClassName: "hide-key" },
      ],
      [
        {
          label: "⏎",
          width: 1.5,
          keyval: Clutter.KEY_Return,
          extraClassName: "enter-key",
        },
       /** {
          label: "🗺",
          width: 1.5,
          action: "languageMenu",
          extraClassName: "layout-key",
        }, **/
      ],
      [
        {
          label: "⇑",
          width: 1,
          level: 0,
          right: true,
          extraClassName: "shift-key-uppercase",
        },
        { label: "?123", width: 1, level: 2 },
      ],
     // [
       // { label: "←", width: 1, keyval: Clutter.KEY_Left },
       // { label: "↑", width: 1, keyval: Clutter.KEY_Up },
       // { label: "↓", width: 1, keyval: Clutter.KEY_Down },
       // { label: "→", width: 1, keyval: Clutter.KEY_Right },
     // ],
    ],
    [
      [
        { label: "⌫", width: 1.5, keyval: Clutter.KEY_BackSpace },
       // { label: "⌦", width: 1, keyval: Clutter.KEY_Delete },
        { label: "⇊", width: 1, action: "hide", extraClassName: "hide-key" },
      ],
      [
        {
          label: "⏎",
          width: 1.5,
          keyval: Clutter.KEY_Return,
        },
       /** {
          label: "🗺",
          width: 1.5,
          action: "languageMenu",
          extraClassName: "layout-key",
        }, **/
      ],
      [
      // { label: "=/<F", width: 3, level: 3, right: true },
       { label: "ABC", width: 1, level: 0 },
      ],
     // [
       // { label: "←", width: 1, keyval: Clutter.KEY_Left },
       // { label: "↑", width: 1, keyval: Clutter.KEY_Up },
       // { label: "↓", width: 1, keyval: Clutter.KEY_Down },
       //{ label: "→", width: 1, keyval: Clutter.KEY_Right },
      //],
    ],
    [
      [
       // { label: "F1", width: 1, keyval: Clutter.KEY_F1 },
       // { label: "F2", width: 1, keyval: Clutter.KEY_F2 },
       // { label: "F3", width: 1, keyval: Clutter.KEY_F3 },
        { label: "⌫", width: 1.5, keyval: Clutter.KEY_BackSpace },
       // { label: "⌦", width: 1, keyval: Clutter.KEY_Delete },
        { label: "⇊", width: 1, action: "hide", extraClassName: "hide-key" },
      ],
      [
       // { label: "F4", width: 1, keyval: Clutter.KEY_F4 },
       // { label: "F5", width: 1, keyval: Clutter.KEY_F5 },
       // { label: "F6", width: 1, keyval: Clutter.KEY_F6 },
        {
          label: "⏎",
          width: 1.5,
          keyval: Clutter.KEY_Return,
          extraClassName: "enter-key",
        },
       /** {
          label: "🗺",
          width: 1.5,
          action: "languageMenu",
          extraClassName: "layout-key",
        },**/
      ],
      [
       // { label: "F7", width: 1, keyval: Clutter.KEY_F7 },
       // { label: "F8", width: 1, keyval: Clutter.KEY_F8 },
       // { label: "F9", width: 1, keyval: Clutter.KEY_F9 },
        { label: "?123", width: 1, level: 2, right: true },
        { label: "ABC", width: 1, level: 0 },
      ],
     // [
       // { label: "F10", width: 1, keyval: Clutter.KEY_F10 },
       // { label: "F11", width: 1, keyval: Clutter.KEY_F11 },
       // { label: "F12", width: 1, keyval: Clutter.KEY_F12 },
       // { label: "←", width: 1, keyval: Clutter.KEY_Left },
       // { label: "↑", width: 1, keyval: Clutter.KEY_Up },
       // { label: "↓", width: 1, keyval: Clutter.KEY_Down },
       // { label: "→", width: 1, keyval: Clutter.KEY_Right },
      //],
    ],
  ];

  /* The first 2 rows in defaultKeysPre/Post belong together with
   * the first 2 rows on each keymap. On keymaps that have more than
   * 4 rows, the last 2 default key rows must be respectively
   * assigned to the 2 last keymap ones.
   */
  if (row < 2) {
    return [defaultKeysPreMod[level][row], defaultKeysPostMod[level][row]];
  } else if (row >= numRows - 2) {
    let defaultRow = row - (numRows - 2) + 2;
    return [
      defaultKeysPreMod[level][defaultRow],
      defaultKeysPostMod[level][defaultRow],
    ];
  } else {
    return [null, null];
  }
}

function override_keyboardControllerConstructor() {
  let deviceManager = Clutter.DeviceManager.get_default();
  this._virtualDevice = deviceManager.create_virtual_device(
    Clutter.InputDeviceType.KEYBOARD_DEVICE
  );

  this._inputSourceManager = InputSourceManager.getInputSourceManager();
  this._sourceChangedId = this._inputSourceManager.connect(
    "current-source-changed",
    this._onSourceChanged.bind(this)
  );
  this._sourcesModifiedId = this._inputSourceManager.connect(
    "sources-changed",
    this._onSourcesModified.bind(this)
  );
  this._currentSource = this._inputSourceManager.currentSource;

  this._controlActive = false;
  this._superActive = false;
  this._altActive = false;

  Main.inputMethod.connect(
    "notify::content-purpose",
    this._onContentPurposeHintsChanged.bind(this)
  );
  Main.inputMethod.connect(
    "notify::content-hints",
    this._onContentPurposeHintsChanged.bind(this)
  );
  Main.inputMethod.connect("input-panel-state", (o, state) => {
    this.emit("panel-state", state);
  });
}

function override_commitString(string, fromKey) {
  // Prevents alpha-numeric key presses from bypassing override_keyvalPress()
  // while ctrl/alt/super are latched
  if (
      this._controlActive
      || this._superActive
      || this._altActive
  ) {
    return false;
  }

  if (string == null) return false;
  /* Let ibus methods fall through keyval emission */
  if (fromKey && this._currentSource.type == InputSourceManager.INPUT_SOURCE_TYPE_IBUS) return false;

  Main.inputMethod.commit(string);
  return true;
}

// Bulk of this method remains unchanged, except for extraButton.connect('released') event listener.
// Overriding it to ensure latched ctrl/alt/super keys are released before keyboard is hidden
function override_loadDefaultKeys(keys, layout, numLevels, numKeys) {
  let extraButton;
  for (let i = 0; i < keys.length; i++) {
    let key = keys[i];
    let keyval = key.keyval;
    let switchToLevel = key.level;
    let action = key.action;
    let icon = key.icon;

    /* Skip emoji button if necessary */
    if (!this._emojiKeyVisible && action == 'emoji')
      continue;

    extraButton = new Keyboard.Key(key.label || '', [], icon);

    extraButton.keyButton.add_style_class_name('default-key');
    if (key.extraClassName != null)
      extraButton.keyButton.add_style_class_name(key.extraClassName);
    if (key.width != null)
        extraButton.setWidth(key.width);

    let actor = extraButton.keyButton;

    extraButton.connect('pressed', () => {
      if (switchToLevel != null) {
        this._setActiveLayer(switchToLevel);
        // Shift only gets latched on long press
        this._latched = switchToLevel != 1;
      } else if (keyval != null) {
        this._keyboardController.keyvalPress(keyval);
      }
    });
    extraButton.connect('released', () => {
      // === Override starts here ===
      if (keyval != null) return this._keyboardController.keyvalRelease(keyval);

      switch (action) {
        case 'hide':
          // Press latched ctrl/super/alt keys again to release them before hiding OSK
          if (this._keyboardController._controlActive) this._keyboardController.keyvalPress(Clutter.KEY_Control_L);
          if (this._keyboardController._superActive) this._keyboardController.keyvalPress(Clutter.KEY_Super_L);
          if (this._keyboardController._altActive) this._keyboardController.keyvalPress(Clutter.KEY_Alt_L);

          this.close();
          break;

        case 'languageMenu':
          this._popupLanguageMenu(actor);
          break;

        case 'emoji':
          this._toggleEmoji();
          break;

        // no default
      }
      // === Override ends here ===
    });

    if (switchToLevel == 0) {
      layout.shiftKeys.push(extraButton);
    } else if (switchToLevel == 1) {
      extraButton.connect('long-press', () => {
        this._latched = true;
        this._setCurrentLevelLatched(this._currentPage, this._latched);
      });
    }

    /* Fixup default keys based on the number of levels/keys */
    if (switchToLevel == 1 && numLevels == 3) {
      // Hide shift key if the keymap has no uppercase level
      if (key.right) {
        /* Only hide the key actor, so the container still takes space */
        extraButton.keyButton.hide();
      } else {
        extraButton.hide();
      }
      extraButton.setWidth(1.5);
    } else if (key.right && numKeys > 8) {
      extraButton.setWidth(2);
    } else if (keyval == Clutter.KEY_Return && numKeys > 9) {
      extraButton.setWidth(1.5);
    } else if (!this._emojiKeyVisible && (action == 'hide' || action == 'languageMenu')) {
      extraButton.setWidth(1.5);
    }

    layout.appendKey(extraButton, extraButton.keyButton.keyWidth);
  }
}

function enable_overrides() {
  Keyboard.Keyboard.prototype["_relayout"] = override_relayout;
  Keyboard.Keyboard.prototype["_loadDefaultKeys"] = override_loadDefaultKeys;
  Keyboard.Keyboard.prototype["_getDefaultKeysForRow"] = override_getDefaultKeysForRow;

  Keyboard.KeyboardController.prototype["constructor"] = override_keyboardControllerConstructor;
  Keyboard.KeyboardController.prototype["keyvalPress"] = override_keyvalPress;
  Keyboard.KeyboardController.prototype["keyvalRelease"] = override_keyvalRelease;
  Keyboard.KeyboardController.prototype["commitString"] = override_commitString;

  Keyboard.KeyboardManager.prototype["_lastDeviceIsTouchscreen"] = override_lastDeviceIsTouchScreen;
}

function disable_overrides() {
  Keyboard.Keyboard.prototype["_relayout"] = backup_relayout;
  Keyboard.Keyboard.prototype["_loadDefaultKeys"] = backup_loadDefaultKeys;
  Keyboard.Keyboard.prototype["_getDefaultKeysForRow"] = backup_DefaultKeysForRow;

  Keyboard.KeyboardController.prototype["constructor"] = backup_keyboardControllerConstructor;
  Keyboard.KeyboardController.prototype["keyvalPress"] = backup_keyvalPress;
  Keyboard.KeyboardController.prototype["keyvalRelease"] = backup_keyvalRelease;
  Keyboard.KeyboardController.prototype["commitString"] = backup_commitString;

  Keyboard.KeyboardManager.prototype["_lastDeviceIsTouchscreen"] = backup_lastDeviceIsTouchScreen;
}

// Extension
function init() {
  backup_relayout = Keyboard.Keyboard.prototype["_relayout"];
  backup_loadDefaultKeys = Keyboard.Keyboard.prototype["_loadDefaultKeys"]
  backup_DefaultKeysForRow = Keyboard.Keyboard.prototype["_getDefaultKeysForRow"];

  backup_keyboardControllerConstructor = Keyboard.KeyboardController.prototype["constructor"];
  backup_keyvalPress = Keyboard.KeyboardController.prototype["keyvalPress"];
  backup_keyvalRelease = Keyboard.KeyboardController.prototype["keyvalRelease"];
  backup_commitString = Keyboard.KeyboardController.prototype["commitString"];

  backup_lastDeviceIsTouchScreen = Keyboard.KeyboardManager._lastDeviceIsTouchscreen;
}

function enable() {
  settings = ExtensionUtils.getSettings(
      "org.gnome.shell.extensions.improvedosk"
  );
  _oskA11yApplicationsSettings = new Gio.Settings({
    schema_id: A11Y_APPLICATIONS_SCHEMA,
  });

  Main.layoutManager.removeChrome(Main.layoutManager.keyboardBox);

  // Set up the indicator in the status area
  if (settings.get_boolean("show-statusbar-icon")) {
    _indicator = new OSKIndicator();
    Main.panel.addToStatusArea("OSKIndicator", _indicator);
  }

  let KeyboardIsSetup = true;
  try {
    Main.keyboard._destroyKeyboard();
  } catch (e) {
    if (e instanceof TypeError) {
      // In case the keyboard is currently disabled in accessability settings, attempting to _destroyKeyboard() yields a TypeError ("TypeError: this.actor is null")
      // This doesn't affect functionality, so proceed as usual. The only difference is that we do not automatically _setupKeyboard at the end of this enable() (let the user enable the keyboard in accessability settings)
      KeyboardIsSetup = false;
    } else {
      // Something different happened
      throw e;
    }
  }

  enable_overrides();

  settings.connect("changed::show-statusbar-icon", function () {
    if (settings.get_boolean("show-statusbar-icon")) {
      _indicator = new OSKIndicator();
      Main.panel.addToStatusArea("OSKIndicator", _indicator);
    } else if (_indicator !== null) {
      _indicator.destroy();
      _indicator = null;
    }
  });

  if (KeyboardIsSetup) {
    Main.keyboard._setupKeyboard();
  }

  Main.layoutManager.addTopChrome(Main.layoutManager.keyboardBox, {
    affectsStruts: settings.get_boolean("resize-desktop"),
    trackFullscreen: false,
  });
}

function disable() {
  Main.layoutManager.removeChrome(Main.layoutManager.keyboardBox);

  let KeyboardIsSetup = true;
  try {
    Main.keyboard._destroyKeyboard();
  } catch (e) {
    if (e instanceof TypeError) {
      // In case the keyboard is currently disabled in accessability settings, attempting to _destroyKeyboard() yields a TypeError ("TypeError: this.actor is null")
      // This doesn't affect functionality, so proceed as usual. The only difference is that we do not automatically _setupKeyboard at the end of this enable() (let the user enable the keyboard in accessability settings)
      KeyboardIsSetup = false;
    } else {
      // Something different happened
      throw e;
    }
  }

  // Remove indicator if it exists
  if (_indicator instanceof OSKIndicator) {
    _indicator.destroy();
    _indicator = null;
  }

  settings = null;

  disable_overrides();

  if (KeyboardIsSetup) {
    Main.keyboard._setupKeyboard();
  }
  Main.layoutManager.addTopChrome(Main.layoutManager.keyboardBox);
}
```
Save ext.js and run the last command.

After this run the last command:
```
mv ext.js extension.js
```

That is how you make the keyboard better. Feel free to customize it better to your liking if you feel like it. 


### Conclusion

Even though I consider myself an experienced Linux user when it comes to customizing/ricing, I ran into a lot of challenges. This was also my first time ever working with customizing a touchscreen device. The hardest challenge to overcome was having an on-screen-keyboard since I tried everything before I stumbled across the gnome OSK extension to modify the inbuilt OSK. After that it was quite easy getting everything else to work. 

I also added our patching software to these tablets which was another process but a pretty easy one. The process is not listed due to security reasons.

There are also some stuff that I probably have not found that need to be fixed -- this doc will be updated at the end with stuff/bugs I found and fixed.

Special thanks to my job for trusting me with this it took a lot of frustration but I am very happy I could make it work for my team (:
