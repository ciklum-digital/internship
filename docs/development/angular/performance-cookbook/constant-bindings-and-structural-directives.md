# Constant Bindings and Structural Directives

## Constant Bindings

Angular allows you to bind constant input parameters. Even if you're not familiar with Angular material - let's look at this example:

```html
<button mat-button color="primary">Text</button>
```

Where `color` is an input parameter. Such approach is an alternative to:

```html
<button mat-button [color]="'primary'">Text</button>
```

BUT, using square brackets tells Angular that this binding is dynamic and should be re-checked every change detection.

Omitting square brackets is called `one-time string initialization`.

You should omit the brackets when all of the following are true:

* The target property accepts a string value.
* The string is a fixed value that you can bake into the template.
* This initial value never changes.

If you have to bind a variable - of course you would use square brackets. Alternatively there is a possibility to get input parameters already in the constructor, it is permissible by using an `Attribute` decorator:

```typescript
import { Component, Attribute } from '@angular/core';
 
@Component({
  selector: 'app-button',
  template: `
    <button [disabled]="disabled">{{ text }}</button>
  `
})
export class ButtonComponent {
  disabled = false;

  constructor(
    @Attribute('text') public text: string,
    @Attribute('disabled') disabled: string | null
  ) {
    this.setDisabled(disabled);
  }

  private setDisabled(disabled: string | null): void {
    if (disabled === 'true' || disabled === 'false') {
      this.disabled = JSON.parse(disabled);
    }
  }
}
```

Next steps:

```html
<app-button disabled="true" text="Some text here"></app-button>
```

NOTICE, that everything is a string if we use an attribute decorator, so if we pass a `disabled="true"` attribute, then we get a string `"true"`, that's why we use `JSON.parse`.

## Custom Structural Directives

Directives by default behave as `OnPush` components, they are re-checked only in cases if the reference to input binding is changed.

Problem: imagine a situation when you need to hide a component when the screen width is less than 960 pixels, how would you do it wit CSS? Something like that I guess:

```css
@media screen and (max-width: 960px) {
  app-header {
    display: none;
  }
}
```

No Bob, that's not a good aproach. Our component still lives in the heap and Angular checks it, let's change do `ngIf`:

```html
<app-header *ngIf="shown"></app-header>
```

In the component:

```typescript
export class TopMenuComponent implements OnDestroy {
  hidden = false;

  private mq: MediaQueryList = null;

  constructor() {
    this.setupMediaQueryListener();
  }

  ngOnDestroy(): void {
    this.mq.removeListener(this.listener);
  }

  private setupMediaQueryListener(): void {
    this.mq = matchMedia('(max-width: 960px)');
    this.mq.addListener(this.listener);
    this.listener(this.mq);
  }

  private listener = ({ matches }: MediaQueryList | MediaQueryListEvent): void => {
    this.hidden = matches;
  };
}
```

This is suitable for a single case, and what will happen if you need to reuse this logic? Duplicate the homogeneous code everywhere?

The answer is - NO. Let's be more declarative and create our custom structural directive:

```typescript
const selector = `
  [hide.lt-sm], [hide.lt-md]
`;

enum Query {
  LtSm = 'screen and (max-width: 599px)',
  LtMd = 'screen and (max-width: 959px)'
}

@Directive({ selector })
export class HideDirective<C = null> implements OnInit {
  /**
   * `screen and (max-width: 599px)`
   */
  @Input('hide.lt-sm') hideLtSm: true | null = null;

  /**
   * `screen and (max-width: 959px)`
   */
  @Input('hide.lt-md') hideLtMd: true | null = null;

  private viewRef: EmbeddedViewRef<C> | null = null;

  constructor(
    private viewContainerRef: ViewContainerRef,
    private templateRef: TemplateRef<C>
  ) {}

  ngOnInit(): void {
    this.observe();
  }

  private observe(): void {
    const query = this.getQuery();
    const mq = matchMedia(query);

    const listener = ({ matches }: MediaQueryList | MediaQueryListEvent) => {
      if (matches) {
        this.destroy();
      } else {
        this.create();
      }
    };

    listener(mq);
    // We can add this event inside Angular's zone as it's triggered only 1-2 times
    mq.addListener(listener);
  }

  private getQuery() {
    const keys = Object.keys(this);

    for (const key of keys) {
      // Means that key is `hideLtSm` and equals `true`
      if (key.startsWith('hide') && this[key]) {
        // We slice and get from `hideLtSm => LtSm`
        const queryKey = key.slice(4, key.length);
        // Then we just get a value from hash map
        return Query[queryKey];
      }
    }

    throw new Error('Please, provide a valid binding for the "HideDirective"!');
  }

  private create(): void {
    this.viewRef = this.viewContainerRef.createEmbeddedView(this.templateRef);
    // Calling `markForCheck` to make sure we will run the change detection when the
    // `HideDirective` is inside a `ChangeDetectionStrategy.OnPush` component
    this.viewRef.markForCheck();
  }

  private destroy(): void {
    this.viewContainerRef.clear();

    if (this.viewRef) {
      this.viewRef.destroy();
      this.viewRef = null;
    }
  }
}
```

How to use this directive?

```html
<app-header *hideLtMd="true"></app-header>
```

So our `app-header` became dynamic as it's wrapped into `ng-template` and if the screen width is less than 960px - this component is destroyed and memory is released. Profit?
