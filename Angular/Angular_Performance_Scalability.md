# Angular Performance and Scalability

> How to ensure performance and scalability in Angular applications — with techniques, code examples, and a real-world challenge story from a production healthcare portal.

---

## Table of Contents

1. [The Core Principle](#1-the-core-principle)
2. [Change Detection — The Biggest Lever](#2-change-detection--the-biggest-lever)
3. [Lazy Loading — Reduce Initial Bundle](#3-lazy-loading--reduce-initial-bundle)
4. [TrackBy in @for — Prevent Full DOM Rebuild](#4-trackby-in-for--prevent-full-dom-rebuild)
5. [Virtual Scrolling — Handle Huge Lists](#5-virtual-scrolling--handle-huge-lists)
6. [Pure Pipes — Cache Expensive Transforms](#6-pure-pipes--cache-expensive-transforms)
7. [RxJS — Prevent Memory Leaks and Excessive Calls](#7-rxjs--prevent-memory-leaks-and-excessive-calls)
8. [Bundle Optimisation](#8-bundle-optimisation)
9. [Real-World Challenge — CareFirst Member Portal](#9-real-world-challenge--carefirst-member-portal)
10. [Quick Reference — Technique Impact](#10-quick-reference--technique-impact)

---

## 1. The Core Principle

Angular performance comes down to one question: **how much work does the framework do on each user interaction?**

Every technique in this guide reduces unnecessary work:
- Fewer change detection cycles
- Smaller initial bundles downloaded by users
- Less DOM manipulation per data update
- Fewer re-renders for unchanged data

The order of attack matters. Profile first, fix the highest-impact thing, measure, repeat.

---

## 2. Change Detection — The Biggest Lever

### The problem with the default strategy

With `ChangeDetectionStrategy.Default`, Angular walks the **entire component tree** on every browser event — a button click, a scroll, a keypress. In a dashboard with 180+ components, that is hundreds of redundant checks per second.

```typescript
// Every component checked on every event
@Component({
  changeDetection: ChangeDetectionStrategy.Default
  // checks all 200 components on every single click
})
```

### Fix 1: OnPush strategy

`ChangeDetectionStrategy.OnPush` tells Angular to skip a component entirely unless:
- Its `@Input` reference changes
- An Observable or Signal it reads emits a new value
- `markForCheck()` is called manually

```typescript
@Component({
  selector: 'app-claims-list',
  changeDetection: ChangeDetectionStrategy.OnPush,  // skipped unless inputs change
  template: `...`
})
export class ClaimsListComponent {
  @Input() claims!: Claim[];    // only re-checks when this reference changes
}
```

### Fix 2: Angular 18 Signals — fine-grained reactivity

Signals go one step further than OnPush. Only the exact template expression that reads a signal re-evaluates — not even the full component template.

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <p>Count: {{ count() }}</p>         <!-- re-renders ONLY when count changes -->
    <p>Doubled: {{ doubled() }}</p>     <!-- re-renders ONLY when count changes -->
    <p>Name: {{ userName() }}</p>       <!-- re-renders ONLY when userName changes -->
  `
})
export class DashboardComponent {

  // Writable signal — set() replaces, update() transforms
  count    = signal(0);
  userName = signal('Suresh');

  // Computed signal — auto-tracks its dependencies
  doubled = computed(() => this.count() * 2);

  // Effect — runs side effects when signals change
  private logEffect = effect(() => {
    console.log('count changed:', this.count());
  });

  increment() {
    this.count.update(n => n + 1);   // only {{ count() }} and {{ doubled() }} re-render
  }
}
```

**Signals vs BehaviorSubject:**

| | BehaviorSubject | Signal |
|--|----------------|--------|
| Read | `.value` or `subscribe()` | call as function: `count()` |
| Write | `.next(value)` | `.set(value)` or `.update(fn)` |
| Derived values | `.pipe(map(...))` | `computed(() => ...)` |
| Template | `async` pipe or subscribe | call directly: `{{ count() }}` |
| Cleanup | must `unsubscribe()` | automatic |
| Change detection | triggers zone | fine-grained — only affected expressions |

### toSignal — convert Observable to Signal

```typescript
// Convert an HTTP Observable to a Signal for cleaner templates
@Component({
  template: `
    @if (users()) {
      @for (user of users()!; track user.id) {
        <p>{{ user.name }}</p>
      }
    } @else {
      <p>Loading...</p>
    }
  `
})
export class UserListComponent {
  private http = inject(HttpClient);

  // toSignal subscribes automatically and unsubscribes on destroy
  users = toSignal(this.http.get<User[]>('/api/users'));
}
```

---

## 3. Lazy Loading — Reduce Initial Bundle

### Route-level lazy loading

```typescript
// app.routes.ts
export const routes: Routes = [
  { path: '', redirectTo: 'dashboard', pathMatch: 'full' },

  // Eager — always included in main bundle
  { path: 'home', component: HomeComponent },

  // Lazy — downloaded ONLY when user visits /dashboard
  {
    path: 'dashboard',
    loadComponent: () =>
      import('./dashboard/dashboard.component')
        .then(m => m.DashboardComponent)
  },

  // Lazy feature module — downloads entire feature lazily
  {
    path: 'claims',
    loadChildren: () =>
      import('./claims/claims.routes')
        .then(m => m.CLAIMS_ROUTES)
  }
];
```

### Template-level lazy loading with @defer

`@defer` splits a component's JavaScript bundle into a separate chunk that only downloads when the trigger fires.

```html
<!-- Loads Chart.js (200 KB) ONLY when user scrolls to this section -->
@defer (on viewport) {
  <app-claims-chart [data]="chartData" />
} @placeholder {
  <!-- shown before trigger fires — contributes layout height -->
  <div style="height: 400px; background: #f5f5f5; border-radius: 8px;">
    Chart loads when visible
  </div>
} @loading (minimum 300ms) {
  <!-- shown while the bundle is downloading -->
  <app-skeleton-chart />
} @error {
  <p>Chart failed to load.</p>
}
```

**All @defer trigger types:**

| Trigger | When it fires |
|---------|--------------|
| `on viewport` | When the placeholder scrolls into view |
| `on interaction` | On first click or keypress inside the placeholder |
| `on idle` | When the browser has spare CPU time |
| `on timer(2000)` | After a 2-second delay |
| `on hover` | On mouse enter |
| `when condition` | When a signal/expression becomes truthy |

---

## 4. TrackBy in @for — Prevent Full DOM Rebuild

Without `track`, Angular has no way to identify which item changed. On any data refresh it destroys and recreates every DOM node in the list.

```html
<!-- Without track — SLOW: full DOM rebuild on every refresh -->
@for (claim of claims; track claim) {
  <app-claim-row [claim]="claim" />
}

<!-- With track claim.id — FAST: only changed items re-render -->
@for (claim of claims; track claim.id) {
  <app-claim-row [claim]="claim" />
}
```

> Angular 18 enforces `track` — the compiler gives an error if you omit it. It is mandatory, not optional.

---

## 5. Virtual Scrolling — Handle Huge Lists

Without virtual scrolling, a list of 2,400 claim rows creates 2,400 DOM nodes — each one participating in every change detection cycle, layout calculation, and scroll event.

CDK Virtual Scroll renders only the visible rows (~12–15) plus a small buffer, regardless of list size.

```typescript
// app.module.ts or component imports
import { ScrollingModule } from '@angular/cdk/scrolling';

@Component({
  standalone: true,
  imports: [ScrollingModule],
  template: `
    <cdk-virtual-scroll-viewport itemSize="56" style="height: 600px; overflow-y: auto;">
      <div *cdkVirtualFor="let claim of claims; trackBy: trackClaim"
           class="claim-row"
           style="height: 56px;">
        <span>{{ claim.date }}</span>
        <span>{{ claim.provider }}</span>
        <span>{{ claim.amount | currency:'GBP' }}</span>
      </div>
    </cdk-virtual-scroll-viewport>
  `
})
export class ClaimsListComponent {
  claims: Claim[] = [];                          // can be 10,000+ items
  trackClaim = (_: number, c: Claim) => c.id;   // stable identity
}
```

**Result:** 2,400 items → 15 DOM nodes rendered at any time.

---

## 6. Pure Pipes — Cache Expensive Transforms

Template methods are called on **every change detection cycle** — even if the value has not changed.

```typescript
// BAD: method called on every CD cycle — hundreds of times per second
// In template: {{ formatAmount(claim.amount) }}
formatAmount(amount: number): string {
  return new Intl.NumberFormat('en-GB', { style: 'currency', currency: 'GBP' }).format(amount);
}
```

```typescript
// GOOD: pure pipe — result cached, only recalculates when input changes
@Pipe({
  name: 'gbpCurrency',
  pure: true,          // default — only recalculates when input reference changes
  standalone: true
})
export class GbpCurrencyPipe implements PipeTransform {
  private formatter = new Intl.NumberFormat('en-GB', {
    style: 'currency', currency: 'GBP'
  });

  transform(amount: number): string {
    return this.formatter.format(amount);
  }
}

// In template: {{ claim.amount | gbpCurrency }}
// Called only when claim.amount reference changes
```

---

## 7. RxJS — Prevent Memory Leaks and Excessive Calls

### Unsubscribe automatically with takeUntilDestroyed

```typescript
@Component({ ... })
export class SearchComponent {
  private destroyRef = inject(DestroyRef);

  results = signal<SearchResult[]>([]);

  ngOnInit() {
    this.searchInput.valueChanges.pipe(
      debounceTime(300),         // wait 300ms after last keystroke — reduces API calls
      distinctUntilChanged(),    // skip if value is same as before
      switchMap(query =>          // cancels the previous HTTP request when new query arrives
        this.searchService.search(query).pipe(
          catchError(() => of([]))
        )
      ),
      takeUntilDestroyed(this.destroyRef)  // auto-unsubscribes when component is destroyed
    ).subscribe(results => this.results.set(results));
  }
}
```

### Key operators and what they prevent

| Operator | Problem it solves |
|----------|-----------------|
| `debounceTime(300)` | Fires 1 API call per pause, not 1 per keystroke |
| `distinctUntilChanged()` | Skips requests when value is identical to previous |
| `switchMap()` | Cancels the in-flight HTTP request when a new value arrives |
| `takeUntilDestroyed()` | Prevents memory leaks — auto-unsubscribes on component destroy |
| `shareReplay(1)` | Multiple subscribers share one HTTP call instead of firing duplicates |

### Parallel API calls with forkJoin

```typescript
// Fires all 3 API calls in parallel — waits for all to complete
forkJoin({
  claims:   this.claimsService.getClaims(),
  benefits: this.benefitsService.getBenefits(),
  profile:  this.profileService.getProfile()
}).pipe(
  takeUntilDestroyed(this.destroyRef)
).subscribe(({ claims, benefits, profile }) => {
  this.claims.set(claims);
  this.benefits.set(benefits);
  this.profile.set(profile);
});
```

---

## 8. Bundle Optimisation

### Production build flags

```bash
# Full optimised build
ng build --configuration production
# What it does:
#   --aot: pre-compiles templates (no runtime compiler shipped)
#   --optimization: tree-shakes unused code + minifies
#   --build-optimizer: removes Angular decorators from output
#   --source-map=false: removes debug source maps

# Analyse the bundle to find what's taking KB
npm install -g source-map-explorer
ng build --source-map
source-map-explorer dist/my-app/*.js
```

### Standalone components (Angular 14+)

```typescript
// BEFORE: NgModule imports ALL components/pipes in the module bundle
// Even if only 1 component uses DatePipe, it's in every component's bundle

// AFTER: standalone — only imports what this component actually uses
@Component({
  standalone: true,
  imports: [
    DatePipe,         // only these are included in this component's chunk
    CurrencyPipe,
    ReactiveFormsModule
  ],
  template: `...`
})
export class ClaimDetailComponent { }
```

### Defer third-party libraries

```typescript
// Chart.js is 200 KB — don't include in main bundle
// BAD: static import — always bundled
import Chart from 'chart.js/auto';

// GOOD: dynamic import — only bundled when function is called
async loadChart() {
  const { Chart } = await import('chart.js/auto');
  new Chart(this.canvas.nativeElement, this.chartConfig);
}
```

---

## 9. Real-World Challenge — CareFirst Member Portal

### Context

The FEP Blue member dashboard displayed claims history, benefit statements, and financial totals for BlueCross BlueShield contract holders. With some accounts having 3+ years of claims data, the dashboard was taking **8–12 seconds** to become interactive. Users were abandoning the page before it finished loading. The performance team escalated it as a critical issue.

### Root cause diagnosis

The first step was profiling with Chrome DevTools' Performance flame chart — not guessing. This showed:

- 60% of main thread time was consumed by Angular change detection
- 25% was consumed by DOM insertion of the claims table on initial load
- 15% was consumed by repeated method calls from template expressions

**Specific problems found:**

| Problem | Impact |
|---------|--------|
| Default change detection on 180+ components | Every scroll event triggered a full tree check |
| Claims table rendering 2,400+ DOM rows on load | 2,400 DOM nodes, all participating in every CD cycle |
| `formatCurrency()` method in template | Called hundreds of times per second on every CD cycle |
| 3 separate API calls blocking serially | Each waited for the previous before starting |
| Chart.js (200 KB) in the main bundle | Downloaded even on pages with no charts |
| No `track` on claim rows | Full DOM rebuild on every filter change |

### Fix sequence

We fixed in order of impact-per-hour-of-work:

**Step 1 — OnPush + Signals on the heaviest components**
```typescript
// Before: Default CD — checked on every event
@Component({ changeDetection: ChangeDetectionStrategy.Default })
export class DashboardComponent {
  claims: Claim[] = [];    // plain array — no reactivity
}

// After: OnPush + Signals — checked only when signal emits
@Component({ changeDetection: ChangeDetectionStrategy.OnPush })
export class DashboardComponent {
  claims   = signal<Claim[]>([]);
  total    = computed(() =>
    this.claims().reduce((sum, c) => sum + c.amount, 0)
  );
  // total re-computes ONLY when claims changes
}
```

**Step 2 — CDK Virtual Scroll on the claims list**
```typescript
// Before: 2,400 DOM nodes
@for (claim of claims; track claim.id) {
  <app-claim-row [claim]="claim" />
}

// After: ~15 DOM nodes rendered at any time
<cdk-virtual-scroll-viewport itemSize="56" style="height: 500px">
  <div *cdkVirtualFor="let claim of claims$">
    <app-claim-row [claim]="claim" />
  </div>
</cdk-virtual-scroll-viewport>
```

**Step 3 — @defer on charts and heavy widgets**
```typescript
// Before: Chart.js in main bundle — 200 KB always downloaded
import Chart from 'chart.js/auto';

// After: @defer — downloaded only when user scrolls to chart section
@defer (on viewport) {
  <app-claims-chart [data]="chartData()" />
} @placeholder {
  <div style="height: 320px"></div>
}
```

**Step 4 — Parallel API calls with forkJoin**
```typescript
// Before: serial — 3 × 400ms = 1,200ms minimum
this.claims$   = this.claimsService.getClaims();     // 400ms
this.benefits$ = this.benefitsService.getBenefits(); // waits for claims
this.profile$  = this.profileService.getProfile();   // waits for benefits

// After: parallel — ~400ms regardless of number of calls
forkJoin({
  claims:   this.claimsService.getClaims(),
  benefits: this.benefitsService.getBenefits(),
  profile:  this.profileService.getProfile()
}).subscribe(({ claims, benefits, profile }) => {
  this.claims.set(claims);
  this.benefits.set(benefits);
  this.profile.set(profile);
});
```

**Step 5 — Pure pipe replacing template method**
```typescript
// Before: method called on every CD cycle
{{ formatAmount(claim.amount) }}

// After: pure pipe — cached until amount reference changes
{{ claim.amount | gbpCurrency }}
```

### Results measured in production

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Time to interactive | 8–12 seconds | 1.4 seconds | **83% faster** |
| Initial bundle size | ~1.8 MB | ~760 KB | **58% smaller** |
| Change detection cycles/sec | ~1,200/sec | ~215/sec | **82% fewer** |
| DOM nodes on load (claims) | 2,400 nodes | 15 nodes | **99% fewer** |
| Page abandonment rate | ~44% | ~11% | **76% reduction** |

### Key lesson

The 50% rendering speed improvement I consistently deliver comes from these techniques applied **systematically**, not from any single clever trick. The order matters:

1. Profile first — measure before you fix
2. Fix change detection — it affects every component
3. Fix the data volume problem — virtual scroll
4. Fix the bundle size — lazy load and @defer
5. Fix the fine-grained issues — pure pipes, trackBy, RxJS operators

---

## 10. Quick Reference — Technique Impact

### Highest impact — fix first

| Technique | When to use | Expected gain |
|-----------|------------|---------------|
| OnPush + Signals | All components with complex templates | 50–80% fewer CD cycles |
| Lazy loading routes | Any route not needed on initial load | 30–60% smaller initial bundle |
| CDK Virtual Scroll | Lists with 200+ items | 90–99% fewer DOM nodes |

### Medium impact — fix next

| Technique | When to use | Expected gain |
|-----------|------------|---------------|
| @defer (on viewport) | Charts, modals, below-fold content | 20–40% smaller initial bundle |
| Pure pipes | Any method called in a template expression | Eliminates redundant calculations |
| track in @for | All lists with mutable data | Eliminates full DOM rebuild on update |
| debounceTime + switchMap | Search inputs, filter controls | Eliminates excess API calls |
| forkJoin | Multiple independent API calls | Parallel execution — saves N×latency |

### Angular 18 — performance-specific features

| Feature | What it does |
|---------|--------------|
| `signal()` | Fine-grained reactive state — only affected expressions re-render |
| `computed()` | Derived signals — memoised, only recalculates when dependencies change |
| `toSignal()` | Converts Observable to Signal — auto-subscribes and unsubscribes |
| `@defer` | Template-level lazy loading with 5 trigger types |
| `@for` with `track` | Required — Angular errors if omitted |
| Standalone components | Fine-grained tree shaking — only imported APIs in the bundle |
| `takeUntilDestroyed()` | No-boilerplate subscription cleanup |

---

## Summary

Angular performance is not about micro-optimisations — it is about understanding what work Angular does on every user interaction and systematically eliminating the unnecessary parts. The three highest-value changes in every large Angular application are:

1. **OnPush change detection + Signals** — tells Angular which components to skip entirely
2. **Lazy loading** — gives users a smaller initial download
3. **Virtual scrolling** — prevents the DOM from growing proportionally to your data size

Everything else is additive. Measure first, then apply in order of impact.

---

*Angular 18 · TypeScript 5.4 · CDK 18 · RxJS 7.8 · Based on production experience at CareFirst BCBS and KraftHeinz*
