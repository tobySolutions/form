---
id: form-validation
title: Form and Field Validation
---

At the core of TanStack Form's functionalities is the concept of validation. TanStack Form makes validation highly customizable:

- You can control when to perform the validation (on change, on input, on blur, on submit...)
- Validation rules can be defined at the field level or at the form level
- Validation can be synchronous or asynchronous (for example, as a result of an API call)

## When is validation performed?

It's up to you! The `<Field />` component accepts some callbacks as props such as `onChange` or `onBlur`. Those callbacks are passed the current value of the field, as well as the fieldAPI object, so that you can perform the validation. If you find a validation error, simply return the error message as string and it will be available in `field.state.meta.errors`.

Here is an example:

```ts
this.#form.field(
  {
    name: 'age',
    validators: {
      onChange: ({ value }: { value: number }) =>
        value < 13 ? 'You must be 13 and above to be an employee' : undefined,
    },
  },
  (field) => {
    return html`
      <label for=${field.name}>Age:</label>
      <input
        id=${field.name}
        name=${field.name}
        .value=${field.state.value}
        type="number"
        @input=${(e: Event) => {
          const target = e.target as HTMLInputElement
          field.handleChange(target.valueAsNumber)
        }}
      />
      ${field.state.meta.isTouched && field.state.meta.errors.length
        ? html`${repeat(
            field.state.meta.errors,
            (__, idx) => idx,
            (error) => {
              return html`<div class="container">${error}</div>`
            },
          )}`
        : nothing}
    `
  },
)
```

In the example above, the validation is done at each keystroke (`onChange`). If, instead, we wanted the validation to be done when the field is blurred, we would change the code above like so:

```ts
this.#form.field(
  {
    name: 'age',
    validators: {
      onBlur: ({ value }: { value: number }) =>
        value < 13 ? 'You must be 13 and above to be an employee' : undefined,
    },
  },
  (field) => {
    return html`
      <label for=${field.name}>Age:</label>
      <input
        id=${field.name}
        name=${field.name}
        .value=${field.state.value}
        type="number"
        @blur=${() => field.handleBlur()}
        @input=${(e: Event) => {
          const target = e.target as HTMLInputElement
          field.handleChange(target.valueAsNumber)
        }}
      />
      ${field.state.meta.isTouched && field.state.meta.errors.length
        ? html`${repeat(
            field.state.meta.errors,
            (__, idx) => idx,
            (error) => {
              return html`<div class="container">${error}</div>`
            },
          )}`
        : nothing}
    `
  },
)
```

So you can control when the validation is done by implementing the desired callback. You can even perform different pieces of validation at different times:

```ts
this.#form.field(
  {
    name: 'age',
    validators: {
      onChange: ({ value }) =>
        value < 13 ? 'You must be 13 and above to be an employee' : undefined,
      onBlur: ({ value }: { value: number }) =>
        value < 13 ? 'You must be 13 and above to be an employee' : undefined,
    },
  },
  (field) => {
    return html`
      <label for=${field.name}>Age:</label>
      <input
        id=${field.name}
        name=${field.name}
        .value=${field.state.value}
        type="number"
        @blur=${() => field.handleBlur()}
        @input=${(e: Event) => {
          const target = e.target as HTMLInputElement
          field.handleChange(target.valueAsNumber)
        }}
      />
      ${field.state.meta.isTouched && field.state.meta.errors.length
        ? html`${repeat(
            field.state.meta.errors,
            (__, idx) => idx,
            (error) => {
              return html`<div class="container">${error}</div>`
            },
          )}`
        : nothing}
    `
  },
)
```

In the example above, we are validating different things on the same field at different times (at each keystroke and when blurring the field). Since `field.state.meta.errors` is an array, all the relevant errors at a given time are displayed. You can also use `field.state.meta.errorMap` to get errors based on _when_ the validation was done (onChange, onBlur etc...). More info about displaying errors below.

## Displaying Errors

Once you have your validation in place, you can map the errors from an array to be displayed in your UI:

```ts
this.#form.field(
  {
    name: 'age',
    validators: {
      onChange: ({ value }) =>
        value < 13 ? 'You must be 13 and above to be an employee' : undefined,
    },
  },
  (field) => {
    return html`
      // ...
      ${field.state.meta.isTouched && field.state.meta.errors.length
        ? html`${repeat(
            field.state.meta.errors,
            (__, idx) => idx,
            (error) => {
              return html`<div class="container">${error}</div>`
            },
          )}`
        : nothing}
    `
  },
)
```

Or use the `errorMap` property to access the specific error you're looking for:

```ts
this.#form.field(
  {
    name: 'age',
    validators: {
      onChange: ({ value }) =>
        value < 13 ? 'You must be 13 and above to be an employee' : undefined,
    },
  },
  (field) => {
    return html`
      // ...
      ${field.state.meta.isTouched && field.state.meta.errorMap['onChange']
        ? html`<em role="alert">${field.state.meta.errorMap['onChange']}</em>`
        : nothing}
    `
  },
)
```

## Validation at field level vs at form level

As shown above, each `<Field>` accepts its own validation rules via the `onChange`, `onBlur` etc... callbacks. It is also possible to define validation rules at the form level (as opposed to field by field) by passing similar callbacks to the `useForm()` hook.

Example:

```tsx
export class App extends LitElement {
  #form = new TanStackFormController(this, {
    defaultValues: {
      age: 0,
    },
    validators: {
      onChange: ({ value }: { value: any }) => {
        return value.age < 13
          ? 'You must be 13 and above to be an employee'
          : undefined
      },
    },
  })

  render() {
    const formErrorMap = this.#form.api.state.errorMap

    return html`
      // ...
      ${formErrorMap.onChange
        ? html`<em
            >There was an error on the form: ${formErrorMap.onChange}</em
          >`
        : nothing}
      //...
    `
  }
}
```

### Setting field-level errors from the form's validators

You can set errors on the fields from the form's validators. One common use case for this is validating all the fields on submit by calling a single API endpoint in the form's `onSubmitAsync` validator.

```tsx
export class TanStackFormDemo extends LitElement {
  #form = new TanStackFormController(this, {
    defaultValues: {
      age: 0,
    },
    validators: {
      onSubmitAsync: async ({ value }: { value: any }) => {
        const isOlderThan13 = await this.verifyAgeOnServer(value.age)
        if (!isOlderThan13) {
          return {
            form: 'Invalid data',
            fields: {
              age: 'You must be 13 and above to be an employee',
            },
          }
        }
        return null
      },
    },
  })

  async verifyAgeOnServer(age: number) {
    return new Promise<boolean>((resolve) => {
      setTimeout(() => {
        resolve(age >= 13)
      }, 1000)
    })
  }

  render() {
    const formErrorMap = this.#form.api.state.errorMap

    return html`
      <div>
        <form
          @submit=${(e: Event) => {
            e.preventDefault()
            e.stopPropagation()
            void this.#form.api.handleSubmit()
          }}
        >
          ${this.#form.field(
            { name: 'age' },
            (field) => html`
              <label for=${field.name}>Age:</label>
              <input
                id=${field.name}
                name=${field.name}
                .value=${field.state.value}
                type="number"
                @input=${(e: Event) => {
                  const target = e.target as HTMLInputElement
                  field.handleChange(target.valueAsNumber)
                }}
              />
              ${field.state.meta.errors
                ? html`<em role="alert"
                    >${field.state.meta.errors.join(', ')}</em
                  >`
                : nothing}
            `,
          )}
          ${formErrorMap.onSubmit
            ? html`<div>
                <em
                  >There was an error on the form: ${formErrorMap.onSubmit}</em
                >
              </div>`
            : nothing}

          <button type="submit">Submit</button>
        </form>
      </div>
    `
  }
}
```

> Something worth mentioning is that if you have a form validation function that returns an error, that error may be overwritten by the field-specific validation.
>
> This means that:
>
> ```js
> export class TanStackFormValidationDemo extends LitElement {
>   #form = new TanStackFormController(this, {
>     defaultValues: { age: 0 },
>     validators: {
>       onChange: ({ value }) => ({
>         fields: { age: value.age < 12 ? 'Too young!' : undefined },
>       }),
>     },
>   })
>
>   render() {
>     return html`
>       <form @submit=${(e) => e.preventDefault()}>
>         ${this.#form.field(
>           {
>             name: 'age',
>             validators: {
>               onChange: ({ value }) =>
>                 value % 2 === 0 ? 'Must be odd!' : undefined,
>             },
>           },
>           (field) => html`
>             <input
>               .value=${field.state.value}
>               type="number"
>               @input=${(e) => field.handleChange(e.target.valueAsNumber)}
>             />
>             ${field.state.meta.errors
>               ? html`<em>${field.state.meta.errors.join(', ')}</em>`
>               : ''}
>           `,
>         )}
>       </form>
>     `
>   }
> }
> ```
>
> Will only show `'Must be odd!` even if the 'Too young!' error is returned by the form-level validation.

## Asynchronous Functional Validation

While we suspect most validations will be synchronous, there are many instances where a network call or some other async operation would be useful to validate against.

To do this, we have dedicated `onChangeAsync`, `onBlurAsync`, and other methods that can be used to validate against:

```ts
this.#form.field(
  {
    name: 'age',
    validators: {
      onChangeAsync: async ({ value }: { value: number }) => {
        await new Promise((resolve) => setTimeout(resolve, 1000))
        return value < 13 ? 'You must be 13 to be an employee' : undefined
      },
    },
  },
  (field) => {
    return html`
      <label for=${field.name}>Age:</label>
      <input
        id=${field.name}
        name=${field.name}
        .value=${field.state.value}
        type="number"
        @input=${(e: Event) => {
          const target = e.target as HTMLInputElement
          field.handleChange(target.valueAsNumber)
        }}
      />
      ${field.state.meta.errors
        ? html`<em role="alert">${field.state.meta.errors.join(', ')}</em>`
        : nothing}
    `
  },
)
```

Synchronous and Asynchronous validations can coexist. For example, it is possible to define both `onBlur` and `onBlurAsync` on the same field:

```ts
this.#form.field(
  {
    name: 'age',
    validators: {
      onBlur: ({ value }) =>
        value < 13 ? 'You must be 13 to be an employee' : undefined,
      onBlurAsync: async ({ value }: { value: number }) => {
        const currentAge = await fetchCurrentAgeOnProfile()
        return value < 13 ? 'You must be 13 to be an employee' : undefined
      },
    },
  },
  (field) => {
    return html`
      <label for=${field.name}>Age:</label>
      <input
        id=${field.name}
        name=${field.name}
        .value=${field.state.value}
        @blur=${() => field.handleBlur()}
        type="number"
        @input=${(e: Event) => {
          const target = e.target as HTMLInputElement
          field.handleChange(target.valueAsNumber)
        }}
      />
      ${field.state.meta.errors
        ? html`<em role="alert">${field.state.meta.errors.join(', ')}</em>`
        : nothing}
    `
  },
)
```

The synchronous validation method (`onBlur`) is run first and the asynchronous method (`onBlurAsync`) is only run if the synchronous one (`onBlur`) succeeds. To change this behaviour, set the `asyncAlways` option to `true`, and the async method will be run regardless of the result of the sync method.

### Built-in Debouncing

While async calls are the way to go when validating against the database, running a network request on every keystroke is a good way to DDOS your database.

Instead, we enable an easy method for debouncing your `async` calls by adding a single property:

```tsx
<form.Field
  name="age"
  asyncDebounceMs={500}
  validators={{
    onChangeAsync: async ({ value }) => {
      // ...
    },
  }}
  children={(field) => {
    return <>{/* ... */}</>
  }}
/>
```

This will debounce every async call with a 500ms delay. You can even override this property on a per-validation property:

```tsx
<form.Field
  name="age"
  asyncDebounceMs={500}
  validators={{
    onChangeAsyncDebounceMs: 1500,
    onChangeAsync: async ({ value }) => {
      // ...
    },
    onBlurAsync: async ({ value }) => {
      // ...
    },
  }}
  children={(field) => {
    return <>{/* ... */}</>
  }}
/>
```

> This will run `onChangeAsync` every 1500ms while `onBlurAsync` will run every 500ms.

## Adapter-Based Validation (Zod, Yup, Valibot)

While functions provide more flexibility and customization over your validation, they can be a bit verbose. To help solve this, there are libraries like [Valibot](https://valibot.dev/), [Yup](https://github.com/jquense/yup), and [Zod](https://zod.dev/) that provide schema-based validation to make shorthand and type-strict validation substantially easier.

Luckily, we support all of these libraries through official adapters:

```bash
$ npm install @tanstack/zod-form-adapter zod
# or
$ npm install @tanstack/yup-form-adapter yup
# or
$ npm install @tanstack/valibot-form-adapter valibot
```

Once done, we can add the adapter to the `validator` property on the form or field:

```tsx
import { zodValidator } from '@tanstack/zod-form-adapter'
import { z } from 'zod'

// ...

const form = useForm({
  // Either add the validator here or on `Field`
  validatorAdapter: zodValidator(),
  // ...
})

<form.Field
  name="age"
  validatorAdapter={zodValidator()}
  validators={{
    onChange: z.number().gte(13, 'You must be 13 to make an account'),
  }}
  children={(field) => {
    return <>{/* ... */}</>
  }}
/>
```

These adapters also support async operations using the proper property names:

```tsx
<form.Field
  name="age"
  validators={{
    onChange: z.number().gte(13, 'You must be 13 to make an account'),
    onChangeAsyncDebounceMs: 500,
    onChangeAsync: z.number().refine(
      async (value) => {
        const currentAge = await fetchCurrentAgeOnProfile()
        return value >= currentAge
      },
      {
        message: 'You can only increase the age',
      },
    ),
  }}
  children={(field) => {
    return <>{/* ... */}</>
  }}
/>
```

### Form Level Adapter Validation

You can also use the adapter at the form level:

```tsx
import { zodValidator } from '@tanstack/zod-form-adapter'
import { z } from 'zod'

// ...

const formSchema = z.object({
  age: z.number().gte(13, 'You must be 13 to make an account'),
})

const form = useForm({
  validatorAdapter: zodValidator(),
  validators: {
    onChange: formSchema,
  },
})
```

If you use the adapter at the form level, it will pass the validation to the fields of the same name.

This means that:

```tsx
<form.Field
  name="age"
  children={(field) => {
    return <>{/* ... */}</>
  }}
/>
```

Will still display the error message from the form-level validation.

## Preventing invalid forms from being submitted

The `onChange`, `onBlur` etc... callbacks are also run when the form is submitted and the submission is blocked if the form is invalid.

The form state object has a `canSubmit` flag that is false when any field is invalid and the form has been touched (`canSubmit` is true until the form has been touched, even if some fields are "technically" invalid based on their `onChange`/`onBlur` props).

You can subscribe to it via `form.Subscribe` and use the value in order to, for example, disable the submit button when the form is invalid (in practice, disabled buttons are not accessible, use `aria-disabled` instead).

```tsx
const form = useForm(/* ... */)

return (
  /* ... */

  // Dynamic submit button
  <form.Subscribe
    selector={(state) => [state.canSubmit, state.isSubmitting]}
    children={([canSubmit, isSubmitting]) => (
      <button type="submit" disabled={!canSubmit}>
        {isSubmitting ? '...' : 'Submit'}
      </button>
    )}
  />
)
```