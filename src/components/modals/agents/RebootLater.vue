<template>
  <q-dialog ref="dialogRef" @hide="onDialogHide">
    <q-card class="dialog-plugin" style="min-width: 30vw">
      <q-bar>
        Schedule reboot on {{ agent.hostname }}
        <q-space />
        <q-btn dense flat icon="close" v-close-popup>
          <q-tooltip class="bg-white text-primary">Close</q-tooltip>
        </q-btn>
      </q-bar>
      <q-card-section>
        <q-input
          type="datetime-local"
          dense
          label="Reboot time"
          stack-label
          filled
          v-model="state.datetime"
          hint="Uses the agent's local time zone"
        />
      </q-card-section>
      <q-card-section class="q-gutter-sm">
        <q-toggle
          v-model="notifyUsers"
          label="Warn logged on users before rebooting"
        />
        <template v-if="notifyUsers">
          <q-input
            type="number"
            dense
            filled
            label="Countdown (minutes)"
            stack-label
            v-model.number="delayMinutes"
            :rules="[(v) => (v >= 1 && v <= 1440) || 'Between 1 and 1440']"
          />
          <q-input
            type="textarea"
            autogrow
            dense
            filled
            label="Message (optional)"
            stack-label
            v-model="message"
            maxlength="512"
            counter
          />
        </template>
      </q-card-section>
      <q-card-actions align="right">
        <q-btn dense flat push label="Cancel" v-close-popup />
        <q-btn
          :loading="loading"
          dense
          flat
          push
          label="Schedule Reboot"
          color="primary"
          @click="scheduleReboot"
        />
      </q-card-actions>
    </q-card>
  </q-dialog>
</template>

<script>
// composition imports
import { ref } from "vue";
import { useQuasar, useDialogPluginComponent, date } from "quasar";
import { scheduleAgentReboot } from "@/api/agents";
import { formatDateInputField } from "@/utils/format";

export default {
  name: "RebootLater",
  emits: [...useDialogPluginComponent.emits],
  props: {
    agent: !Object,
  },
  setup(props) {
    // setup quasar dialog plugin
    const { dialogRef, onDialogHide, onDialogOK } = useDialogPluginComponent();
    const $q = useQuasar();

    // setup reboot later logic
    const state = ref({
      datetime: formatDateInputField(date.addToDate(Date.now(), { hours: 1 })),
    });
    const loading = ref(false);
    const notifyUsers = ref(false);
    const delayMinutes = ref(5);
    const message = ref("");

    async function scheduleReboot() {
      loading.value = true;

      try {
        const payload = notifyUsers.value
          ? {
              ...state.value,
              delay: delayMinutes.value * 60,
              message: message.value,
            }
          : state.value;
        const ret = await scheduleAgentReboot(props.agent.agent_id, payload);
        $q.dialog({
          title: "Reboot pending",
          style: "width: 40vw",
          message: `A reboot has been scheduled for ${ret.time} on ${props.agent.hostname}. It can be cancelled from the Pending Actions menu until the scheduled time.`,
        }).onDismiss(onDialogOK);
      } catch (e) {
        console.error(e);
      }
      loading.value = false;
    }

    return {
      // reactive data
      state,
      loading,
      notifyUsers,
      delayMinutes,
      message,

      // methods
      scheduleReboot,

      // quasar dialog
      dialogRef,
      onDialogHide,
    };
  },
};
</script>
