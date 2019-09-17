# RxJS

## RxJS Caching Operators

All `cold` observables generate events on the first subscription and also `cold` observables are immutable (factories that always return `new Observable(...)`). `Hot` observables are streams that generate events no matter if they have subscribers or not.

Example of `hot` observable would be your Redux state. You can have 0 subscriptions to your state change but it will still generate events under the hood by applying new state.

Example of `cold` observables:

```typescript
of(20).subscribe(...);
from(promise).subscribe(...);
http.get('/api/users').subscribe(...);
fromEvent(element, 'keyup').subscribe(...);
```

`of(20)` factory returns an observable that will generate `20` only when you subscribe, same as `from|http|fromEvent`.

A typical example would be the situation when we need to load data once and with the following subscriptions do not make any requests, but immediately generate cached data:

```typescript
export class CountriesService {
  // Note we need to create stream only once and pipe it with `shareReplay(1)`
  private countries$ = this.http.get<Country[]>('/api/countries').pipe(shareReplay(1));

  constructor(private http: HttpClient) {}

  getCountries(): Observable<Country[]> {
    return this.countries$;
  }
}
```

That's where the `shareReplay` operator comes to the game. The above example is a wrapper over:

```typescript
export class CountriesService {
  private bufferSize = 1;

  private countries$ = new ReplaySubject<Country[]>(this.bufferSize);

  private refCount = 0;

  constructor(private http: HttpClient) {}

  getCountries(): Observable<Country[]> {
    if (this.refCount === this.bufferSize) {
      return this.countries$;
    }

    this.refCount++;

    return this.http.get<Country[]>('/api/countries').pipe(
      tap(countries => {
        this.countries$.next(countries);
        this.countries$.complete();
      })
    );
  }
}
```

`shareReplay` is more elegant, smaller and easier to read. This is a declarative way to buffer some chunk of data except of manually controlling cache.
