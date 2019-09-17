# Lazy Loading

## Dynamic Import

The dynamic import is a feature of the `esnext` module system and also of new browsers. To start using dynamic `import` - we have to edit our `tsconfig.app.json`:

```json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "module": "esnext"
  }
}
```

This propogates to third-party libraries that we don’t need to bundle, e.g. `chart.js`:

```typescript
export class ChartComponent {
  ngOnInit(): void {
    this.zone.runOutsideAngular(() => this.createChart());
  }

  private async createChart(): Promise<void> {
    const { Chart } = await import('chart.js');
    new Chart(...);
  }
}
```

## Preloading Strategies for Third-party Modules

Dynamic import is basically one of the coolest features that was integrated into the V8 and Webpack in 2017. Never requires third-party dependencies statically. This is particularly bad on mobile devices with flaky network connections, low bandwidth and limited processing power.

Imagine the situation that we want to show the user 100 pictures, we do not want to load them all at once. We'd like to defer fetching these images until a user scrolls near them. We would use, for example, the `vanilla-lazyload` package. It also requires `window.IntersectionObserver`, it's not supported everywhere so we would need a polyfill. Let's look at the resolver code below:

```typescript
@Injectable({ providedIn: 'root' })
export class LazyLoadResolver implements Resolve<unknown> {
  private packages = [
    import(/* webpackChunkName: 'vanilla-lazyload' */ 'vanilla-lazyload')
  ];

  constructor() {
    if (typeof window.IntersectionObserver === 'undefined') {
      this.packages.push(
        import(/* webpackChunkName: 'intersection-observer' */ 'intersection-observer')
      );
    }
  }

  resolve(): unknown {
    return forkJoin(this.packages).pipe(
      map(([LazyLoad]) => LazyLoad.default)
    );
  }
}
```

We will not use a dynamic import inside a component, so component will be able to access the `LazyLoad` already in the constructor itself:

```typescript
@Component({
  selector: 'app-posts',
  template: `
    <img *ngFor="let image of images" attr.data-src="{{ image }}" />
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class PostsComponent implements AfterViewInit, OnDestroy {
  images = Array.from<string>({ length: 100 }).fill('https://picsum.photos/600');

  private LazyLoad: typeof import('vanilla-lazyload') = this.route.snapshot.data.LazyLoad;

  private instance = null;

  constructor(private route: ActivatedRoute, private zone: NgZone) {}

  ngAfterViewInit(): void {
    this.zone.runOutsideAngular(() => this.createLazyLoad());
  }

  ngOnDestroy(): void {
    this.instance.destroy();
    this.instance = null;
  }

  private createLazyLoad(): void {
    this.instance = new this.LazyLoad({
      selector: 'img'
    });
  }
}
```

BTW lazy loading images is already supported in Chrome 75, so we could rewrite our resolver code as follows:

```typescript
@Injectable({ providedIn: 'root' })
export class LazyLoadResolver implements Resolve<unknown> {
  private packages = [
    import(/* webpackChunkName: 'vanilla-lazyload' */ 'vanilla-lazyload')
  ];

  constructor() {
    if (typeof window.IntersectionObserver === 'undefined') {
      this.packages.push(
        import(/* webpackChunkName: 'intersection-observer' */ 'intersection-observer')
      );
    }
  }

  resolve(): unknown {
    if ('loading' in HTMLImageElement.prototype) {
      return null;
    }

    return forkJoin(this.packages).pipe(
      map(([LazyLoad]) => LazyLoad.default)
    );
  }
}
```

And add the same check here:

```typescript
@Component({
  selector: 'app-posts',
  template: `
    <ng-template [ngIf]="lazyLoadingSupported" [ngIfElse]="unsupported">
      <img *ngFor="let image of images" src="{{ image }}" loading="lazy" />
    </ng-template>

    <ng-template #unsupported>
      <img *ngFor="let image of images" attr.data-src="{{ image }}" />
    </ng-template>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class PostsComponent implements AfterViewInit, OnDestroy {
  lazyLoadingSupported = 'loading' in HTMLImageElement.prototype;

  images = Array.from<string>({ length: 100 }).fill('https://picsum.photos/600');

  private LazyLoad: typeof import('vanilla-lazyload') | null = this.route.snapshot.data.LazyLoad;

  private instance = null;

  constructor(private route: ActivatedRoute, private zone: NgZone) {}

  ngAfterViewInit(): void {
    if (this.lazyLoadingSupported) {
      return;
    }

    this.zone.runOutsideAngular(() => this.createLazyLoad());
  }

  ngOnDestroy(): void {
    if (this.instance) {
      this.instance.destroy();
      this.instance = null;
    }
  }

  private createLazyLoad(): void {
    this.instance = new this.LazyLoad({
      selector: 'img'
    });
  }
}
```
