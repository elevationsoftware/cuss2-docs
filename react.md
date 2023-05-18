# [CUSS React](https://www.blndspt.com/cuss-2-0/) &middot; [![GitHub license](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/facebook/react/blob/master/LICENSE) [![npm version](https://img.shields.io/npm/v/react.svg?style=flat)](https://www.npmjs.com/package/react)

***[CUSS (Common Use Self-Service)](https://en.wikipedia.org/wiki/Common-use_self-service) React*** is a modern [Typescript](https://en.wikipedia.org/wiki/TypeScript) library facilitating rapid application development of Self-Service check-in apps, self-tagging apps and self bag-drop apps using React's common principles.

The library and corresponding app platform also ensure backwards compatibility to legacy 1.X versions of CUSS.

You can have ***CUSS 2.0 NOW*** and run a modern browser entirely without plugins or Java.  Finally, your Information Security department will be able to sign off on your CUSS applications.



## Installation

CUSS React is a private NPM package and requires a `.npmrc` file at the top level of your React project.  Acquire an NPM token from us and place it in this file as such:

`//registry.npmjs.org/:_authToken=TOKEN-GOES-HERE`

With such a file in place, simple install the libraries:

```
npm install @elevated/cuss2 @elevated/cuss2/react
```
OR
```
yarn add @elevated/cuss2 @elevated/cuss2/react
```

## Basic Usage

Wrap your App in a Cuss Connector and choose your desired defaults.  This will connect your application to a CUSS Platform and let you use all the declaritive hooks in other components.

See the Sandbox Section below for information on how to setup your configuration and secrets.

```jsx
import React from 'react';
import ReactDOM from 'react-dom';
import App from './App';
import { Cuss2Connector, ICuss2ServiceOptions } from '@elevated-libs/cuss2-react';

// Pull from an env file or json config
const options:ICuss2ServiceOptions = {
  cussUrl: 'xxxx',
  clientId: 'XX',
  clientSecret: 'xxxx-xxxx-xxxx',
};

ReactDOM.render(
  <React.StrictMode>
    <Cuss2Connector options={options}>
      <App />
    </Cuss2Connector>
  </React.StrictMode>,
  document.getElementById('root')
);
```

Consume hooks and start subscribing as necessary.
```jsx
import { ApplicationStates } from '@elevated-libs/cuss2';
import { useCuss2 } from '@elevated-libs/cuss2-react';

export default function App() {

  const { cuss2 } = useCuss2();
  const handleStateChange = async (state: ApplicationStates) => {
    console.log('called handle state change with:' + state);
    try {
      switch (state) {
        case ApplicationStates.INITIALIZE:
          console.log('you are initialize');
          await cuss2?.requestUnavailableState();
          break;
        case ApplicationStates.UNAVAILABLE:
          console.log('you are unavailable');
          await cuss2?.requestAvailableState();
          break;
        case ApplicationStates.AVAILABLE:
          console.log('you are available');
          // Platform will take over based on Single or Multiple
          // Application Mode
          break;
        case ApplicationStates.ACTIVE:
          console.log('you have been activated');
          break;
      }
    }
    catch (e) {
      console.error(e);
    }
  };

  useEffect(() => {
    cuss2?.stateChange.subscribe((state: ApplicationStates) => handleStateChange(state));

    return () => { cuss2?.stateChange.unsubscribe(); }
  }, [cuss2]);

  return (
    // ...
  );
}
```

## The Sandbox

While you are developing your application, you can use the Elevated CUSS Sandbox against real platform responses.  Watch your application respond correctly to CUSS Events like a paper jam or a required device unavailable.
