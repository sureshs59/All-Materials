# Angular Real-Time Practice Project

A GitHub-ready Angular practice project covering performance and RxJS scenarios commonly asked in interviews.

## Topics Covered

- Parallel API calls with `forkJoin`
- Sequential API calls with `switchMap`
- Sequential API calls using `Promise` and `firstValueFrom`
- `trackBy` for list rendering performance
- `combineLatest`, `debounceTime`, `distinctUntilChanged`, `switchMap`
- `mergeMap` / `flatMap`
- Async pipe
- Pure pipe
- State management with `BehaviorSubject`
- `shareReplay` for API response caching
- `OnPush` change detection

## Tech Stack

- Angular 18 style standalone components
- TypeScript
- RxJS
- Angular Reactive Forms

## How to Run

```bash
npm install
npm start
```

Open:

```text
http://localhost:4200
```

## Project Structure

```text
src/app
 ├── components/practice-dashboard
 │   ├── practice-dashboard.component.ts
 │   └── practice-dashboard.component.html
 ├── models
 │   └── app.models.ts
 ├── pipes
 │   └── discount.pipe.ts
 ├── services
 │   └── mock-api.service.ts
 ├── state
 │   └── cart-state.service.ts
 └── app.component.ts
```

## 1. Parallel API Calls - forkJoin

Use `forkJoin` when multiple APIs are independent and you need all responses together.

```ts
forkJoin({
  user: this.api.getUser(1),
  orders: this.api.getOrders(1),
  profile: this.api.getProfile(1)
}).subscribe(result => {
  this.user = result.user;
  this.orders = result.orders;
  this.profile = result.profile;
});
```

### Interview Explanation

`forkJoin` executes independent observables in parallel and emits only after all observables complete. It is ideal for Angular `HttpClient` calls because HTTP observables complete after returning a response.

## 2. Sequential API Calls - switchMap

Use `switchMap` when the next API depends on the previous response.

```ts
this.api.getUser(1).pipe(
  switchMap(user => {
    this.user = user;
    return this.api.getOrders(user.id);
  })
).subscribe(orders => {
  this.orders = orders;
});
```

### Interview Explanation

`switchMap` is useful for dependent API calls and cancelable streams. If a new source value arrives, the previous inner observable is canceled.

## 3. Sequential Calls with Promise

```ts
async loadSequentialWithPromise(): Promise<void> {
  const user = await firstValueFrom(this.api.getUser(1));
  const orders = await firstValueFrom(this.api.getOrders(user.id));
}
```

### Observable vs Promise

| Observable | Promise |
|---|---|
| Can emit multiple values | Emits one value |
| Cancelable via unsubscribe | Not cancelable by default |
| Supports RxJS operators | Limited chaining |
| Best for Angular streams and HTTP | Good for simple async flows |

## 4. trackBy Example

```html
<li *ngFor="let user of users; trackBy: trackByUserId">
  {{ user.name }}
</li>
```

```ts
trackByUserId(index: number, user: User): number {
  return user.id;
}
```

### Interview Explanation

`trackBy` improves performance by helping Angular identify which list items changed. Angular can reuse DOM elements instead of recreating the full list.

## 5. combineLatest + debounceTime + distinctUntilChanged + switchMap

Use this pattern for real-time search with filters.

```ts
products$ = combineLatest([
  this.searchControl.valueChanges.pipe(
    debounceTime(300),
    distinctUntilChanged()
  ),
  this.categoryControl.valueChanges
]).pipe(
  switchMap(([searchText, category]) =>
    this.api.searchProducts(searchText, category)
  )
);
```

### When to Use

- `combineLatest`: when multiple inputs decide one result
- `debounceTime`: wait before calling API while user types
- `distinctUntilChanged`: avoid duplicate API calls
- `switchMap`: cancel previous API request and use latest

## 6. mergeMap / flatMap Example

In RxJS, `flatMap` is an alias style concept for `mergeMap`.

```ts
of(1, 2).pipe(
  mergeMap(userId => this.api.getUser(userId))
).subscribe(user => console.log(user));
```

### When to Use

Use `mergeMap` when all requests should complete and you do not want to cancel previous requests.

## 7. Async Pipe

```ts
cartItems$ = this.cartState.cartItems$;
```

```html
<div *ngIf="cartItems$ | async as cartItems">
  {{ cartItems.length }}
</div>
```

### Interview Explanation

Async pipe automatically subscribes and unsubscribes. It helps avoid memory leaks and keeps templates reactive.

## 8. Pure Pipe

```ts
@Pipe({ name: 'discount', standalone: true, pure: true })
export class DiscountPipe implements PipeTransform {
  transform(price: number, discountPercent: number): number {
    return price - (price * discountPercent / 100);
  }
}
```

```html
{{ product.price | discount:10 }}
```

### Interview Explanation

Pure pipes execute only when the input reference changes. This improves performance compared with calling methods directly from templates.

## 9. State Management with BehaviorSubject

```ts
private cartItemsSubject = new BehaviorSubject<Product[]>([]);
cartItems$ = this.cartItemsSubject.asObservable();

addItem(item: Product): void {
  const currentItems = this.cartItemsSubject.value;
  this.cartItemsSubject.next([...currentItems, item]);
}
```

### Interview Explanation

For small and medium Angular apps, services with `BehaviorSubject` are a practical state management approach. `BehaviorSubject` stores the latest value and immediately emits it to new subscribers.

## 10. shareReplay API Cache

```ts
cachedUser$ = this.api.getUser(1).pipe(
  shareReplay(1)
);
```

### Interview Explanation

`shareReplay(1)` shares the latest emitted value with multiple subscribers and avoids duplicate API calls for the same data.

## 11. Performance Checklist

- Use `OnPush` change detection
- Use `trackBy` for large lists
- Use lazy loading for feature modules
- Use async pipe instead of manual subscriptions
- Use `debounceTime` for search boxes
- Use `distinctUntilChanged` to avoid duplicate calls
- Use `switchMap` for latest-request-only scenarios
- Use `forkJoin` for independent parallel API calls
- Use pure pipes for template transformation
- Avoid nested subscriptions
- Use `shareReplay` carefully for API caching

## Interview Summary

For Angular real-time scenarios, I use `forkJoin` for parallel APIs, `switchMap` for dependent or cancelable APIs, `combineLatest` when multiple streams decide the result, `debounceTime` and `distinctUntilChanged` for optimized search, `trackBy` for list performance, async pipe for subscription cleanup, pure pipes for efficient transformations, and services with `BehaviorSubject` for feature-level state management.
