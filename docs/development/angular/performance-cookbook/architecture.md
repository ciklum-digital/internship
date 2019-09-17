# Architecture

## Type Safe Architectures

TypeScript and Angular allow you to maximize the typing of the architecture thanks to the options of both compilers, which makes it possible to avoid a large number of production errors.

The TypeScript compiler currently has nine strict compiler flags that can be used as tools to improve code quality and, to varying degrees, increase the soundness of TypeScript’s type system:

* **noImplicitAny**
* **noImplicitThis**
* **noUnusedLocals**
* **noUnusedParameters**
* **alwaysStrict**
* **strictBindCallApply**
* **strictNullChecks**
* **strictFunctionTypes**
* **strictPropertyInitialization**

These options help to imperatively interact with the compiler, while the compiler will constantly tell you where an error may occur, you will agree and correct the code, or convince the compiler otherwise.

Using these options this code will not work anymore:

```typescript
@Component({
  selector: 'app-button',
  templateUrl: './button.component.html'
})
export class ButtonComponent {
  @Input() disabled: boolean;
 
  @Input() text: string;
}
```

You will get - `Property 'disabled' has no initializer and is not definitely assigned in the constructor`. And it's definitely true, you just declared property and marked it as "may-existing". TypeScript compiler forces you to mark those properties as `optional` or initialize them with some value. You can also use a `non-null assertion operator` that tells the TypeScript compiler that this property will be 100% initialized later:

```typescript
@Input() text: string = null!;
```

`!` operator ensures compiler that this property exists and will be re-assigned later with `string` value, `disabled` can be marked as optional:

```typescript
@Input() disabled?: boolean;
```

This means that this property may not exist.

This is how your `tsconfig.json` should look like:

```json
{
  "compilerOptions": {
    "strict": true,
    "importHelpers": true,
    "noUnusedLocals": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "suppressImplicitAnyIndexErrors": true
  }
}
```

Angular's template compiler can also be configured via `angularCompilerOptions`, your `tsconfig.app.json` should have following properties:

```json
{
  "extends": "./tsconfig.json",
  "include": [
    "src/**/*.ts"
  ],
  "exclude": [
    "src/test.ts",
    "src/**/*.spec.ts"
  ],
  "angularCompilerOptions": {
    "trace": true,
    "fullTemplateTypeCheck": true,
    "strictInjectionParameters": true
  }
}
```

`fullTemplateTypeCheck` - tells the compiler to enable the binding expression validation phase of the template compiler which uses TypeScript to validate binding expressions (will be **true** by default in Angular 9 or 10).

`strictInjectionParameters` - tells the compiler to report an error for a parameter supplied whose injection type cannot be determined (will be **true** by default in Angular 9 or 10).

## Code Tuning Strategies

> "Performance is only loosely related to code speed. To the extent that you work on your code’s speed, you’re not working on other quality characteristics. Be wary of sacrificing other characteristics to make your code faster. Your work on speed might hurt overall performance rather than help it." - Steve McConnell, Code Complete

That's true. But when to tune?

> Use a high-quality design. Make the program right. Make it modular and easily modifiable so that it’s easy to work on later. When it’s complete and correct, check the performance. If the program lumbers, make it fast and small. Don’t optimize until you know you need to.

Use all the right design methods initially. Use the capabilities of `zone.js` to work with third-party modules. Mark your components as `OnPush` - INITIALLY, and not in the case when you've got 1000 components and you do not know what exactly broke.
