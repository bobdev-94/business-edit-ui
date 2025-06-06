<template>
  <section
    id="certify-section"
    class="pb-6"
  >
    <h2>{{ sectionNumber }} Certify</h2>

    <div class="py-4">
      Enter the legal name of the person authorized to complete and submit these changes.
    </div>

    <div :class="{ 'invalid-section': invalidSection }">
      <v-card
        flat
        class="section-container py-6"
      >
        <CertifyShared
          :currentDate="getCurrentDate"
          :certifiedBy="getCertifyState.certifiedBy"
          :isCertified="getCertifyState.valid"
          :entityDisplay="readableEntityType"
          :message="certifyMessage"
          :isStaff="IsAuthorized(AuthorizedActions.THIRD_PARTY_CERTIFY_STMT)"
          :firstColumn="3"
          :secondColumn="9"
          :validate="validate"
          :invalidSection="invalidSection"
          :disableEdit="disableEdit"
          @update:certifiedBy="onCertifiedBy($event)"
          @update:isCertified="onIsCertified($event)"
        />
      </v-card>
    </div>
  </section>
</template>

<script lang="ts">
import { Component, Mixins, Prop } from 'vue-property-decorator'
import { Action, Getter } from 'pinia-class'
import { Certify as CertifyShared } from '@bcrs-shared-components/certify/'
import { DateMixin } from '@/mixins/'
import { CertifyIF, ResourceIF } from '@/interfaces/'
import { CorpTypeCd, GetCorpFullDescription } from '@bcrs-shared-components/corp-type-module/'
import { useStore } from '@/store/store'
import { IsAuthorized } from '@/utils'
import { AuthorizedActions } from '@/enums'

@Component({
  components: {
    CertifyShared
  }
})
export default class CertifySection extends Mixins(DateMixin) {
  // for template
  readonly IsAuthorized = IsAuthorized
  readonly AuthorizedActions = AuthorizedActions

  @Getter(useStore) getCertifyState!: CertifyIF
  @Getter(useStore) getCurrentDate!: string
  @Getter(useStore) getResource!: ResourceIF
  @Getter(useStore) getEntityType!: CorpTypeCd

  @Action(useStore) setCertifyState!: (x: CertifyIF) => void
  @Action(useStore) setCertifyStateValidity!: (x: boolean) => void

  /** Prop to provide section number. */
  @Prop({ default: '' }) readonly sectionNumber!: string

  /** Whether to perform validation. */
  @Prop({ default: false }) readonly validate!: boolean

  /** To determine whether user input is enabled. */
  @Prop({ required: true }) readonly disableEdit!: boolean

  /** Called when component is mounted. */
  mounted (): void {
    this.setCertifyState(
      {
        valid: this.getCertifyState.valid,
        certifiedBy: this.getCertifyState.certifiedBy
      }
    )
  }

  /** Get the entity type in readable format */
  get readableEntityType (): string {
    return (this.getResource?.certifyText || GetCorpFullDescription(this.getEntityType))
  }

  /** Get the certify resource message */
  get certifyMessage (): string {
    return this.getResource?.certifyClause
  }

  /** Handler for Valid change event. */
  onIsCertified (val: boolean): void {
    this.setCertifyState(
      {
        valid: val,
        certifiedBy: this.getCertifyState.certifiedBy
      }
    )
    this.setCertifyStateValidity(Boolean(val && this.getCertifyState.certifiedBy))
  }

  /** Handler for CertifiedBy change event. */
  onCertifiedBy (val: string): void {
    this.setCertifyState(
      {
        valid: this.getCertifyState.valid,
        certifiedBy: val
      }
    )
    this.setCertifyStateValidity(Boolean(this.getCertifyState.valid && val))
  }

  /** True if invalid class should be set for certify section container. */
  get invalidSection (): boolean {
    return (this.validate && !(this.getCertifyState.valid && this.getCertifyState.certifiedBy))
  }
}
</script>

<style lang="scss" scoped>
@import '@/assets/styles/theme.scss';

.invalid {
  margin-left: -4px !important;
  border-left: 4px solid $BCgovInputError;
}

.invalid-label {
  color: $BCgovInputError;
}
</style>
