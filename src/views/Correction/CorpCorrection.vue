<template>
  <section id="ben-correction-view">
    <header>
      <h1>Register Correction</h1>
    </header>

    <section
      id="original-filing-date"
      class="mt-6"
    >
      <p>
        <span id="original-filing-date-label">Original Filing Date:</span>
        {{ originalFilingDate }}
      </p>
    </section>

    <YourCompanyWrapper class="mt-10">
      <div>
        <EntityName />
        <NameTranslation />
      </div>
      <RecognitionDateTime />
      <OfficeAddresses />
      <BusinessContactInfo />
      <FolioInformation />
    </YourCompanyWrapper>

    <PeopleAndRoles class="mt-10" />

    <ShareStructures class="mt-10" />

    <Articles class="mt-10" />

    <CompletingParty
      v-if="isClientErrorCorrection"
      class="mt-10"
      sectionNumber="1."
      :validate="true"
    />

    <Detail
      class="mt-10"
      :sectionNumber="isClientErrorCorrection ? '2.' : '1.'"
      :validate="true"
    />

    <CertifySection
      v-if="isClientErrorCorrection"
      class="mt-10"
      :sectionNumber="isClientErrorCorrection ? '3.' : '2.'"
      :validate="true"
      :disableEdit="false"
    />

    <StaffPayment
      v-if="IsAuthorized(AuthorizedActions.STAFF_PAYMENT)"
      class="mt-10"
      :sectionNumber="isClientErrorCorrection ? '4.' : '2.'"
      @haveChanges="onStaffPaymentChanges()"
    />
  </section>
</template>

<script lang="ts">
import { Component, Emit, Mixins, Prop, Watch } from 'vue-property-decorator'
import { Action } from 'pinia-class'
import { Articles } from '@/components/Alteration/'
import {
  BusinessContactInfo, CertifySection, CompletingParty, Detail, EntityName, FolioInformation, NameTranslation,
  OfficeAddresses, PeopleAndRoles, RecognitionDateTime, ShareStructures, StaffPayment, YourCompanyWrapper
} from '@/components/common/'
import { CommonMixin, FeeMixin, FilingTemplateMixin } from '@/mixins/'
import { AuthServices, DateUtilities, LegalServices } from '@/services/'
import { StaffPaymentOptions } from '@bcrs-shared-components/enums/'
import { CorrectionFilingIF, EntitySnapshotIF, ResourceIF } from '@/interfaces/'
import * as Resources from '@/resources/Correction/'
import { useStore } from '@/store/store'
import { CorpTypeCd } from '@bcrs-shared-components/corp-type-module'
import { IsAuthorized } from '@/utils'
import { AuthorizedActions } from '@/enums'

/** Correction sub-component for corp class "BC" entities. */
@Component({
  components: {
    Articles,
    BusinessContactInfo,
    CertifySection,
    CompletingParty,
    Detail,
    EntityName,
    FolioInformation,
    NameTranslation,
    OfficeAddresses,
    PeopleAndRoles,
    RecognitionDateTime,
    ShareStructures,
    StaffPayment,
    YourCompanyWrapper
  }
})
export default class CorpCorrection extends Mixins(CommonMixin, FeeMixin, FilingTemplateMixin) {
  // for template
  readonly IsAuthorized = IsAuthorized
  readonly AuthorizedActions = AuthorizedActions

  // Store getters
  // @Getter(useStore) getEntityType!: CorpTypeCd

  // Store actions
  @Action(useStore) setHaveUnsavedChanges!: (x: boolean) => void
  @Action(useStore) setResource!: (x: ResourceIF) => void

  /** The draft correction filing to process. */
  @Prop({ default: () => null }) readonly correctionFiling!: CorrectionFilingIF

  /** The original filing date, in Pacific time. */
  get originalFilingDate (): string {
    return DateUtilities.yyyyMmDdToPacificDate(this.getCorrectedFilingDate, true)
  }

  /** The resource object for a correction filing. */
  get correctionResource (): ResourceIF {
    switch (this.getEntityType) {
      case CorpTypeCd.BC_CCC: return Resources.CorrectionResourceCc
      case CorpTypeCd.BC_COMPANY: return Resources.CorrectionResourceBc
      case CorpTypeCd.BC_ULC_COMPANY: return Resources.CorrectionResourceUlc
      case CorpTypeCd.BEN_CONTINUE_IN: return Resources.CorrectionResourceCben
      case CorpTypeCd.BENEFIT_COMPANY: return Resources.CorrectionResourceBen
      case CorpTypeCd.CCC_CONTINUE_IN: return Resources.CorrectionResourceCcc
      case CorpTypeCd.CONTINUE_IN: return Resources.CorrectionResourceC
      case CorpTypeCd.ULC_CONTINUE_IN: return Resources.CorrectionResourceCul
      default: return null // should never happen
    }
  }

  /**
   * Called when correction filing is fetched and this component can continue loading.
   * Must be "immediate" because this component is only rendered when we have the data.
   */
  @Watch('correctionFiling', { immediate: true })
  private async onCorrectionFiling (): Promise<void> {
    // fetch the rest of the data
    try {
      // safety check
      if (!this.correctionFiling) throw (new Error('Missing correction filing. Try reloading the page.'))

      // fetch entity snapshot
      const entitySnapshot = await this.fetchEntitySnapshot()

      // parse draft correction filing and entity snapshot into store
      this.parseCorrectionFiling(this.correctionFiling, entitySnapshot)

      if (!this.correctionResource) {
        throw new Error(`Invalid correction resource entity type = ${this.getEntityType}`)
      }

      // set the resources
      this.setResource(this.correctionResource)

      // initialize Fee Summary data
      this.setFilingData([this.correctionResource.filingData])

      // pre-select No Fee option
      this.setStaffPayment({ option: StaffPaymentOptions.NO_FEE } as any)

      // tell App that we're finished loading
      this.emitHaveData()
    } catch (err) {
      console.log(err) // eslint-disable-line no-console
      this.emitFetchError(err)
    }

    // now that all data is loaded, wait for things to stabilize and reset flag
    this.$nextTick(() => this.setHaveUnsavedChanges(false))
  }

  /** Fetches the entity snapshot. */
  private async fetchEntitySnapshot (): Promise<EntitySnapshotIF> {
    const items = await Promise.all([
      LegalServices.fetchBusinessInfo(this.getBusinessId),
      AuthServices.fetchAuthInfo(this.getBusinessId),
      LegalServices.fetchAddresses(this.getBusinessId),
      LegalServices.fetchParties(this.getBusinessId),
      LegalServices.fetchShareStructure(this.getBusinessId),
      LegalServices.fetchNameTranslations(this.getBusinessId),
      LegalServices.fetchResolutions(this.getBusinessId)
    ])

    if (items.length !== 7) throw new Error('Failed to fetch entity snapshot')

    return {
      businessInfo: items[0],
      authInfo: items[1],
      addresses: items[2],
      orgPersons: items[3],
      shareStructure: items[4],
      nameTranslations: items[5],
      resolutions: items[6]
    } as EntitySnapshotIF
  }

  /** Emits Fetch Error event. */
  @Emit('fetchError')
  // eslint-disable-next-line @typescript-eslint/no-unused-vars
  private emitFetchError (err: unknown = null): void {}

  /** Emits Have Data event. */
  @Emit('haveData')
  // eslint-disable-next-line @typescript-eslint/no-unused-vars
  private emitHaveData (haveData = true): void {}
}
</script>

<style lang="scss" scoped>
#original-filing-date-label {
  letter-spacing: -0.04rem;
  font-weight: bold;
}
</style>
