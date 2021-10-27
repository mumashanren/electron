# Frameless Window

> Open a window without toolbars, borders, or other graphical "chrome".

A frameless window is a window that has no
[chrome](https://developer.mozilla.org/en-US/docs/Glossary/Chrome), the parts of
the window, like toolbars, that are not a part of the web page. These are
options on the [`BrowserWindow`](browser-window.md) class.

## Create a frameless window

To create a frameless window, you need to set `frame` to `false` in
[BrowserWindow](browser-window.md)'s `options`:

```javascript
const { BrowserWindow } = require('electron')
const win = new BrowserWindow({ width: 800, height: 600, frame: false })
win.show()
```

### Alternatives

There's an alternative way to specify a chromeless window on macOS and Windows.
Instead of setting `frame` to `false` which disables both the titlebar and window controls,
you may want to have the title bar hidden and your content extend to the full window size,
yet still preserve the window controls ("traffic lights" on macOS) for standard window actions.
You can do so by specifying the `titleBarStyle` option:

#### `hidden`

Results in a hidden title bar and a full size content window. On macOS, the title bar still has the standard window controls (“traffic lights”) in the top left.

```javascript
const { BrowserWindow } = require('electron')
const win = new BrowserWindow({ titleBarStyle: 'hidden' })
win.show()
```

### Alternatives on macOS

#### `hiddenInset`

Results in a hidden title bar with an alternative look where the traffic light buttons are slightly more inset from the window edge.

```javascript
const { BrowserWindow } = require('electron')
const win = new BrowserWindow({ titleBarStyle: 'hiddenInset' })
win.show()
```

#### `customButtonsOnHover`

Uses custom drawn close, and miniaturize buttons that display
when hovering in the top left of the window. The fullscreen button
is not available due to restrictions of frameless windows as they
interface with Apple's macOS window masks. These custom buttons prevent
issues with mouse events that occur with the standard window toolbar buttons.
This option is only applicable for frameless windows.

```javascript
const { BrowserWindow } = require('electron')
const win = new BrowserWindow({ titleBarStyle: 'customButtonsOnHover', frame: false })
win.show()
```

## Windows Control Overlay

When using a frameless window in conjuction with `win.setWindowButtonVisibility(true)` on macOS, using one of the `titleBarStyle`s as described above so
that the traffic lights are visible, or using `titleBarStyle: hidden` on Windows, you can access the Window Controls Overlay [JavaScript APIs][overlay-javascript-apis] and
[CSS Environment Variables][overlay-css-env-vars] by setting the `titleBarOverlay` option to true. Specifying `true` will result in an overlay with default system colors.

You can specify the height of the overlay by setting `titleBarOverlay` to an object with the option `height`. On Windows, you can also specify the color of the overlay and its symbols with the options `color` and `symbolColor`. If the `height` option is not specified, the default title bar height for that system is used. And if a color option is not specified, its system color for the window control buttons is used:

```javascript
const { BrowserWindow } = require('electron')
const win = new BrowserWindow({
  titleBarStyle: 'hidden',
  titleBarOverlay: true
})
win.show()
```

```javascript
const { BrowserWindow } = require('electron')
const win = new BrowserWindow({
  titleBarStyle: 'hidden',
  titleBarOverlay: {
    height: 40,
    color: '#2f3241',
    symbolColor: '#74b1be'
  }
})
win.show()
```

## Transparent window

By setting the `transparent` option to `true`, you can also make the frameless
window transparent:

```javascript
const { BrowserWindow } = require('electron')
const win = new BrowserWindow({ transparent: true, frame: false })
win.show()
```

### Limitations

* You can not click through the transparent area. We are going to introduce an
  API to set window shape to solve this, see
  [our issue](https://github.com/electron/electron/issues/1335) for details.
* Transparent windows are not resizable. Setting `resizable` to `true` may make
  a transparent window stop working on some platforms.
* The `blur` filter only applies to the web page, so there is no way to apply
  blur effect to the content below the window (i.e. other applications open on
  the user's system).
* The window will not be transparent when DevTools is opened.
* On Windows operating systems,
  * transparent windows will not work when DWM is
  disabled.
  * transparent windows can not be maximized using the Windows system menu or by double clicking the title bar. The reasoning behind this can be seen on [this pull request](https://github.com/electron/electron/pull/28207).
* On Linux, users have to put `--enable-transparent-visuals --disable-gpu` in
  the command line to disable GPU and allow ARGB to make transparent window,
  this is caused by an upstream bug that [alpha channel doesn't work on some
  NVidia drivers](https://bugs.chromium.org/p/chromium/issues/detail?id=369209) on
  Linux.
* On Mac, the native window shadow will not be shown on a transparent window.

## Click-through window

To create a click-through window, i.e. making the window ignore all mouse
events, you can call the [win.setIgnoreMouseEvents(ignore)][ignore-mouse-events]
API:

```javascript
const { BrowserWindow } = require('electron')
const win = new BrowserWindow()
win.setIgnoreMouseEvents(true)
```

### Forwarding

Ignoring mouse messages makes the web page oblivious to mouse movement, meaning
that mouse movement events will not be emitted. On Windows operating systems an
optional parameter can be used to forward mouse move messages to the web page,
allowing events such as `mouseleave` to be emitted:

```javascript
const { ipcRenderer } = require('electron')
const el = document.getElementById('clickThroughElement')
el.addEventListener('mouseenter', () => {
  ipcRenderer.send('set-ignore-mouse-events', true, { forward: true })
})
el.addEventListener('mouseleave', () => {
  ipcRenderer.send('set-ignore-mouse-events', false)
})

// Main process
const { ipcMain } = require('electron')
ipcMain.on('set-ignore-mouse-events', (event, ...args) => {
  BrowserWindow.fromWebContents(event.sender).setIgnoreMouseEvents(...args)
})
```

This makes the web page click-through when over `el`, and returns to normal
outside it.

## Draggable region

By default, the frameless window is non-draggable. Apps need to specify
`-webkit-app-region: drag` in CSS to tell Electron which regions are draggable
(like the OS's standard titlebar), and apps can also use
`-webkit-app-region: no-drag` to exclude the non-draggable area from the
 draggable region. Note that only rectangular shapes are currently supported.

Note: `-webkit-app-region: drag` is known to have problems while the developer tools are open. See this [GitHub issue](https://github.com/electron/electron/issues/3647) for more information including a workaround.

To make the whole window draggable, you can add `-webkit-app-region: drag` as
`body`'s style:

```html
<body style="-webkit-app-region: drag">
</body>
```

And note that if you have made the whole window draggable, you must also mark
buttons as non-draggable, otherwise it would be impossible for users to click on
them:

```css
button {
  -webkit-app-region: no-drag;
}
```

If you're only setting a custom titlebar as draggable, you also need to make all
buttons in titlebar non-draggable.

## Text selection

In a frameless window the dragging behavior may conflict with selecting text.
For example, when you drag the titlebar you may accidentally select the text on
the titlebar. To prevent this, you need to disable text selection within a
draggable area like this:

```css
.titlebar {
  -webkit-user-select: none;
  -webkit-app-region: drag;
}
```

## Context menu

On some platforms, the draggable area will be treated as a non-client frame, so
when you right click on it a system menu will pop up. To make the context menu
behave correctly on all platforms you should never use a custom context menu on
draggable areas.

[ignore-mouse-events]: browser-window.md#winsetignoremouseeventsignore-options
[overlay-javascript-apis]: https://github.com/WICG/window-controls-overlay/blob/main/explainer.md#javascript-apis
[overlay-css-env-vars]: https://github.com/WICG/window-controls-overlay/blob/main/explainer.md#css-environment-variables