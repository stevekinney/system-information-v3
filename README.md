# System Information

This is a super simple little application that we're going to use to learn how to get started building [Electron](https://electronjs.org) applications.

## Getting Set Up with the System Information Example Application

To install the dependencies:

```sh
npm install
```

To start up the application:

```sh
npm start
```

Electron will look for the `main` property in your `package.json`. In this example, this is set to `src/main.js`.

At this point, if you start up the application, you won't see much of anything, but you _will_ see that you have a new application running. It just doesn't do much just yet.

## Spawning Your First Window

In order to create your first window, add the following to `main.js`:

```js
const { app, BrowserWindow } = require('electron');

const createWindow = () => {
  const mainWindow = new BrowserWindow({
    width: 400,
    height: 600,
  });

  mainWindow.loadFile('./src/index.html');
};

app.whenReady().then(() => {
  createWindow();
});
```

Run `npm start` to start your application back up.

## Quitting the Application When All of the Windows Are Closed

Electron makes it easy to build applications that run on multiple platforms. That said, it's on use to add some of the functionality that makes our application fit in on each of those platforms.

- On macOS, you can close all of the windows and the application will remain open.
- On Windows and Linux, closing all of the applications will quit the application.

We can make our application behave appropriately by checking to see what platform we're running on and then taking the appropriate action. For example, let's add the following to `main.js`:

```javascript
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit();
});
```

Whenever all of the windows are closed, we're looking to see if we're _not_ running on a Mac. If so, we can safely quit the application.

## Opening a New Window Whenever the Application is Activated

We also need to do the opposite now. The typical behavior on macOS is to open a new window when the icon is clicked when no window is open.

Luckily, we don't _need_ to check what OS we're on since if there are _no_ open windows the application will have quit on Windows and Linux.

Let's update our `whenReady` hook to reflect this:

```javascript
app.whenReady().then(() => {
  createWindow();

  app.on('activate', () => {
    if (BrowserWindow.getAllWindows().length === 0) {
      createWindow();
    }
  });
});
```

## Using a Preload Script to Send Data From Node to Chromium

In modern versions of Electron, renderer processes cannot access Node APIs directly. (There are ways to get around this, but that creates a number of security concerns, so I won't be going into that here.)

We can use a special preload script to explicitly expose functionality from the main process to a given renderer process.

We'll use Node's `path` module to make our lives a little bit easier.

```diff
diff --git a/src/main.js b/src/main.js
index 8349a9b..89a7bc9 100644
--- a/src/main.js
+++ b/src/main.js
@@ -1,4 +1,5 @@
 const { app, BrowserWindow } = require('electron');
+const path = require('path');

 const createWindow = () => {
   const mainWindow = new BrowserWindow({
```

Next, we'll also tell the browser window where to find our preload script.

```diff
diff --git a/src/main.js b/src/main.js
index 89a7bc9..ea0a4c9 100644
--- a/src/main.js
+++ b/src/main.js
@@ -5,6 +5,9 @@ const createWindow = () => {
   const mainWindow = new BrowserWindow({
     width: 400,
     height: 600,
+    webPreferences: {
+      preload: path.join(__dirname, 'preload.js'),
+    },
   });

   mainWindow.loadFile('./src/index.html');
```

### Creating a Preload Script

Right now, `src/preload.js` is empty. Let's add some functionality. Preload scripts have special powers because they allow you to access all of the goodness of the Node.js world, but also have access to the DOM. They're allowed to do this because they're referenced from the main process and _not_ from a the renderer process, which could be vulnerable to script injection.

We can start by filling in the version fields once the DOM has loaded.

```js
window.addEventListener('DOMContentLoaded', () => {
  const nodeVersion = process.versions.node;
  const chromiumVersion = process.versions.chrome;
  const electronVersion = process.versions.electron;

  const nodeVersionElement = document.getElementById('node-version');
  const chromiumVersionElement = document.getElementById('chromium-version');
  const electronVersionElement = document.getElementById('electron-version');

  nodeVersionElement.innerText = nodeVersion;
  chromiumVersionElement.innerText = chromiumVersion;
  electronVersionElement.innerText = electronVersion;
});
```

## Interprocess Communication

This works for a one-time thing? But, one of the big promises of Electron is that we can access the system from the UI, right? Yes. But, we need to explictly expose this functionality.

There are a few ways to do this. One way us to use a `contextBridge`.

Versions don't change, but there are some other things like whether or not the user is on battery power, that might. Let's use an Electron API to read this informaiton and then we'll add that to the UI as well.

- The preload script can communicate with the main process using Interprocess Communication.
- The main process can expose functionality to the preload script.
- The preload script can expose this functionality to the Chromium scripts in the renderer process.

Let's add the following to the `src/main.js`:

```javascript
const { app, BrowserWindow, ipcMain, powerMonitor } = require('electron');

// …

ipcMain.handle('get-power-information', () => {
  const currentThermalState = powerMonitor.getCurrentThermalState();
  const isOnBatteryPower = powerMonitor.isOnBatteryPower();

  return {
    currentThermalState,
    isOnBatteryPower,
  };
});
```

Now, in `src/preload.js`, we talk to the main process and ask for this information.

```javascript
const { contextBridge, ipcRenderer } = require('electron');

const getPowerInformation = async () => {
  const result = await ipcRenderer.invoke('get-power-information');
  return result;
};

contextBridge.exposeInMainWorld('system', {
  getPowerInformation,
});
```

And now in `src/renderer.js`:

```javascript
const currentThermalStateElement = document.getElementById('thermal-state');
const onBatteryPowerElement = document.getElementById('battery-power');

const updatePowerInformation = async () => {
  const { currentThermalState, isOnBatteryPower } =
    await system.getPowerInformation();

  currentThermalStateElement.innerText = currentThermalState;
  onBatteryPowerElement.innerText = isOnBatteryPower;
};

updatePowerInformation();
```

### Wiring Up the Refresh Button

Lastly, we can add an event to the refresh button that will allow us to communicate back to the main process and then update the values in the DOM.

```javascript
document
  .getElementById('refresh')
  .addEventListener('click', updatePowerInformation);
```

## Exposing Values Using the Context Bridge

Right now, we're updating the version in the DOM when the DOM loads. But, what if we took the same approach as we did for `system.getPowerInformation` and also exposed the versions as global variables. We could do something like this in `src/preload.js`:

```javascript
contextBridge.exposeInMainWorld('versions', {
  node: process.versions.node,
  chromium: process.versions.chrome,
  electron: process.versions.electron,
});
```

And now, in `src/renderer.js`, we can add the following:

```javascript
const nodeVersionElement = document.getElementById('node-version');
const chromiumVersionElement = document.getElementById('chromium-version');
const electronVersionElement = document.getElementById('electron-version');

// …

nodeVersionElement.innerText = versions.node;
chromiumVersionElement.innerText = versions.chromium;
electronVersionElement.innerText = versions.electron;
```

The `diff` might look something like this:

```diff
diff --git a/src/preload.js b/src/preload.js
index 62bfb79..51e4321 100644
--- a/src/preload.js
+++ b/src/preload.js
@@ -9,16 +9,8 @@ contextBridge.exposeInMainWorld('system', {
   getPowerInformation,
 });

-window.addEventListener('DOMContentLoaded', () => {
-  const nodeVersion = process.versions.node;
-  const chromiumVersion = process.versions.chrome;
-  const electronVersion = process.versions.electron;
-
-  const nodeVersionElement = document.getElementById('node-version');
-  const chromiumVersionElement = document.getElementById('chromium-version');
-  const electronVersionElement = document.getElementById('electron-version');
-
-  nodeVersionElement.innerText = nodeVersion;
-  chromiumVersionElement.innerText = chromiumVersion;
-  electronVersionElement.innerText = electronVersion;
+contextBridge.exposeInMainWorld('versions', {
+  node: process.versions.node,
+  chromium: process.versions.chrome,
+  electron: process.versions.electron,
 });
diff --git a/src/renderer.js b/src/renderer.js
index 0736be8..afb4f21 100644
--- a/src/renderer.js
+++ b/src/renderer.js
@@ -1,5 +1,8 @@
 const currentThermalStateElement = document.getElementById('thermal-state');
 const onBatteryPowerElement = document.getElementById('battery-power');
+const nodeVersionElement = document.getElementById('node-version');
+const chromiumVersionElement = document.getElementById('chromium-version');
+const electronVersionElement = document.getElementById('electron-version');

 const updatePowerInformation = async () => {
   const { currentThermalState, isOnBatteryPower } =
@@ -9,6 +12,10 @@ const updatePowerInformation = async () => {
   onBatteryPowerElement.innerText = isOnBatteryPower;
 };

+nodeVersionElement.innerText = versions.node;
+chromiumVersionElement.innerText = versions.chromium;
+electronVersionElement.innerText = versions.electron;
+
 updatePowerInformation();

 document
```

## Exercise One: Exposing the Current User and Platform

Can you fill in the two missing fields: the current user and platform as part of that `system` property we added to `window`?

Both of these are available on the `process` object as `process.env.USER` and `process.platform` respectively.

## Exercise Two: Quitting the Application

Can you add an additional button to the DOM that communicates back to the main process and tells the application to quit?
