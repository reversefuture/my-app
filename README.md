# CLI
https://www.electronforge.io/cli#commands

## create-electron-app 
Quickly scaffold an Electron project with a full build pipeline
Init:
> npm init electron-app@latest my-app
with tempalte:
> npm init electron-app@latest my-app -- --template=webpack
### 用了template就不要再执行 npx electron-forge import了

Starting 
> cd my-app
> npm start

build:
> npm run make

publish:
> npm run publish

### cli
> npm install --save-dev @electron-forge/cli
Init:
We recommend using the create-electron-app script (which uses this command) to get started rather than running Init directly.
> npx electron-forge init --template=webpack
Import:
>npx electron-forge import
Package:
```shell
# By default, the package command corresponds to a package npm script:
npm run package -- --arch="ia32"
# If there is no package script:
npx electron-forge package --arch="ia32
```
Make:
```shell
# By default, the make command corresponds to a make npm script:
npm run make -- --arch="ia32"
# If there is no make script:
npx electron-forge make --arch="ia32"
```
make multiple:
> npm run make -- --arch="ia32,x64"
Publish:
```shell
# By default, the publish command corresponds to a publish npm script:
npm run publish -- --from-dry-run
# If there is no publish script:
npx electron-forge publish -- --from-dry-run
```
Pragmatic usage:
```js
const { api } = require('@electron-forge/core');

const main = async () => {
  await api.package({
    // add package command options here
  });
};

main();
```

# Start app
Electron is a native wrapper layer for web apps and is run in a Node.js environment.

The main script you defined in package.json is the entry point of any Electron application. This script controls the main process, which runs in a Node.js environment and is responsible for controlling your app's lifecycle, displaying native interfaces, performing privileged operations, and managing renderer processes (more on that later).

## REPL
Read-Eval-Print-Loop (REPL) is a simple, interactive computer programming environment that takes single user inputs (i.e. single expressions), evaluates them, and returns the result to the user.
> npx electron .
will tell the Electron executable to look for the main script in the current directory and run it in dev mode.

## Main process
Electron exposes the Node.js repl module through the --interactive CLI flag. Assuming you have electron installed as a local project dependency, you should be able to access the REPL with the following command:

> ./node_modules/.bin/electron --interactive

** Not available on windows **

## Loading a web page into a BrowserWindow
In Electron, each window displays a web page that can be loaded either from a local HTML file or a remote web address
```js
const { app, BrowserWindow } = require('electron')

const createWindow = () => {
  const win = new BrowserWindow({
    width: 800,
    height: 600
  })

  win.loadFile('index.html')
}

app.whenReady().then(() => {
  createWindow()
})
```

- app, which controls your application's event lifecycle.
- BrowserWindow, which creates and manages app windows.
Many of Electron's core modules are Node.js event emitters that adhere to Node's asynchronous event-driven architecture. The app module is one of these emitters.

Each web page your app displays in a window will run in **a separate process called a renderer process** (or simply renderer for short). Renderer processes have access to the same JavaScript APIs and tooling you use for typical front-end web development, such as using webpack to bundle and minify your code or React to build your user interfaces.

## Platform
Checking against Node's **process.platform** variable can help you to run code conditionally on certain platform.

### Quit the app when all windows are closed (Windows & Linux)
On Windows and Linux, closing all windows will generally quit an application entirely. 
```js
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit()
})
```

### macos
In contrast, macOS apps generally continue running even without any windows open.** Activating the app when no windows are available should open a new one.**
To implement this feature, listen for the app module's activate event, and call your existing createWindow() method if no BrowserWindows are open.
**Because windows cannot be created before the ready event**, you should only listen for activate events after your app is initialized. Do this by only listening for activate events inside your existing whenReady() callback.
```js
app.whenReady().then(() => {
  createWindow()

  app.on('activate', () => {
    if (BrowserWindow.getAllWindows().length === 0) createWindow()
  })
})
```

# debug
In the launch.json file  to create 3 configurations:
```js
{
  "version": "0.2.0",
  "compounds": [
    {
      "name": "Main + renderer",
      "configurations": ["Main", "Renderer"],
      "stopAll": true
    }
  ],
  "configurations": [
    {
      "name": "Renderer",
      "port": 9222,
      "request": "attach",
      "type": "chrome",
      "webRoot": "${workspaceFolder}"
    },
    {
      "name": "Main",
      "type": "node",
      "request": "launch",
      "cwd": "${workspaceFolder}",
      "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/electron",
      "windows": {
        "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/electron.cmd"
      },
      "args": [".", "--remote-debugging-port=9222"],
      "outputCapture": "std",
      "console": "integratedTerminal"
    }
  ]
}
```

Main is used to start the main process and also expose port 9222 for remote debugging (--remote-debugging-port=9222). This is the port that we will use to attach the debugger for the Renderer. Because the main process is a Node.js process, the type is set to node.
Renderer is used to debug the renderer process. Because the main process is the one that creates the process, we have to "attach" to it ("request": "attach") instead of creating a new one. The renderer process is a web one, so the debugger we have to use is chrome.
Main + renderer is a compound task that executes the previous ones simultaneously

# Preload
Electron's main process is a Node.js environment that has full operating system access. On top of Electron modules, you can also access Node.js built-ins, as well as any packages installed via npm. On the other hand, renderer processes run web pages and do not run Node.js by default for security reasons.

**To bridge Electron's different process types together, we will need to use a special script called a preload**.

## Augmenting the renderer with a preload script
From Electron 20 onwards, preload scripts are sandboxed by default and no longer have access to a full Node.js environment. Practically, this means that you have a polyfilled require function that only has access to a limited set of APIs.

Available API	Details: 
> Electron modules	Renderer process modules
> Node.js modules	events, timers, url
> Polyfilled globals	Buffer, process, clearImmediate, setImmediate
https://www.electronjs.org/docs/latest/tutorial/sandbox

**Preload scripts are injected before a web page loads in the renderer**, similar to a Chrome extension's content scripts. To add features to your renderer that require privileged access, **you can define global objects through the contextBridge API**.

preload.js
```js
const { contextBridge } = require('electron')
// contextBridge Create a safe, bi-directional, synchronous bridge across isolated contexts
contextBridge.exposeInMainWorld('versions', {
  node: () => process.versions.node,
  chrome: () => process.versions.chrome,
  electron: () => process.versions.electron
  // we can also expose variables, not just functions
})
```

main.js
```js
const { app, BrowserWindow } = require('electron')
const path = require('node:path')

const createWindow = () => {
  const win = new BrowserWindow({
    width: 800,
    height: 600,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js')
    }
  })

  win.loadFile('index.html')
}

app.whenReady().then(() => {
  createWindow()
})
```

use the version info in renderer.js:
```js
const information = document.getElementById('info')
information.innerText = `This app is using Chrome (v${versions.chrome()}), Node.js (v${versions.node()}), and Electron (v${versions.electron()})`
```
You can also access via  window.versions

In index.html:
```html
...
 <script src="./renderer.js"></script>
```

## Communicating between processes
As we have mentioned above, Electron's main and renderer process have distinct responsibilities and are not interchangeable.
The solution for this problem is to use Electron's ipcMain and ipcRenderer modules for inter-process communication (IPC). **To send a message from your web page to the main process**, you can set up a main process handler with ipcMain.handle and then expose a function that calls ipcRenderer.invoke to trigger the handler in your preload script.

**set up a invoke call in the preload script:**
```js
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('versions', {
  node: () => process.versions.node,
  chrome: () => process.versions.chrome,
  electron: () => process.versions.electron,
  ping: () => ipcRenderer.invoke('ping') // trigger handler
  // You never want to directly expose the entire ipcRenderer module via preload. This would give your renderer the ability to send arbitrary IPC messages to the main process, which becomes a powerful attack vector for malicious code.
  // we can also expose variables, not just functions
})
```

**set up your handle listener in the main process**
 ```js
 const { app, BrowserWindow, ipcMain } = require('electron')
...
app.whenReady().then(() => {
  ipcMain.handle('ping', () => 'pong') // setup handler
  createWindow()
})
 ```

Once you have the sender and receiver set up, you can now send messages from the renderer to the main process through the 'ping' channel you just defined.
renderer.js
```js
const func = async () => {
  const response = await window.versions.ping()
  console.log(response) // prints out 'pong'
}

func()
```

# Adding Features
https://www.electronjs.org/docs/latest/tutorial/examples

# Packaging Your Application
## Electron Forge Basic
Electron Forge is an all-in-one tool for packaging and distributing Electron applications.
Under the hood, it combines a lot of existing Electron tools (e.g. electron-packager, @electron/osx-sign, electron-winstaller, etc.) into a single interface so you do not have to worry about wiring them all together.
## Importing your project into Forge
> npm install --save-dev @electron-forge/cli
> npx electron-forge import

Once the conversion script is done, Forge should have added a few scripts to your package.json file.
```js
package.json
  //...
  "scripts": {
    "start": "electron-forge start",
    "package": "electron-forge package",
    "make": "electron-forge make"
  },
  //...
```

## Creating a distributable
> npm run make

It will first run electron-forge package under the hood, which bundles your app code together with the Electron binary. The packaged code is generated into a folder.
It will then use this packaged app folder to create a separate distributable for each configured maker.

## Important: signing your code
Code signing is an important part of shipping desktop applications, and is mandatory for the auto-update step in the final part of the tutorial.
On macOS, code signing is done at the app packaging level. On Windows, distributable installers are signed instead. If you already have code signing certificates for Windows and macOS, you can set your credentials in your Forge configuration.

forge.config.js:
```js
// mac os
module.exports = {
  packagerConfig: {
    osxSign: {},
    // ...
    osxNotarize: {
      tool: 'notarytool',
      appleId: process.env.APPLE_ID,
      appleIdPassword: process.env.APPLE_PASSWORD,
      teamId: process.env.APPLE_TEAM_ID
    }
    // ...
  }
}
// windows
module.exports = {
  // ...
  makers: [
    {
      name: '@electron-forge/maker-squirrel',
      config: {
        certificateFile: './cert.pfx',
        certificatePassword: process.env.CERTIFICATE_PASSWORD
      }
    }
  ]
  // ...
}
```

# Publishing and Updating
In this part, you will publish your app to GitHub releases and integrate automatic updates into your app code.

The Electron maintainers provide a free auto-updating service for open-source apps at https://update.electronjs.org. 

## Publishing a GitHub release
Electron Forge has Publisher plugins that can automate the distribution of your packaged application to various sources

### Generating a personal access token
 create a new personal access token (PAT) with the public_repo scope, which gives write access to your public repositories. Make sure to keep this token a secret.

### Setting up the GitHub Publisher
> npm install --save-dev @electron-forge/publisher-github

### Configuring the publisher in Forge
forge.config.js:
```js
module.exports = {
  publishers: [
    {
      name: '@electron-forge/publisher-github',
      config: {
        repository: {
          owner: 'github-user-name',
          name: 'github-repo-name'
        },
        prerelease: false,
        draft: true
      }
    }
  ]
}
```

### Setting up your authentication token
By default, it will use the value stored in the GITHUB_TOKEN environment variable.
> $env:GITHUB_TOKEN="x"
查看
> $env:GITHUB_TOKE
cmd:
> set GITHUB_TOKEN=x
查看
> echo %name%

### Running the publish command
package.json:
```js
  //...
  "scripts": {
    "start": "electron-forge start",
    "package": "electron-forge package",
    "make": "electron-forge make",
    "publish": "electron-forge publish"
  },
  //...
```
> npm run publish
You can publish for different architectures by passing in the --arch flag to your Forge commands.

### Bonus: Publishing in GitHub Actions
Publishing locally can be painful, especially because you can only create distributables for your host operating system (i.e. you can't publish a Window .exe file from macOS).
**we recommend setting up your building and publishing flow in a Continuous Integration pipeline if you do not have access to machines.**

A solution for this would be to publish your app via automation workflows such as GitHub Actions, which can run tasks in the cloud on Ubuntu, macOS, and Windows. This is the exact approach taken by Electron Fiddle. You can refer to Fiddle's Build and Release pipeline and Forge configuration for more details.

GitHub Actions makes it easy to automate all your software workflows, now with world-class CI/CD. Build, test, and deploy your code right from GitHub. Make code reviews, branch management, and issue triaging work the way you want.
https://docs.github.com/en/actions/quickstart


## Instrumenting your updater code
Electron apps do this via the autoUpdater module, which reads from an update server feed to check if a new version is available for download.
The update.electronjs.org service provides an updater-compatible feed.

After your release is published to GitHub, the update.electronjs.org service should work for your application. The only step left is to configure the feed with the autoUpdater module.

To make this process easier, the Electron team maintains the update-electron-app module, which sets up the autoUpdater boilerplate for update.electronjs.org in one function call — no configuration required. This module will search for the update.electronjs.org feed that matches your project's package.json "repository" field.
> npm install update-electron-app
Then, import the module and call it immediately in the main process.
main.js: 
> require('update-electron-app')()
And that is all it takes! Once your application is packaged, it will update itself for each new GitHub release that you publish.
### Using other update services
https://www.electronjs.org/docs/latest/tutorial/updates
