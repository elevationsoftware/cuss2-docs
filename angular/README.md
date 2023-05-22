# @elevated-libs/cuss2-angular

## About [CUSS Angular](https://www.blndspt.com/cuss-2-0/)

CUSS [(Common Use Self-Service)](https://www.iata.org/en/programs/passenger/common-use/#tab-2) Angular is a modern Typescript library facilitating rapid application development of Self-Service check-in apps, self-tagging apps and self bag-drop apps.  

This library provides an angular interface for interacting with a CUSS 2 platform. It provides services to interact with and subscribe to events from the CUSS platform and the devices connected to it.

The library and corresponding app platform also ensure backwards compatibility to legacy 1.X versions of CUSS.

Interact with a CUSS 2 API using a simple interface leveraging the asynchronicity of event driven architectures. By using the Elevated CUSS library you will get:

  - Simple device interfaces
  - Subscribable events for all CUSS states and device status

You can have CUSS 2 NOW and run a modern browser entirely without plugins or Java. Finally, your Information Security department will be able to sign off on your CUSS applications.  
 
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
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';

import { AppComponent } from './app.component';
import { Cuss2Module, ICuss2ServiceOptions } from '@elevated-libs/cuss2-angular';
import { environment } from 'src/environments/environment';
import { FormsModule } from '@angular/forms';
import { HomepageComponent } from './homepage/homepage.component';

@NgModule({
  declarations: [
    AppComponent,
    HomepageComponent,
  ],
  imports: [
    BrowserModule,
    FormsModule,
    // importing the cuss2 module
    Cuss2Module.forRoot(environment.cuss2Config as ICuss2ServiceOptions)
  ],
  exports: [
    FormsModule
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }
```
Now looking into the environments file, the following defititions should be present.

```ts
export const environment = {
  production: true,
    cuss2Config: {
      cussUrl: 'https://cuss2.herokuapp.com',
      oauthUrl: 'https://cuss2.herokuapp.com',
      clientId: 'EB',
      clientSecret: 'd69736c0-5b9b-4c6d-a5e5-14e722e9127c'
  }
};
```

Then you can choose make a `cuss-setup.service.ts` file to handle application states and defining required devices:
```ts
import { Injectable } from '@angular/core';
import { Cuss2Service, ICuss2ServiceOptions } from '@elevated-libs/cuss2-angular';
import { Cuss2, StateChange } from '@elevated-libs/cuss2';
import { environment } from 'src/environments/environment';

@Injectable({
  providedIn: 'root'
})
export class CussSetupService {
  cuss2: Cuss2;

  constructor(
    private cuss2Service: Cuss2Service,
  ) { }

  async initCuss() {
    
    try {

      this.cuss2 = await this.cuss2Service.start(environment.cuss2Config as ICuss2ServiceOptions);

      if (!this.cuss2) {
        throw new Error('CUSS2 not initialized');
      }

      this.cuss2.stateChange.subscribe(async (state: StateChange) => {
        console.log('STATE', state.current);
      });

      // Allowing the library to handled the application transitions based on a device state
      // Making required devices
      // By setting the required property on a cuss2 component, the library will autotatically move the application to unavailable 
      // any time the component broadcast an non healthy Status Code
      if (this.cuss2.barcodeReader) { this.cuss2.barcodeReader.required = true; }

      // Subscription fires when the application becomes ACTIVE
      this.cuss2.activated.subscribe(async () => {
        console.log('App is active');
        if (this.deviceOffline) {
          this.deviceOffline = false;
        }
      });

      // Subscription fires when the application becomes AVAILABLE
      this.cuss2.deactivated.subscribe(async (current: any) => {
        console.log('APPLICATION DEACTIVATED - now: ' + current);
        this.deviceOffline = true;
      });

    }
    catch(e) {
      console.error(e);
    }
  }

}

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
##  1. BarcodeReader 

In the component ts file import and use the library:
```js
import { Component, OnDestroy, OnInit } from '@angular/core';
import { BarcodeReader, BarcodeReaderService } from '@elevated-libs/cuss2-angular';

@Component({
   selector: 'app-barcode-reader', 
   templateUrl: './barcode-reader.component.html', 
   styleUrls: ['./barcode-reader.component.scss'] 
 })

export class BarcodeReaderComponent implements OnInit, OnDestroy {
   private alive: boolean = true;
   private device: BarcodeReader;
   private barcodeData: string;
   onReady$: BehaviorSubject<BarcodeReader | null> = this.barcodeReaderService.onReady;
 
   constructor(
     private barcodeReaderService: BarcodeReaderService
   ) { }
  
  ngOnInit() {
    this.onReady$.pipe(takeWhile(() => this.alive)).subscribe((barcodeReader: BarcodeReader | null) => { 
       if (barcodeReader) { 
         this.device = barcodeReader; 
         this.device.enable();
       } 
     });

     // Barcode data
    this.barcodeReaderService.onRead.pipe(takeWhile(() => this.alive)).subscribe((data: string[]) => {
      console.log(`Boarding pass BCBP data ${data}`)
    });
  }
 
  ngOnDestroy(): void { 
     this.alive = false;
     this.device?.disable(); 
  }
 
 }

```

HTML Example:

```html
<section class="container my-3" id="barcodeReader">
  <ng-container *ngIf="onReady$ | async as barcodeReader; else unavailable">
    <div  class="card">
      <div class="card-body">
        <h2>
          Barcode Reader
        </h2>
        <div class="row my-3">
          <div class="col input-group">
            <span class="input-group-text {{barcodeReader?.ready ? 'text-white bg-dark' : 'text-dark bg-light'}}">{{barcodeReader?.ready ? 'READY' : 'UNAVAILABLE'}}</span>
            <span class="input-group-text mx-3 {{barcodeReader?.pending ? 'text-white bg-dark' : 'text-dark bg-light'}}">QUERY</span>
          </div>
          <div class="col input-group">
            <span *ngIf="barcodeReader?.ready" class="input-group-text {{barcodeReader?.enabled ? 'text-white bg-dark' : 'text-dark bg-light'}}">{{barcodeReader?.enabled ? 'ENABLED' : 'DISABLED'}}</span>
            <span *ngIf="barcodeReader?.status !== 'OK'" class="input-group-text {{barcodeReader?.status}}">{{barcodeReader?.status}}</span>
          </div>
        </div>
        <div class="row my-3">
          <div class="col input-group">
            <span class="input-group-text">Last Scan:</span>
            <input class="form-control" type="text" [value]="barcodeReader?.previousData" readonly>
          </div>
        </div>
      </div>
    </div>
  </ng-container>
  <ng-template #unavailable>
    <div  class="card">
      <div class="card-body">
        <p>
          Device Unavailable
        </p>
      </div>
    </div>
  </ng-template>
</section>


```
## 2. BagPrinter

In the component ts file import and use the library:
```ts
import { Component } from '@angular/core';
import { BagTagPrinter, Cuss2 } from '@elevated-libs/cuss2';
import { BagtagPrinterService } from '@elevated-libs/cuss2-angular';
import { BehaviorSubject } from 'rxjs';

interface IBtpData {
  assets: string,
  coupon: string
}

@Component({
  selector: 'app-bag-tag-printer',
  templateUrl: './bag-tag-printer.component.html',
  styleUrls: ['./bag-tag-printer.component.scss']
})
export class BagTagPrinterComponent {

  bagTagPrinterData: IBtpData = {
  assets: 'BTT0801~J 500262=#01C0M5493450304#02C0M5493450304#03B1MA020250541=06#04B1MK200464141=06#05L0 A258250000#\n',
  coupon: 'BTP080101#01THIS IS A#02BAG TAG#03123#04456#0501#'
  };

  onReady$: BehaviorSubject<BagTagPrinter | null> = this.bagTagPrinterService.onReady;

  constructor(
    private bagTagPrinterService: BagtagPrinterService
  ) { }


async printBagTag(btp: BagTagPrinter): Promise<void> {
    const assets: string[] = this.bagTagPrinterData.assets.split(/[\r\n]/g).filter(a => !!a.trim().length)
    btp.setupAndPrintRaw(assets, this.bagTagPrinterData.coupon);
  }
}
```

HTML Example:
```html
<section class="container my-3" id="bagTagPrinter">
  <ng-container *ngIf="cussInstance$ | async as cuss2; else unavailable">
    <div *ngIf="onReady$ | async as bagTagPrinter;" class="card">
      <div class="card-body">
        <h2>
          Bag Tag Printer
        </h2>
        <div class="row my-3">
          <div class="col input-group">
            <span class="input-group-text {{bagTagPrinter?.ready ? 'text-white bg-dark' : 'text-dark bg-light'}}">{{bagTagPrinter?.ready ? 'READY' : 'UNAVAILABLE'}}</span>
            <span class="input-group-text mx-3 {{bagTagPrinter?.pending ? 'text-white bg-dark' : 'text-dark bg-light'}}">QUERY</span>
          </div>
          <div class="col input-group">
            <span *ngIf="bagTagPrinter?.status !== 'OK'" class="input-group-text {{bagTagPrinter?.status}}">{{bagTagPrinter?.status}}</span>
            <span *ngIf="bagTagPrinter?.mediaPresent" class="input-group-text">MEDIAPRESENT=true</span>
          </div>
        </div>
        <hr>
        <div class="row my-3">
          <div class="col input-group">
            <span class="input-group-text {{bagTagPrinter?.feeder?.ready ? 'text-white bg-dark' : 'text-dark bg-light'}}">{{bagTagPrinter?.feeder?.ready ? 'READY' : 'UNAVAILABLE'}}</span>
            <span class="input-group-text mx-3 {{bagTagPrinter?.feeder?.pending ? 'text-white bg-dark' : 'text-dark bg-light'}}">QUERY</span>
            <span *ngIf="bagTagPrinter?.feeder?.status !== 'OK'" class="input-group-text {{bagTagPrinter?.feeder?.status}}">{{bagTagPrinter?.feeder?.status}}</span>
          </div>
          <div class="col input-group">
            <span class="input-group-text {{bagTagPrinter?.dispenser?.ready ? 'text-white bg-dark' : 'text-dark bg-light'}}">{{bagTagPrinter?.dispenser?.ready ? 'READY' : 'UNAVAILABLE'}}</span>
            <span class="input-group-text mx-3 {{bagTagPrinter?.dispenser?.pending ? 'text-white bg-dark' : 'text-dark bg-light'}}">QUERY</span>
            <span *ngIf="bagTagPrinter?.dispenser?.status !== 'OK'" class="input-group-text {{bagTagPrinter?.dispenser?.status}}">{{bagTagPrinter?.dispenser?.status}}</span>
          </div>
        </div>
        <hr>
        <div class="row my-3">
          <div class="col input-group">
            <span class="input-group-text">Assets</span>
            <textarea class="form-control" id="bagTagPrinter_assets" [(ngModel)]="bagTagPrinterData.assets" readonly></textarea><br>
          </div>
        </div>
        <div class="row my-3">
          <div class="col input-group">
            <span class="input-group-text" for="bagTagPrinter_coupon">Coupon</span>
            <input class="form-control" id="bagTagPrinter_coupon" type="text" [(ngModel)]="bagTagPrinterData.coupon" [ngModelOptions]="{standalone: true}" readonly>
            <button class="btn btn-dark" (click)="printBagTag(bagTagPrinter)" [disabled]="!cuss2 || cuss2.state !== 'ACTIVE'">Print Bag Tag</button>
          </div>
        </div>
      </div>
    </div>
  </ng-container>
</section>

```

## 3. Card Reader
In a component ts file, import and consume the carReader service. This service is meant to provide a backwards compatible option to interface with the magstripe device. The magstripe was officially removed from the CUSS 2 specification.

```ts
import { Component, OnDestroy, OnInit } from '@angular/core';
import { CardReader, CardReaderService} from '@elevated-libs/cuss2-angular';
import { BehaviorSubject, takeWhile } from 'rxjs';

@Component({
  selector: 'app-card-reader',
  templateUrl: './card-reader.component.html',
  styleUrls: ['./card-reader.component.scss']
})
export class CardReaderComponent implements OnInit, OnDestroy {
  private alive: boolean = true;
  private device: CardReader;

  // When developing you can change timeout to determine how long the scanner is active.
  cardReaderTimeout: number = 10000;

  onReady$: BehaviorSubject<CardReader | null> = this.cardReaderService.onReady;

  constructor(
    private cardReaderService: CardReaderService
  ) { }
  
  ngOnInit(): void {
    this.onReady$.pipe(takeWhile(() => this.alive)).subscribe((cardReader: CardReader | null) => {
      if (cardReader) {
        this.device = cardReader;
        this.device?.enablePayment(true);
      }
    });
  }

   ngOnDestroy(): void {
    this.alive = false;
    this.device?.enablePayment(false);
  }
}

```
