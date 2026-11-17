<template>
  <q-dialog ref="dialogRef" @hide="onHide">
    <q-card class="q-dialog-plugin" style="width: 70vw; max-width: 80vw">
      <q-card-section class="row items-center q-gutter-sm">
        <div class="text-h6">{{ entry.command }}</div>
        <q-chip
          dense
          :color="finished ? 'positive' : 'info'"
          text-color="white"
          :icon="finished ? 'check' : undefined"
        >
          <q-spinner v-if="!finished" size="1em" class="q-mr-xs" />
          {{ finished ? "Done" : "Running, updates every few seconds" }}
        </q-chip>
        <q-btn
          dense
          flat
          size="md"
          icon="content_copy"
          @click="copyOutput(output)"
          ><q-tooltip>Copy to Clipboard</q-tooltip></q-btn
        >
      </q-card-section>

      <q-card-section class="q-pt-none">
        <pre
          ref="preRef"
          class="q-pa-sm"
          style="max-height: 65vh; overflow: auto"
          >{{ output }}</pre
        >
      </q-card-section>

      <q-card-actions align="right">
        <q-btn label="OK" @click="onDialogOK" />
      </q-card-actions>
    </q-card>
  </q-dialog>
</template>

<script setup>
// Live output of a Linux patch install (#2556): the agent streams apt/apk
// output into the history entry, this re-reads it until the run is done
import { ref, computed, nextTick, onMounted } from "vue";
import { useDialogPluginComponent } from "quasar";
import { copyOutput } from "@/utils/helpers";
import { fetchAgentHistory } from "@/api/agents";

const props = defineProps({
  agentId: { type: String, required: true },
  entry: { type: Object, required: true },
});

defineEmits([...useDialogPluginComponent.emits]);

const { dialogRef, onDialogHide, onDialogOK } = useDialogPluginComponent();

const POLL_MS = 3000;
// a running install is among the newest entries
const PAGE_SIZE = 50;

const output = ref(props.entry.results || "");
const preRef = ref(null);
let timer = null;

// the agent ends every run with one of these, the server marks unstarted ones
const finished = computed(
  () =>
    /^== \d\d:\d\d:\d\d (Finished|Skipped|Failed)/m.test(output.value) ||
    output.value.startsWith("Not started"),
);

async function poll() {
  const data = await fetchAgentHistory(props.agentId, {
    page: 1,
    page_size: PAGE_SIZE,
    ordering: "-time",
  });
  const row = data?.results?.find((r) => r.id === props.entry.id);
  if (row && row.results !== output.value) {
    // keep following the end unless the admin scrolled up
    const el = preRef.value;
    const atBottom =
      !el || el.scrollHeight - el.scrollTop - el.clientHeight < 40;
    output.value = row.results || "";
    if (atBottom) {
      await nextTick();
      if (preRef.value) preRef.value.scrollTop = preRef.value.scrollHeight;
    }
  }
  if (!finished.value) timer = setTimeout(poll, POLL_MS);
}

function onHide() {
  clearTimeout(timer);
  onDialogHide();
}

onMounted(() => {
  if (!finished.value) timer = setTimeout(poll, POLL_MS);
});
</script>
