---
theme: default
---

# Angular 19 Signals

---

# 1. Why Signals?

### Simpler state management

Replaces complex RxJS boilerplate with an intuitive, easy-to-use API for local
state.

### Better performance

Enables fine-grained updates so Angular only re-renders what actually changed,
instead of checking the whole component tree.

---

# 1. Why Signals?

### Before

```ts
count = 0;

increment() {
  this.count++;
}
```

```html
{{ count }}
```

### After

```ts
count = signal(0);

increment() {
  this.count.update(v => v + 1);
}
```

```html
{{ count() }}
```

---

# 2. `signal()`

Create writable reactive state.

### Before

```ts
count = 0;
increment() { this.count++; }
reset() { this.count = 0; }
```

### After

```ts
count = signal(0);
increment() { this.count.update(v => v + 1); }
reset() { this.count.set(0); }
```

---

# 3. `computed()`

Derived state, no need to manually synchronize values (memoized and lazy).

### Before

```ts
firstName = '';
lastName = '';
get fullName() { return `${this.firstName} ${this.lastName}`; }
```

### After

```ts
firstName = signal("John");
lastName = signal("Doe");
fullName = computed(() => `${this.firstName()} ${this.lastName()}`);
```

### ❌ Avoid

- Calling `async` code and making mutations

---

# 4. `effect()`

Run side effects whenever signals change.

### Before

```ts
ngOnInit() {
  this.userService.user$.subscribe(user => { console.log(user); });
}
```

### After

```ts
user = signal<User | null>(null);
effect(() => {
  console.log(this.user());
});
```

---

# 4. `effect()`

### ✅ Use case

- Synchronize with browser APIs (`localStorage`,...)

```ts
effect(() => {
  localStorage.setItem("theme", this.theme());
});
```

- Trigger UI actions

```ts
effect(() => {
  if (this.showDialog()) {
    this.dialog.open();
  }
});
```

---

# 4. `effect()`

### ❌ Avoid

- Async subscriptions without cleanup
- Expensive calculations

- Keeping signals in sync

```ts
effect(() => {
  this.filtered.set(filter(this.items(), this.search()));
});
```

- Reading and writing the same signal

```ts
effect(() => {
  this.count.set(this.count() + 1);
});
```

---

# 5. `input()`

Signal-based replacement for `@Input`.

### Before

```ts
@Input() title!: string;
```

### After

```ts
title = input.required<string>();
```

### Usage

```ts
// use computed instead of ngOnChanges
computed(() => this.title().toUpperCase());
```

---

# 6. `output()`

Modern replacement for `@Output`.

### Before

```ts
@Output() saved = new EventEmitter<User>();

save() {
  this.saved.emit(user);
}
```

### After

```ts
saved = output<User>();

save() {
  this.saved.emit(user);
}
```

---

# 7. `model()`

Signal-powered two-way binding.

### Before

```ts
@Input() value!: string;
@Output() valueChange = new EventEmitter<string>();

notify() { this.valueChange.emit("changed-value"); }
```

```html
<app-component [(value)]="username" />
```

### After

```ts
value = model<string>("");

notify() { this.value.emit("changed-value"); }
```

```html
<app-component [(value)]="username" />
```

---

# 8. Signal Queries

Replaces decorators like `@ViewChild` and `@ContentChildren`.

### Before

```ts
@ViewChild(ButtonComponent) button!: ButtonComponent;
@ContentChildren(ItemComponent) items!: QueryList<ItemComponent>;
```

### After

```ts
button = viewChild(ButtonComponent);
items = contentChildren(ItemComponent);
```

---

# 9. `linkedSignal()`

A writable value that automatically resets when its source changes, in contrast
to `computed()` which is readonly.

### Before

```ts
selectedUser = signal(users()[0]);

effect(() => {
  selectedUser.set(users()[0]);
});
```

### After

```ts
selectedUser = linkedSignal(() => users()[0]);
```

---

# 10. `resource()`

Reactive async data fetch. Ideal when services already return `Promise`.

### Before

```ts
userId = signal(1);
user = signal<User | null>(null);
effect((onCleanup) => {
  const sub = this.httpClient
    .get<User>(`/api/users/${this.userId()}`)
    .subscribe((user) => this.user.set(user));
  onCleanup(() => sub.unsubscribe());
});
```

### After

```ts
userId = signal(1);
user = resource({
  request: () => this.userId(),
  loader: ({ request }) =>
    firstValueFrom(this.httpClient.get<User>(`/api/users/${request}`)),
});
```

---

# 10. `resource()`

### Usage

```ts
user.value(); // the loaded value (User | undefined)
user.isLoading(); // whether a request is currently in progress
user.error(); // the last error, if the request failed
```

```html
@if (user.isLoading()) {
<p>Loading...</p>
} @else if (user.error()) {
<p>Failed to load user.</p>
} @else {
<h1>{{ user.value()?.name }}</h1>
}
```

---

# 11. `rxResource()`

Use RxJS Observables as Resources. Ideal when services already return
`Observables`.

```ts
userId = signal(1);
user = rxResource({
  request: () => this.userId(),
  loader: ({ request }) => this.userService.getUser(request),
});
```

```ts
user.value(); // the loaded value (User | undefined)
user.isLoading(); // whether a request is currently in progress
user.error(); // the last error, if the request failed
```

```html
@if (user.isLoading()) {
<p>Loading...</p>
} @else if (user.error()) {
<p>Failed to load user.</p>
} @else {
<h1>{{ user.value()?.name }}</h1>
}
```

---

# 12. `afterRenderEffect()`

Run effects **after** Angular finishes rendering the DOM.

### Before

```ts
// Runs after every change detection
ngAfterViewChecked() {
    chart.update();
}
```

### After

```ts
// Runs after the view is rendered and tracked signals change
afterRenderEffect(() => {
  chart.update();
});
```

---

# 13. RxJS Interop

## `toSignal()`

### Before

```ts
user$ = service.user$;
```

```html
{{ user$ | async }}
```

### After

```ts
user = toSignal(service.user$);
```

```html
{{ user() }}
```

---

# 13. RxJS Interop

## `toObservable()`

Signal -> Observable

```ts
search = signal("");
search$ = toObservable(search);
```

---

# Summary

### Prefer

- ✅ `computed()` over `effect()` for derived values
- ✅ `input()` over `@Input`
- ✅ `output()` over `@Output`
- ✅ `model()` for two-way binding
- ✅ `linkedSignal()` for writable state tied to another signal
- ✅ `resource()`/`rxResource()` for async state
- ✅ `toSignal()` to bridge existing RxJS services

### Avoid

- ❌ Using `effect()` to synchronize state that can be expressed with
  `computed()` or `linkedSignal()`
- ❌ Mutating objects inside signals without updating the signal
- ❌ Converting every Observable to a signal—keep RxJS where it excels (complex
  async/event streams), and use signals for component-local reactive state.
