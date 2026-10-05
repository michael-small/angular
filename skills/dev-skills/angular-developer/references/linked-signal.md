# Dependent State with `linkedSignal`

The `linkedSignal` function lets you create writable state that is intrinsically linked to some other state. It is perfect for state that needs a default value derived from an input or another signal, but can still be independently modified by the user.

If the source state changes, the `linkedSignal` resets to a new computed value.

## Basic Usage

When you only need to recompute based on a source, pass a computation function. `linkedSignal` works like `computed`, but the resulting signal is writable (you can call `.set()` or `.update()` on it).

```ts
import { Component, signal, linkedSignal } from '@angular/core';

@Component({...})
export class ShippingMethodPicker {
  protected readonly shippingOptions = signal(['Ground', 'Air', 'Sea']);

  // Defaults to the first option.
  // If shippingOptions changes, selectedOption resets to the new first option.
  protected readonly selectedOption = linkedSignal(() => this.shippingOptions()[0]);

  changeShipping(index: number) {
    // We can still manually update this signal!
    this.selectedOption.set(this.shippingOptions()[index]);
  }
}
```

## Advanced Usage: Accounting for Previous State

Sometimes, when the source state changes, you want to preserve the user's manual selection if it is still valid. To do this, use the object syntax providing `source` and `computation`.

The `computation` function receives the new value of the source, and a `previous` object containing the previous source value and the previous `linkedSignal` value.

```ts
interface ShippingMethod { id: number; name: string; }

@Component({...})
export class ShippingMethodPicker {
  protected readonly shippingOptions = signal<ShippingMethod[]>([
    {id: 0, name: 'Ground'}, {id: 1, name: 'Air'}, {id: 2, name: 'Sea'}
  ]);

  protected readonly selectedOption = linkedSignal<ShippingMethod[], ShippingMethod>({
    source: this.shippingOptions,
    computation: (newOptions, previous) => {
      // If the newly loaded options still contain the user's previously
      // selected option, keep it selected. Otherwise, reset to the first option.
      return newOptions.find(opt => opt.id === previous?.value.id) ?? newOptions[0];
    }
  });
}
```

## Advanced Usage: Customizing the set Operation

Sometimes you may want the `set` and `update` operations of a `linkedSignal` to write back to the source of truth instead of updating the `linkedSignal`'s value directly. You can customize this behavior by passing a `set` function in the options.

The custom `set` function receives two arguments:

1. The new value being set.
2. A `rawSet` function, which you can invoke to update the `linkedSignal`'s internal state directly (matching the default behavior).

_NOTE: Using `rawSet` allows you to update the `linkedSignal`'s value directly. This can be useful to prevent the computation from running, for example if it is an expensive derivation and you already know the result._

### Writing back to a source signal

Consider editing temperature in Fahrenheit, but a Celsius signal is used as its source of truth:

```typescript
const tempC = signal(0);
const tempF = linkedSignal(() => (tempC() * 9) / 5 + 32, {
  set: (valF) => tempC.set(((valF - 32) * 5) / 9),
});

console.log(tempF()); // 32

// Setting Fahrenheit updates Celsius, which reactively updates Fahrenheit
tempF.set(212);
console.log(tempC()); // 100
console.log(tempF()); // 212
```

### Updating a property inside a parent object

Another common scenario is updating a specific property inside a parent object. The parent is held in a signal, and you link to a nested property:

```typescript
interface Order {
  id: number;
  shippingMethod: string;
}

const order = signal<Order>({
  id: 42,
  shippingMethod: 'Ground',
});

const shippingMethod = linkedSignal(() => order().shippingMethod, {
  set: (newMethod) => {
    // Perform an immutable update to write the change back to the order
    order.update((currentOrder) => ({
      ...currentOrder,
      shippingMethod: newMethod,
    }));
  },
});

console.log(shippingMethod()); // 'Ground'

// Updating the shippingMethod updates the parent order object
shippingMethod.set('Air');
console.log(order()); // { id: 42, shippingMethod: 'Air' }
console.log(shippingMethod()); // 'Air'
```

### When to use `linkedSignal` vs `computed` vs `effect`

- Use `computed`: When state is **strictly** derived from other state and should never be manually updated.
- Use `linkedSignal`: When state is derived from other state, but the user **must** be able to override or manually update it.
- **Never** use `effect` to sync one piece of state to another. That is an anti-pattern. Use `computed` or `linkedSignal` instead.
