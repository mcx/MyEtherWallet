<template>
  <mew-module
    class="d-flex flex-grow-1 pt-6"
    :title="title"
    :has-elevation="true"
    :has-indicator="true"
  >
    <template #moduleBody>
      <div>
        <v-textarea
          v-model="message"
          outlined
          label="Signature"
          :value="message"
        ></v-textarea>
        <div class="text-right">
          <mew-button
            btn-style="light"
            title="Clear all"
            class="mr-4"
            @click.native="clearAll"
          />
          <mew-button
            title="Sign"
            :disabled="disableSignBtn"
            @click.native="signMessage"
          />
        </div>
      </div>
    </template>
  </mew-module>
</template>

<script>
import { mapState } from 'vuex';
import { getRTLOLTLOSafeString } from '@/core/helpers/getRTLOLTLOSafeString.js';
import SignAndVerifyMessage from '@/modules/message/handlers';
export default {
  name: 'ModuleMessageSign',
  data() {
    return {
      title: 'Sign Message',
      message: '',
      signature: '',
      signAndVerify: ''
    };
  },
  computed: {
    ...mapState('wallet', ['instance']),
    disableSignBtn() {
      return this.message === '';
    }
  },
  mounted() {
    this.signAndVerify = new SignAndVerifyMessage();
  },
  methods: {
    signMessage() {
      try {
        this.signAndVerify
          .signMessage(getRTLOLTLOSafeString(this.message))
          .then(() => {
            this.message = '';
          })
          .catch(e => {
            this.message = '';
            this.instance.errorHandler(e.message);
          });
      } catch (e) {
        this.instance.errorHandler(e.hasOwnProperty('message') ? e.message : e);
      }
    },
    clearAll() {
      this.signature = '';
      this.message = '';
    }
  }
};
</script>
