<template>
  <v-dialog
    v-model="dialog"
    width="45rem"
    persistent
    :attach="attach"
    content-class="file-pay-nr-error-dialog"
  >
    <v-card>
      <v-card-title id="dialog-title">
        Invalid Name Request (NR) / Incorporation Application
      </v-card-title>
      <v-card-text id="dialog-text">
        <!-- display errors -->
        <div class="font-15 mb-4">
          <p>
            <b>
              The Name Request {{ getNameRequestNumber || 'Unknown' }} and the Incorporation Application for
              {{ getNameRequestLegalName || 'Unknown' }} are no longer valid.
            </b>
          </p>
          <p>If you still wish to incorporate a Benefit Company, please contact Registry Staff as soon as possible.</p>
          <div class="mt-5 info-section">
            IMPORTANT:
          </div>
          <div class="info-section">
            Once the reservation period for a Name Request expires or is otherwise cancelled, that name becomes
            available to anyone wishing to start their business with that name.
          </div>
        </div>
        <p class="font-15">
          Registries contact information:
        </p>
        <ErrorContact />
      </v-card-text>

      <v-divider class="my-0" />

      <!-- if there are errors, or neither errors nor warnings... -->
      <v-card-actions>
        <v-spacer />
        <v-btn
          id="dialog-okay-button"
          color="primary"
          text
          @click="okay()"
        >
          OK
        </v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<script lang="ts">
import Vue from 'vue'
import { Component, Prop, Emit } from 'vue-property-decorator'
import { Getter } from 'pinia-class'
import { ErrorContact } from '@/components/common/'
import { useStore } from '@/store/store'

@Component({
  components: {
    ErrorContact
  }
})
export default class FileAndPayInvalidNameRequestDialog extends Vue {
  /** Prop to display the dialog. */
  @Prop() readonly dialog!: boolean

  /** Prop to provide attachment selector. */
  @Prop() readonly attach!: string

  @Getter(useStore) getNameRequestNumber!: string
  @Getter(useStore) getNameRequestLegalName!: string

  @Emit() okay (): void {}
}
</script>
