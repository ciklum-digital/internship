# Using zone.js

## Capabilities of zone.js

Use `zone.js` capabilities when you gonna work with some browser asynchronous API or external library.

ATTENTION! This is very important. Imagine a simple application that uses `chart.js` library, `chart.js` adds a lot of event listeners like `mousemove` etc. This means that your application will be re-rendered XXX times if you just move your mouse over the chart:

```typescript
import { Component, OnInit, ViewChild, ElementRef } from '@angular/core';

import * as Chart from 'chart.js';

@Component({
  selector: 'app-line-chart',
  template: `
    <canvas #canvas></canvas>
  `
})
export class LineChartComponent implements OnInit {
  @ViewChild('canvas') canvas: ElementRef<HTMLCanvasElement> = null!;

  // Angular 8+
  @ViewChild('canvas', { static: true }) canvas: ElementRef<HTMLCanvasElement> = null!;

  private options = {
    ...
  };

  ngOnInit(): void {
    const ctx = this.canvas.nativeElement.getContext('2d');
    new Chart(ctx, this.options); // BAD!
  }
}
```

Let's you the `NgZone` class:

```typescript
import { Component, OnInit, ViewChild, ElementRef, NgZone } from '@angular/core';

import * as Chart from 'chart.js';

@Component({
  selector: 'app-line-chart',
  template: `
    <canvas #canvas></canvas>
  `
})
export class LineChartComponent implements OnInit {
  @ViewChild('canvas') canvas: ElementRef<HTMLCanvasElement> = null!;

  // Angular 8+
  @ViewChild('canvas', { static: true }) canvas: ElementRef<HTMLCanvasElement> = null!;

  private options = {
    ...
  };

  constructor(private zone: NgZone) {}

  ngOnInit(): void {
    const ctx = this.canvas.nativeElement.getContext('2d');

    this.zone.runOutsideAngular(() => {
      new Chart(ctx, this.options); // GOOD!
    });
  }
}
```

`runOutsideAngular` means to run a callback in the parent zone, so Angular will not be notified about asynchronous tasks and `ApplicationRef.tick` will not be run.

## requestAnimationFrame

`requestAnimationFrame` is monkey patched by `zone.js`. Using this function - you also let Angular know that an asynchronous event is running somewhere, when Angular gets to know - it invokes `ApplicationRef.tick()`.

```typescript
import {
  Component,
  ChangeDetectionStrategy,
  NgZone,
  OnDestroy,
  ElementRef
} from '@angular/core';

@Component({
  selector: 'app-progress',
  template: '',
  styles: [
    `
      :host {
        display: grid;
        height: 5px;
        background: red;
      }
    `
  ],
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ProgressComponent implements OnDestroy {
  private width = 0;

  private requestId: number;

  constructor(zone: NgZone, private host: ElementRef<HTMLElement>) {
    // BAD
    this.requestId = requestAnimationFrame(this.animate);

    // GOOD
    zone.runOutsideAngular(() => {
      this.requestId = requestAnimationFrame(this.animate);
    });
  }

  ngOnDestroy(): void {
    this.cancel();
  }

  private animate = (): void => {
    if (this.width > 100) {
      return this.cancel();
    }

    this.host.nativeElement.style.width = `${this.width++}%`;
    this.requestId = requestAnimationFrame(this.animate);
  };

  private cancel(): void {
    cancelAnimationFrame(this.requestId);
  }
}
```
