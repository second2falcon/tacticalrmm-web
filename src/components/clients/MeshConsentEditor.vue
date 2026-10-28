<template>
  <div>
    <q-toggle
      :model-value="modelValue !== null"
      :label="label"
      @update:model-value="(on) => emit('update:modelValue', on ? 0 : null)"
    />
    <div v-if="modelValue !== null" class="row q-col-gutter-sm q-pl-lg">
      <div v-for="group in groups" :key="group.name" class="col-4">
        <div class="text-weight-medium">{{ group.name }}</div>
        <q-checkbox
          v-for="flag in group.flags"
          :key="flag.bit"
          dense
          class="full-width"
          :label="flag.label"
          :model-value="(modelValue & flag.bit) !== 0"
          @update:model-value="emit('update:modelValue', modelValue ^ flag.bit)"
        />
      </div>
    </div>
    <div v-else class="text-caption text-grey q-pl-lg">{{ offHint }}</div>
  </div>
</template>

<script setup>
// MeshCentral user consent bitmask (#2292, #2404), null = not set here
defineProps({
  modelValue: { type: Number, default: null },
  label: { type: String, required: true },
  offHint: { type: String, required: true },
});
const emit = defineEmits(["update:modelValue"]);

const groups = [
  {
    name: "Desktop",
    flags: [
      { bit: 0x01, label: "Notify user" },
      { bit: 0x08, label: "Ask for consent" },
      { bit: 0x40, label: "Show privacy bar" },
    ],
  },
  {
    name: "Terminal",
    flags: [
      { bit: 0x02, label: "Notify user" },
      { bit: 0x10, label: "Ask for consent" },
    ],
  },
  {
    name: "Files",
    flags: [
      { bit: 0x04, label: "Notify user" },
      { bit: 0x20, label: "Ask for consent" },
    ],
  },
];
</script>
