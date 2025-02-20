<router lang="yaml">
  name: Kubernetes Settings
</router>
<template>
  <notifications
    class="k8s-wrapper"
    :notifications="notificationsList"
  >
    <div class="labeled-input">
      <label>Kubernetes version</label>
      <select class="select-k8s-version" :value="settings.kubernetes.version" @change="onChange($event)">
        <option v-for="item in versions" :key="item" :value="item" :selected="item === settings.kubernetes.version">
          {{ item }}
        </option>
      </select>
    </div>
    <system-preferences
      v-if="hasSystemPreferences"
      :memory-in-g-b="settings.kubernetes.memoryInGB"
      :number-c-p-us="settings.kubernetes.numberCPUs"
      :avail-memory-in-g-b="availMemoryInGB"
      :avail-num-c-p-us="availNumCPUs"
      :reserved-memory-in-g-b="6"
      :reserved-num-c-p-us="1"
      @update:memory="handleUpdateMemory"
      @update:cpu="handleUpdateCPU"
      @warning="handleWarning"
      @error="handleError"
    />
    <labeled-input :value="settings.kubernetes.port" label="Port" type="number" data-test="portConfig" @input="handleUpdatePort" />

    <split-button
      class="role-secondary btn-reset"
      data-test="k8sResetBtn"
      label="Reset Kubernetes"
      value="auto"
      :disabled="hasError || cannotReset"
      :options="[{id: 'wipe', label: 'Reset Kubernetes and Container Images'}]"
      @input="reset"
    />
    <label>
      Resetting Kubernetes to default will delete all workloads and configuration
    </label>
  </notifications>
</template>

<script>
import os from 'os';

import { ipcRenderer } from 'electron';
import semver from 'semver';

import SplitButton from '@/components/form/SplitButton.vue';
import LabeledInput from '@/components/form/LabeledInput.vue';
import Notifications from '@/components/Notifications.vue';
import SystemPreferences from '@/components/SystemPreferences.vue';
import * as K8s from '@/k8s-engine/k8s';

/** @typedef { import("../config/settings").Settings } Settings */

const NotificationLevels = ['error', 'warning', 'info', 'success'];

export default {
  name:       'K8s',
  title:      'Kubernetes Settings',
  components: {
    SplitButton,
    LabeledInput,
    Notifications,
    SystemPreferences,
  },
  data() {
    return {
      /** @type {{ key: string, message: string, level: string }} */
      notifications: { },
      state:         ipcRenderer.sendSync('k8s-state'),
      currentPort:   0,
      /** @type Settings */
      settings:      ipcRenderer.sendSync('settings-read'),
      /** @type {string[]} */
      versions:      [],
      progress:      {
        current: 0,
        max:     0,
      },
    };
  },

  computed: {
    hasSystemPreferences() {
      return !os.platform().startsWith('win');
    },
    availMemoryInGB() {
      return Math.ceil(os.totalmem() / 2 ** 30);
    },
    availNumCPUs() {
      return os.cpus().length;
    },
    cannotReset() {
      return ![K8s.State.STARTED, K8s.State.ERROR].includes(this.state);
    },
    notificationsList() {
      return Object.keys(this.notifications).map(key => ({
        key,
        message: this.notifications[key].message,
        color:   this.notifications[key].level,
      })).sort((left, right) => {
        return NotificationLevels.indexOf(left.color) - NotificationLevels.indexOf(right.color);
      });
    },
    hasError() {
      return Object.entries(this.notifications)
        ?.some(([_key, val]) => val.level === 'error');
    }
  },

  created() {
    this.$store.dispatch(
      'page/setHeader',
      { title: this.t('k8s.title') }
    );
    if (this.hasSystemPreferences) {
      // We don't configure WSL metrics, so don't bother making these checks on Windows.
      if (this.settings.kubernetes.memoryInGB > this.availMemoryInGB) {
        alert(`Reducing memory size from ${ this.settings.kubernetes.memoryInGB } to ${ this.availMemoryInGB }`);
        this.settings.kubernetes.memoryInGB = this.availMemoryInGB;
      }
      if (this.settings.kubernetes.numberCPUs > this.availNumCPUs) {
        alert(`Reducing # of CPUs from ${ this.settings.kubernetes.numberCPUs } to ${ this.availNumCPUs }`);
        this.settings.kubernetes.numberCPUs = this.availNumCPUs;
      }
    }
  },

  mounted() {
    const that = this;

    ipcRenderer.on('k8s-check-state', (event, stt) => {
      that.$data.state = stt;
    });
    ipcRenderer.on('k8s-current-port', (event, port) => {
      this.currentPort = port;
    });
    ipcRenderer.send('k8s-current-port');
    ipcRenderer.on('k8s-restart-required', (event, required) => {
      console.log(`restart-required-all`, required);
      for (const key in required) {
        console.log(`restart-required`, key, required[key]);
        if (required[key].length > 0) {
          const message = `The cluster must be reset for ${ key } change from ${ required[key][0] } to ${ required[key][1] }.`;

          this.handleNotification('info', `restart-${ key }`, message);
        } else {
          this.handleNotification('info', `restart-${ key }`, '');
        }
      }
    });
    ipcRenderer.on('k8s-versions', (event, versions) => {
      this.$data.versions = versions;
      if (versions.length === 0) {
        const message = 'No versions of Kubernetes were found';

        this.handleNotification('error', 'no-versions', message);
      } else if (!versions.includes(this.settings.kubernetes.version)) {
        const oldVersion = this.settings.kubernetes.version;

        if (oldVersion) {
          const message = `Saved Kubernetes version ${ oldVersion } not available, using ${ versions[0] }.`;

          this.handleNotification('info', 'invalid-version', message);
        }
        this.settings.kubernetes.version = versions[0];
      }
    });
    ipcRenderer.on('settings-update', (event, settings) => {
      // TODO: put in a status bar
      console.log('settings have been updated');
      this.$data.settings = settings;
    });
    ipcRenderer.send('k8s-restart-required');
    ipcRenderer.send('k8s-versions');
  },

  methods: {
    /**
     * Reset a Kubernetes cluster to default at the same version
     * @param { 'auto' | 'wipe' } mode How to do the reset
     */
    reset(mode) {
      const wipe = (mode === 'wipe') || (this.state !== K8s.State.STARTED);
      const consequence = {
        true:  'Wiping Kubernetes will delete all workloads, configuration, and images.',
        false: 'Resetting Kubernetes will delete all workloads and configuration.',
      }[wipe];

      if (confirm(`${ consequence } Do you want to proceed?`)) {
        for (const key in this.notifications) {
          this.handleNotification('info', key, '');
        }
        this.state = K8s.State.STOPPING;
        ipcRenderer.send('k8s-reset', wipe ? 'wipe' : 'fast');
      }
    },
    restart() {
      this.state = K8s.State.STOPPING;
      ipcRenderer.send('k8s-restart');
    },
    onChange(event) {
      if (event.target.value !== this.settings.kubernetes.version) {
        let confirmationMessage = '';

        if (this.settings.kubernetes.port !== this.currentPort) {
          confirmationMessage = `Changing versions will require a full reset of Kubernetes (loss of workloads) because the desired port has also changed (from ${ this.currentPort } to ${ this.settings.kubernetes.port })`;
        } else if (semver.lt(event.target.value, this.settings.kubernetes.version)) {
          confirmationMessage = `Changing from version ${ this.settings.kubernetes.version } to ${ event.target.value } will reset Kubernetes.`;
        } else {
          confirmationMessage = `Changing from version ${ this.settings.kubernetes.version } to ${ event.target.value } will upgrade Kubernetes`;
        }
        confirmationMessage += ' Do you want to proceed?';
        if (confirm(confirmationMessage)) {
          ipcRenderer.invoke('settings-write', { kubernetes: { version: event.target.value } })
            .then(() => this.restart());
        } else {
          alert('The Kubernetes version was not changed');
        }
      }
    },
    handleUpdateMemory(value) {
      this.settings.kubernetes.memoryInGB = value;
      ipcRenderer.invoke('settings-write',
        { kubernetes: { memoryInGB: value } });
    },
    handleUpdateCPU(value) {
      this.settings.kubernetes.numberCPUs = value;
      ipcRenderer.invoke('settings-write',
        { kubernetes: { numberCPUs: value } });
    },
    handleUpdatePort(value) {
      this.settings.kubernetes.port = value;
      ipcRenderer.invoke('settings-write',
        { kubernetes: { port: value } });
    },
    handleNotification(level, key, message) {
      if (message) {
        this.$set(this.notifications, key, {
          key, level, message
        });
      } else {
        this.$delete(this.notifications, key);
      }
    },
    handleWarning(key, message) {
      this.handleNotification('warning', key, message);
    },
    handleError(key, message) {
      this.handleNotification('error', key, message);
    },
  },
};
</script>

<style scoped>
.k8s-wrapper >>> .contents {
  padding-left: 1px;
}
.k8s-wrapper >>> .contents > *:not(hr) {
  max-width: calc(100% - 20px);
}
.select-k8s-version {
  width: inherit;
  display: inline-block;
}

.btn-reset {
  margin-right: 1rem;
}
</style>
