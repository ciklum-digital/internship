# Angular Ivy for Dummies

Before you start we advise you to get acquainted with this article [Asynchronous Modules and Components in Angular Ivy](https://blog.angularindepth.com/asynchronous-modules-and-components-in-angular-ivy-1c1d79d45bd3).

## Metaprogramming

What is metaprogramming? There is a great answer on the Stack Overflow:

> Metaprogramming refers to a variety of ways a program has knowledge of itself or can manipulate itself.

Ivy is just a new world with the new rendering engine and the new compilation pipeline. Components are no more compiled into separate component factories. Let's look at the simple components and how it's compiled using Ivy and View Engine:

```ts
@Component({
  selector: 'app-button',
  template: `
    <button>Click me</button>
  `
})
export class ButtonComponent {}
```

View Engine output:

```ts
import {
  Component,
  ɵvid as viewDef,
  ɵeld as elementDef,
  ɵViewFlags as ViewFlags,
  ɵNodeFlags as NodeFlags,
  ɵted as textDef
} from '@angular/core';

export function Button_Component_0() {
  return viewDef(
    ViewFlags.None,
    [
      elementDef(
        // Index
        0,
        NodeFlags.None,
        null,
        null,
        // Child nodes count
        1,
        'button',
        [],
        null,
        null,
        null,
        null,
        null
      ),
      textDef(
        // -1 means there is no need to run change detection on this node
        -1,
        null,
        ['Click me']
      )
    ],
    null,
    null
  );
}
```

Ivy output:

```ts
import {
  Component,
  ɵComponentType as ComponentType,
  ɵComponentDef as ComponentDef,
  ɵɵdefineComponent as defineComponent,
  ɵRenderFlags as RenderFlags,
  ɵɵelementStart as elementStart,
  ɵɵelementEnd as elementEnd,
  ɵɵtext as text
} from '@angular/core';

export class ButtonComponent {
  static ngComponentDef: ComponentDef<ButtonComponent> = defineComponent({
    // Binding counts
    vars: 0,
    // Node counts
    consts: 2,
    type: ButtonComponent,
    selectors: [['app-button']],
    factory: (type?: ComponentType<ButtonComponent>) => new (type || ButtonComponent)(),
    template: (rf: RenderFlags, ctx: ButtonComponent) => {
      if (rf & RenderFlags.Create) {
        elementStart(0, 'button');
        text(1, 'Click me');
        elementEnd();
      }
    }
  });
}
```

Let's see more examples. This is how pipes are compiled:

```ts
import {
  Pipe,
  Injectable,
  ɵPipeDef as PipeDef,
  ɵɵdefinePipe as definePipe
} from '@angular/core';

@Pipe({ name: 'fibonacci' })
// Note this is needed since Angular 9+
@Injectable()
export class FibonacciPipe {
  // After compilation
  static ngPipeDef: PipeDef<FibonacciPipe> = definePipe({
    name: 'fibonacci',
    type: FibonacciPipe,
    factory: () => new FibonacciPipe()
  });
}
```

This is how services are compiled:

```ts
import {
  Injectable,
  defineInjectable,
  ɵɵInjectableDef as InjectableDef,
  ApplicationRef,
  inject
} from '@angular/core';

@Injectable({ providedIn: 'root' })
export class UserService {
  // After compilation
  static ngInjectableDef: InjectableDef<UserService> = defineInjectable({
    token: UserService,
    providedIn: 'root',
    factory: () => new UserService(inject(ApplicationRef))
  });

  constructor(private app: ApplicationRef) {}
}
```

This is how directives are compiled:

```ts
import {
  Directive,
  ɵDirectiveDef as DirectiveDef,
  ɵɵdefineDirective as defineDirective
} from '@angular/core';

@Directive({ selector: 'ngIf' })
export class NgIf {
  // After compilation
  static ngDirectiveDef: DirectiveDef<NgIf> = defineDirective({
    selectors: [['ngIf']],
    type: NgIf,
    factory: () => new NgIf()
  });
}
```

So we've got static properties and no separately compiled factories. This means that we're able to override those properties.

## Practice Metaprogramming

Let's implement the `UnsubscribeFrom` decorator that will take array name as an argument, that will contain subscriptions:

```ts
import {
  ɵComponentType as ComponentType,
  ɵDirectiveType as DirectiveType,
  ɵComponentDef as ComponentDef,
  ɵDirectiveDef as DirectiveDef
} from '@angular/core';

export function UnsubscribeFrom(arrayName: string): ClassDecorator {
  return <T = unknown>(target: Function) => {
    const type = target as ComponentType<T> | DirectiveType<T>;
    const def = getDef<T>(type);

    // Original `ngOnDestroy`
    const onDestroy: (() => void) | null = def.onDestroy;

    // `function` for capturing context
    def.onDestroy = function() {
      // Invoke original `ngOnDestroy` if it exists
      onDestroy && onDestroy.call(this);

      for (const subscription of this[arrayName]) {
        subscription.unsubscribe();
      }
    };
  };
}

function getDef<T>(
  type: ComponentType<T> | DirectiveType<T>
): ComponentDef<T> | DirectiveDef<T> {
  return (
    (type as ComponentType<T>).ngComponentDef || (type as DirectiveType<T>).ngDirectiveDef
  );
}
```

This decorator can be applied to directives and components thus we need the `getDef` function that will return their static properties. Life cycle hooks are also stored in their definitions. Let's see below how we could use this decorator:

```ts
@UnsubscribeFrom('subscriptions')
@Component({
  selector: 'app-search',
  templateUrl: './search.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class SearchComponent {
  @ViewChild('search', { static: true }) search: ElementRef<HTMLInputElement> = null!;

  subscriptions: Subscription[] = [
    fromEvent(this.search.nativeElement, 'keyup').subscribe(value => {
      console.log(value);
    })
  ];
}
```

As you see we don't need to unsubscribe manually. We've already provided the declarative way to do this.
