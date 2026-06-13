---
name: livewire-development
description: >-
  Develops reactive Livewire 4 components. Works with Filament v5 + Livewire 4.
  Activate when creating, updating, or modifying Livewire components; working with
  wire:model, wire:click, wire:loading, or any wire:* directive; adding real-time
  updates, loading states, or reactivity; debugging behavior; writing Livewire tests.
  Activate when: livewire, livewire component, wire:, wire:model, wire:click,
  reactive, alpine, dispatch event, livewire test.
---

# Livewire Development (v4)

> Works with **Livewire v4** (Filament v5). Some patterns changed from v3 — see "Livewire v4 — Changes" below.

## When to Apply

Activate this skill when:
- Creating Livewire components
- Modifying state or behavior of existing component
- Debugging reactivity or lifecycle
- Writing Livewire component tests
- Adding Alpine.js
- Working with `wire:*` directives

## Documentation

Use `search-docs` for detailed Livewire 4 patterns.

## Basic Usage

### Create components

```bash
php artisan make:livewire Posts\\CreatePost
```

### Core concepts

- State lives on the server; UI reflects it.
- Every Livewire request goes through the Laravel backend — always validate and authorize.

## Livewire v4 — Changes (versus v3)

### Self-closing tags (REQUIRED)

```blade
{{-- v3 --}}
<livewire:component>

{{-- v4 --}}
<livewire:component />
```

### wire:model — `.deep` for child events

```blade
{{-- v3 — responded to child events automatically --}}
<input wire:model="name">

{{-- v4 — needs .deep --}}
<input wire:model.deep="name">
```

### wire:transition — Native View Transitions API

```blade
{{-- v3 --}}
<div wire:transition.opacity.duration.300ms>

{{-- v4 — no modifiers --}}
<div wire:transition>
```

### config/livewire.php

```php
// v3
'layout' => 'layouts.app',

// v4
'component_layout' => 'layouts.app',
```

## Conventions (valid in v3 and v4)

- `wire:model.live` for real-time reactivity (non-live is deferred)
- Namespace: `App\Livewire` (not `App\Http\Livewire`)
- `$this->dispatch()` for events (not `emit` or `dispatchBrowserEvent`)
- Layout view: `components.layouts.app`

### Available directives

`wire:show`, `wire:transition`, `wire:cloak`, `wire:offline`, `wire:target`, `wire:loading`, `wire:dirty`.

### Alpine

- Alpine is included with Livewire — do not include separately.
- Included plugins: persist, intersect, collapse, focus.

## Best Practices

### Component Structure

- Livewire components require a single root element.
- Use `wire:loading` and `wire:dirty` for delightful loading states.

### Using Keys in Loops

```blade
@foreach ($items as $item)
    <div wire:key="item-{{ $item->id }}">
        {{ $item->name }}
    </div>
@endforeach
```

### Lifecycle Hooks

Prefer lifecycle hooks like `mount()`, `updatedFoo()` for initialization and reactive side effects:

```php
public function mount(User $user) { $this->user = $user; }
public function updatedSearch() { $this->resetPage(); }
```

## JavaScript Hooks

```js
document.addEventListener('livewire:init', function () {
    Livewire.hook('request', ({ fail }) => {
        if (fail && fail.status === 419) {
            alert('Your session expired');
        }
    });

    Livewire.hook('message.failed', (message, component) => {
        console.error(message);
    });
});
```

## Testing

```php
Livewire::test(Counter::class)
    ->assertSet('count', 0)
    ->call('increment')
    ->assertSet('count', 1)
    ->assertSee(1)
    ->assertStatus(200);
```

```php
$this->get('/posts/create')
    ->assertSeeLivewire(CreatePost::class);
```

## Common Pitfalls

- Forgetting `wire:key` in loops causes unexpected behavior
- Using `wire:model` expecting real-time (use `wire:model.live`)
- Not validating/authorizing in Livewire actions (treat as HTTP request)
- Including Alpine.js separately (already included with Livewire)
- **v4**: forgetting self-closing tags (`<livewire:component />`)
- **v4**: using `wire:model` expecting to respond to child events (needs `.deep`)
- **v4**: using modifiers in `wire:transition` (removed in v4)