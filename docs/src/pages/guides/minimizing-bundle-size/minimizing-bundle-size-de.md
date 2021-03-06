# Paketgröße minimieren

<p class="description">Erfahren Sie mehr über die Tools, mit denen Sie die Paketgröße reduzieren können.</p>

## Paketgröße zählt

Die Paketgröße von Material-UI wird sehr ernst genommen. We take size snapshots on every commit for every package and critical parts of those packages ([view the latest snapshot](/size-snapshot)). Wir können, kombiniert mit [dangerJS](https://danger.systems/js/), [detaillierte Änderungen der Bündelgröße](https://github.com/mui-org/material-ui/pull/14638#issuecomment-466658459) bei jedem Pull Request prüfen.

## Wie kann ich die Paketgröße reduzieren?

Der Einfachheit halber stellt Material-UI seine vollständige API auf der oberste Ebene des `material-ui` Imports zur Verfügung. If you're using ES6 modules and a bundler that supports tree-shaking ([`webpack` >= 2.x](https://webpack.js.org/guides/tree-shaking/), [`parcel` with a flag](https://en.parceljs.org/cli.html#enable-experimental-scope-hoisting/tree-shaking-support)) you can safely use named imports and expect only a minimal set of Material-UI components in your bundle:

```js
import { Button, TextField } from '@material-ui/core';
```

⚠️ Be aware that tree-shaking is an optimization that is usually only applied to production bundles. Development bundles will contain the full library which can lead to **slower startup times**. Dies macht sich insbesondere dann bemerkbar, wenn Sie aus `@material-ui/icons` importieren. Die Startzeiten können ungefähr 6-mal langsamer sein als ohne benannte Importe von der API der obersten Ebene.

If this is an issue for you, you have various options:

### Option 1

Sie können Pfadimporte verwenden, um zu vermeiden, dass nicht verwendete Module abgerufen werden. Zum Beispiel anstelle von:

```js
import { Button, TextField } from '@material-ui/core';
```

verwende:

```js
// 🚀 Fast
import Button from '@material-ui/core/Button';
import TextField from '@material-ui/core/TextField';
```

This is the option we document in **all** the demos because it requires no configuration. We encourage it for library authors extending our components. Head to [Option 2](#option-2) for the approach that yields the best DX and UX.

Beim direkten Importieren auf diese Weise werden die Exporte in [`@material-ui/core/index.js`](https://github.com/mui-org/material-ui/blob/master/packages/material-ui/src/index.js) nicht verwendet. Diese Datei kann trotzdem als praktische Referenz für die öffentlichen Module dienen.

Be aware that we only support first and second level imports. Alles drunter wird als privat betrachtet und kann zu einer Duplizierung des Moduls in Ihrem Bundle führen.

```js
// ✅ OK
import { Add as AddIcon } from '@material-ui/icons';
import { Tabs } from '@material-ui/core';
//                                 ^^^^ 1st or top-level

// ✅ OK
import AddIcon from '@material-ui/icons/Add';
import Tabs from '@material-ui/core/Tabs';
//                                  ^^^^ 2nd level

// ❌ NOT OK
import TabIndicator from '@material-ui/core/Tabs/TabIndicator';
//                                               ^^^^^^^^^^^^ 3rd level
```

### Option 2

This option provides the best DX and UX. However, you need to apply the following steps correctly.

#### 1. Configure Babel

Wählen Sie eines der folgenden Plugins:

- [babel-plugin-import](https://github.com/ant-design/babel-plugin-import) with the following configuration:
    
    `yarn add -D babel-plugin-import`
    
    Create a `.babelrc.js` file in the root directory of your project:

```js
  const plugins = [
    [
      'babel-plugin-import',
      {
        'libraryName': '@material-ui/core',
        // Use "'libraryDirectory': ''," if your bundler does not support ES modules
        'libraryDirectory': 'esm',
        'camel2DashComponentName': false
      },
      'core'
    ],
    [
      'babel-plugin-import',
      {
        'libraryName': '@material-ui/icons',
        // Use "'libraryDirectory': ''," if your bundler does not support ES modules
        'libraryDirectory': 'esm',
        'camel2DashComponentName': false
      },
      'icons'
    ]
  ];

  module.exports = {plugins};
  ```

- [babel-plugin-transform-imports](https://www.npmjs.com/package/babel-plugin-transform-imports) with the following configuration:

  `yarn add -D babel-plugin-transform-imports`

  Create a `.babelrc.js` file in the root directory of your project:

  ```js
  const plugins = [
    [
      'babel-plugin-transform-imports',
      {
        '@material-ui/core': {
          // Use "transform: '@material-ui/core/${member}'," if your bundler does not support ES modules
          'transform': '@material-ui/core/esm/${member}',
          'preventFullImport': true
        },
        '@material-ui/icons': {
          // Use "transform: '@material-ui/icons/${member}'," if your bundler does not support ES modules
          'transform': '@material-ui/icons/esm/${member}',
          'preventFullImport': true
        }
      }
    ]
  ];

  module.exports = {plugins};
  ```

If you are using Create React App, you will need to use a couple of projects that let you use `.babelrc` configuration, without ejecting. 

  `yarn add -D react-app-rewired customize-cra`

  Create a `config-overrides.js` file in the root directory:

  ```js
  /* config-overrides.js */
  const { useBabelRc, override } = require('customize-cra')

  module.exports = override(
    useBabelRc()
  );  
  ```

  If you wish, `babel-plugin-import` can be configured through `config-overrides.js` instead of `.babelrc` by using this [configuration](https://github.com/arackaf/customize-cra/blob/master/api.md#fixbabelimportslibraryname-options).

  Modify your `package.json` start command:

```diff
  "scripts": {
-  "start": "react-scripts start"
+  "start": "react-app-rewired start"
  }
```

    Note: You may run into errors like these:
    

        Module not found: Can't resolve '@material-ui/core/makeStyles' in '/your/project'
        Module not found: Can't resolve '@material-ui/core/createStyles' in '/your/project'
      ```
    
      This is because `@material-ui/styles` is re-exported through `core`, but the full import is not allowed.
    
      You have an import like this in your code:
    
      `import {makeStyles, createStyles} from '@material-ui/core';`
    
      The fix is simple, define the import separately:
    
      `import {makeStyles, createStyles} from '@material-ui/core/styles';`
    
      Enjoy significantly faster start times.
    
    #### 2. Convert all your imports
    
    Finally, you can convert your exisiting codebase to this option with our [top-level-imports](https://github.com/mui-org/material-ui/blob/master/packages/material-ui-codemod/README.md#top-level-imports) codemod.
    It will perform the following diffs:
    
    ```diff
    -import Button from '@material-ui/core/Button';
    -import TextField from '@material-ui/core/TextField';
    +import { Button, TextField } from '@material-ui/core';
    

## ECMAScript

Das auf npm veröffentlichte Paket ist mit [Babel](https://github.com/babel/babel) **transpiliert**, um die [ unterstützten Plattformen](/getting-started/supported-platforms/) zu berücksichtigen.

Wir veröffentlichen auch eine zweite Version der Komponenten. Sie finden diese Version unter den [`/es` Ordner](https://unpkg.com/@material-ui/core/es/). Die gesamte nicht offizielle Syntax wird auf den [ECMA-262 Standard](https://www.ecma-international.org/publications/standards/Ecma-262.htm) transpiliert, nichts mehr. Dies kann verwendet werden, um separate Bundles für verschiedene Browser zu erstellen. Ältere Browser erfordern mehr transpilierte JavaScript-Funktionen. Dies erhöht die Größe des Packets. Für die Laufzeitfunktionen von ES2015 sind keine polyfills enthalten. IE11 + und Evergreen-Browser unterstützen alle erforderlichen Funktionen. Wenn Sie Unterstützung für andere Browser benötigen, sollten Sie [`@babel/polyfill`](https://www.npmjs.com/package/@babel/polyfill) in Betracht ziehen.

⚠️ Um die Duplizierung von Code in Benutzerpaketen zu minimieren, raten wir **dringend davon ab** den `/es` Ordner zu benutzten.