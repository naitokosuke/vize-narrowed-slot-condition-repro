# vize-narrowed-slot-condition-repro

Reported as [ubugeeei-prod/vize#10660](https://github.com/ubugeeei-prod/vize/issues/10660).

Minimal reproduction for a false type error in [vize](https://github.com/ubugeeei-prod/vize)'s content mapper.

When a component with an explicit `<template #default>` is repeated by `v-for` inside a `v-else` branch, the generated code repeats the `v-if` condition inside a block where TypeScript has already narrowed it away. The comparison then reports TS2367.

## Steps

```sh
pnpm install
pnpm check
```

[`src/Status.vue`](./src/Status.vue):

```vue
<script setup lang="ts">
import { ref } from "vue";
import Box from "./Box.vue";

const status = ref<"pending" | "error" | "success">("pending");
const fruits = ref(["Apple"]);
</script>

<template>
  <p v-if="status === 'pending'">Loading</p>
  <div v-else>
    <Box v-for="fruit in fruits" :key="fruit">
      <template #default>{{ fruit }}</template>
    </Box>
  </div>
</template>
```

`pnpm check` runs `tsc --runExternalCode --noEmit -p tsconfig.json` with `"contentMappers": [{ "package": "vize", "extensions": [".vue"] }]`.

## Expected

No error. Nothing in the template compares `status` inside the `v-else` branch.

## Actual

```
src/Status.vue(179,194): error TS2367: This comparison appears to be unintentional because the types '"error" | "success"' and '"pending"' have no overlap.
  This location is in virtual code produced by the content mapper 'vize@0.569.0' and has no corresponding location in the original file.
src/Status.vue(261,194): error TS2367: This comparison appears to be unintentional because the types '"error" | "success"' and '"pending"' have no overlap.
  This location is in virtual code produced by the content mapper 'vize@0.569.0' and has no corresponding location in the original file.
```

## Notes

- The generated code, from `vize check --save-virtual-ts-for src/Status.vue src`:

  ```ts
  if (!(status === 'pending')) {
    // …
        // @ts-ignore Inference-only guard; authored v-if checks own diagnostics.
        const __vize_slot_host_9 = (…)(new (…)((!(status === 'pending')) ? {
        } : (undefined as unknown as never), …), (…)(Box)((!(status === 'pending')) ? {
  ```

  [`generate_slot_host_binding`](https://github.com/ubugeeei-prod/vize/blob/v0.569.0/crates/vize_canon/src/virtual_ts/expressions/generic_props_call.rs#L160-L220) writes the guard twice but emits one `// @ts-ignore`. The props literal between them contains a line break, so the second guard lands on the next line, which the `@ts-ignore` does not cover.
- [#10216](https://github.com/ubugeeei-prod/vize/issues/10216) (fixed) had the same shape in the "Component props in v-for scope" block. This is a different emitter, and it still fails on 0.569.0, where the #10216 reproduction passes.
- Without `v-for`, or with the default slot written implicitly (`<Box v-for="…">{{ fruit }}</Box>`), there is no error.
- `vize check src` reports `No type errors found!`. Vize's own checker also gets the error from the native TypeScript checker, but it drops diagnostics that have no location in the `.vue` file.

## Environment

vize 0.569.0, TypeScript 7.1.0-beta, Vue 3.5.44, Node v26.12.1, pnpm 12.9.0, macOS (arm64)
