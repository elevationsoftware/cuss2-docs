# @elevated-libs/cuss2-angular

## About [CUSS Angular](https://www.blndspt.com/cuss-2-0/)

CUSS [(Common Use Self-Service)](https://en.wikipedia.org/wiki/Common-use_self-service) Angular is a modern Typescript library facilitating rapid application development of Self-Service check-in apps, self-tagging apps and self bag-drop apps.  

This library provides an angular interface for interacting with a CUSS 2.0 platform. It provides services to interact with and subscribe to events from the CUSS platform and the devices connected to it.

The library and corresponding app platform also ensure backwards compatibility to legacy 1.X versions of CUSS.

Interact with a CUSS 2.0 Restful API using a simple interface leveraging the asynchronicity of event driven architectures. By using the Elevated CUSS library you will get:

  - Simple device interfaces
  - Subscribable events for all CUSS states and device status

You can have CUSS 2.0 NOW and run a modern browser entirely without plugins or Java. Finally, your Information Security department will be able to sign off on your CUSS applications.  
 
## The Sandbox - Coming Soon

While you are developing your application, you can use the Elevated CUSS Sandbox against real platform responses. Watch your application respond correctly to CUSS Events like a paper jam or a required device unavailable.
- [CUSS Sandbox]() - Coming Soon

## The Angular Demo App

Another tool that was created was an angular demo app that uses this library. It is a simple app that shows how to make use of the lib in an Angular way. 
- [Angular Demo App](https://github.com/elevationsoftware/cuss2-angular-demo)

## Getting Started

1. Request an access token from the Elevation Software Team.  
  
2. Generate an `.npmrc` file and add the token to this file.


To install the lib run:

```sh
npm install @elevated-libs/cuss2-angular
```
In the `app.module.ts` file you can add the following code:
```js
...
import { Cuss2Module, ICuss2ServiceOptions } from '@elevated-libs/cuss2-angular';
...

...
imports: [
    Cuss2Module.forRoot(environment.cuss2Config as ICuss2ServiceOptions)
],
...
```
Then you can choose make a `cuss-setup.service.ts` file with the following code:
```js
...
import { Cuss2Service, ICuss2ServiceOptions } from '@elevated-libs/cuss2-angular';
...

...
this.cuss2 = await this.cuss2Service.start(environment.cuss2Config as ICuss2ServiceOptions);
...
```
## Usage and Examples
### Usage
___
Import the related device service from the lib

Services available for use:

- BagTagPrinterService
- BoardingPassPrinterService
- CardReaderService
- DocumentReaderService
- BarcodeReaderService
- KeypadService
- AnnouncementService

Each device service provides a set of observables to subscribe to and can be used to interact with the device.

| Device Service Observables   | Description                                                 |
| ---------------------------- | ----------------------------------------------------------- |
| onReady    | Triggers when the device is ready to use.                   |                     
| onError    | Unable to detect the device in the platform.                |                     
| onData     | Solicited and unsolicited events coming from the device.    |      
| onOk       | Emits when the platform says the device in on an OK status. |
| onRead     | Emits data read from the device. *card/barcode/document readers* only.       |
___

| Printer Only Observables         | Description                                                       |
| -------------------------------- | ----------------------------------------------------------------- |
| onPaperLow     | The printer is low on paper.                                      |                     
| onPaperJam     | Printer is triggering jam events.                                 |                     
| onPaperOut     | The printer is out of paper.                                      |      
| onPrinterError | Triggered by any error that prevents the printer from continuing. |
___

| CUSS2 Service Observables | Description                                                                                                                                                   |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| instance                  | The instance of the cuss2 object.                                                                                                                             |
| onConnect                 | Emits when the connection to cuss is established and the instance of the Cuss2 object is available.                                                           |
| onError                   | Emits when an error occurs during the connection to cuss.                                                                                                     |
| onActive                  | Emits when the application moves to active.                                                                                                                   |
| onStopped                 | Emits when the application moves to stopped.                                                                                                                  |
| onAccessibleMode          | Emits when user insert headphones into the ada pad. It will also trigger with a false value when the application is active and the user remove the headphone. |
| onAvailable               | Emits when application becomes unavailable.                                                                                                                   |  


- for all available methods and properties see [@elevated-libs/cuss2](https://github.com/elevationsoftware/elevated-lib-cuss).


### Examples
For more examples please take a look at the [documentation](https://elevationsoftware.github.io/cuss2-angular/).
____
###  1. BarcodeReader 

In the component ts file import and use the library:
```js
...
import { BarcodeReader, BarcodeReaderService } from '@elevated-libs/cuss2-angular';
...

...
onReady$: BehaviorSubject<BarcodeReader | null> = this.barcodeReaderService.onReady;

constructor(
    private barcodeReaderService: BarcodeReaderService
  ) { }

  ngOnInit(): void {
    this.onReady$.pipe(takeWhile(() => this.alive)).subscribe((barcodeReader: BarcodeReader | null) => {
      if (barcodeReader) {
        this.device = barcodeReader;
        this.device.enable();
      }
    });
  }

  ngOnDestroy(): void {
    this.alive = false;
    this.device?.disable();
  }
...
```

HTML Example:

```html
...
<ng-container *ngIf="onReady$ | async as barcodeReader; else unavailable">
...

...
<span>Last Scan:</span>
<input type="text" [value]="barcodeReader?.previousData" readonly>
...

```
### 2. BagPrinter

In the component ts file import and use the library:
```js
...
import { BagtagPrinterService, Cuss2Service } from '@elevated-libs/cuss2-angular';
...

...
bagTagPrinterData: IBtpData = {
  assets: 'setUpData' + this.cussSetupService.companyLogo,
  coupon: 'thisIsYourBoardingPassData'
  };

onReady$: BehaviorSubject<BagTagPrinter | null> = this.bagTagPrinterService.onReady;
...

...
  async printBagTag(btp: BagTagPrinter): Promise<void> {
    const assets: string[] = this.bagTagPrinterData.assets.split(/[\r\n]/g).filter(a => !!a.trim().length)
    btp.setupAndPrintRaw(assets, this.bagTagPrinterData.coupon);
  }
...
```

HTML Example:
```html
...
<ng-container *ngIf="cussInstance$ | async as cuss2; else unavailable">
<div *ngIf="onReady$ | async as bagTagPrinter; else unavailable" class="card">
...

...
<button (click)="printBagTag(bagTagPrinter)" [disabled]="!cuss2 || cuss2.state !== 'ACTIVE'">Print Bag Tag</button>
...
```
