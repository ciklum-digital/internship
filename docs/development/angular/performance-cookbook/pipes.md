# Pipes

## Pure Pipes

Use pure pipes.

Pipes can be pure and impure. What's the difference? Impure pipes has an `ngOnDestroy` lifecycle hook.

```typescript
@Pipe({ name: 'bufferToBase64' })
export class BufferToBase64Pipe implements PipeTransform {
  transform(buffer: ArrayBuffer): string {
    const rawString = String.fromCharCode(...new Uint8Array(buffer));
    return btoa(rawString);
  }
}
```

The above pipe is `pure`, pipes are `pure` by default, you don't need to specify an option in the `@Pipe` decorator. BUT, the above pipe doesn't support `ngOnDestroy`:

```typescript
export class BufferToBase64Pipe implements OnDestroy, PipeTransform {
  ngOnDestroy(): void {
    console.log('They will never see me...');
  }
}
```

You have to set the `pure` option to `false`:

```typescript
@Pipe({
  name: 'bufferToBase64',
  pure: false
})
export class BufferToBase64Pipe implements OnDestroy, PipeTransform {
  ngOnDestroy(): void {
    console.log('Aight, they gonna see me...');
  }

  transform(...) { ... }
}
```

That's how `async` pipe works, it's impure and unsubscribes from the observable thanks to `ngOnDestroy` hook.

The main difference is memoization, `pure` pipes are basically memoizers. What does it mean? When Angular performs change detection - it checks whether to invoke the `transform` method or not. Angular caches previous arguments and will compare them with incoming arguments in the future, also Angular caches results of `transform` calls.

Don't be afraid of using pipes, they are incredible performance boosters due to memoization.
