It is possible to write VS Code extensions that are based on Code for IBM i. That means your extension can use the connection that the user creates in your extension. This is not an extension tutorial, but an intro on how to access the APIs available within Code for IBM i.

For example, you might be a vendor that produces lists or HTML that you'd like to be accessible from within Visual Studio Code.

## Imports

```
const { instance, CustomUI, Field } = vscode.extensions.getExtension(`halcyontechltd.code-for-ibmi`).exports;
```

We provide three base APIs for you to use. This document only covers `instance`. We have another page on [CustomUI](https://github.com/halcyon-tech/vscode-ibmi/blob/master/docs/api/custom-ui.md).

`instance` has four methods for you to use:

* `getConnection(): `[`IBMi`](https://github.com/halcyon-tech/vscode-ibmi/blob/master/src/api/IBMi.js)`|undefined` to get the current connection. Will return `undefined` when the current workspace is not connected to a remote system.
  * `IBMi` has methods that can be used to send ILE, QSH and pase commands to the IBM i.
* `getContent(): `[`IBMiContent`](https://github.com/halcyon-tech/vscode-ibmi/blob/master/src/api/IBMiContent.js) to work with content on the current connection
   * `IBMiContent` has methods to run SQL statements (requires `db2util`), get the contents of tables (without `db2util`) and read and write members/streamfiles.
* `getConfig(): `[`Configuration`](https://github.com/halcyon-tech/vscode-ibmi/blob/master/src/api/Configuration.js) to get/set configuration for the current connection
* `on(event: string, callback: Function): void` to add an event handler. Available events:
  * `connected` which can be used to determine when Code for IBM i has established a connection.

## Examples

See the following code bases for large examples of extensions that use Code for IBM i:

* [VS Code extension to manage IBM i IWS services](https://github.com/halcyon-tech/vscode-ibmi-iws)

### `connected` event

Using the `instance.on` method, your extension can determine when Code for IBM i has connected to a remote system:

```js
// The module 'vscode' contains the VS Code extensibility API
// Import the module and reference it with the alias vscode in your code below
const vscode = require(`vscode`);

const { instance } = vscode.extensions.getExtension(`halcyontechltd.code-for-ibmi`);

// this method is called when your extension is activated
// your extension is activated the very first time the command is executed

function activate(context) {
  // Use the console to output diagnostic information (console.log) and errors (console.error)
  // This line of code will only be executed once when your extension is activated
  console.log(`Congratulations, your extension "your-extention" is now active!`);

  instance.on(`connected`, () => {
    const config = instance.getConfig();
    
    console.log(`your-extention knows that you connected to ` + config.host);
  });

}

// this method is called when your extension is deactivated
function deactivate() {}

module.exports = {
  activate,
  deactivate
};
```

### Is there a connection?

You can use `instance.getConnection()` to determine if there is a connection:

```js
  async getChildren(element) {
    const connection = instance.getConnection();

    /** @type {vscode.TreeItem[]} */
    let items = [];

    if (connection) {
      //Do work here...

    } else {
      items = [new vscode.TreeItem(`Please connect to an IBM i and refresh.`)];
    }

    return items;
  }
```

### Temporary library

Please remember that you cannot use `QTEMP` between commands since each command runs in a new job. Please refer to `instance.getConfig().tempLibrary` for a temporary library.

### Running commands with the user library list

By default, the `remoteCommand` method does not follow the user library list from VS Code. You must write your own code to grab the library list and set it before executing the command. Due to the way the `system` command in pase works, we actually use `QSH` to set the library list and execute the command:

```js
  async runCommandWithLibl(ileCommand) {
    const connection = instance.getConnection();
    const content = instance.getContent();
    const config = instance.getConfig();

    //We need to add it backwards since `liblist` adds items to the front
    let libl = config.libraryList.slice(0).reverse();

    const result = await connection.qshCommand([
      //We also have to clear the initial library list to change it.
      `liblist -d ` + connection.defaultUserLibraries.join(` `),
      `liblist -c ` + config.currentLibrary,
      `liblist -a ` + libl.join(` `),
      `system -s "${ileCommand}"`,
    ]);
    
    return result;
  }
```

### Storing config specific to the connection

It is likely there will configuration that is specific to a connection. You can easily use `Configuration` to get and set configuration for the connection that is specific to your extension:

```js
const config = instance.getConfig();
let someArray = config.get(`someArray`);

if (!someArray) {
  //This means this config doesn't exist for the connection
  someArray = [];
}

someArray.push(someUserItem);

config.set(`someArray`, someArray);
```