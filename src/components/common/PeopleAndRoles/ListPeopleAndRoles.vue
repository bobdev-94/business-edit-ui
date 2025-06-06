<template>
  <div id="list-people-roles">
    <!-- conditionally render add component -->
    <v-expand-transition>
      <div
        v-if="renderOrgPersonForm && isNaN(activeIndex)"
        id="people-roles-add"
        class="section-container"
        :class="{
          'section-container': !isSummaryView,
          'invalid-section': isComponentInvalid
        }"
      >
        <OrgPerson
          :currentOrgPerson="currentOrgPerson"
          :activeIndex="activeIndex"
          @addEdit="emitAddEdit($event)"
          @remove="emitRemove($event)"
          @reset="emitReset()"
        />
      </div>
    </v-expand-transition>

    <!-- List Display Section -->
    <div
      v-if="currentPeopleAndRoles.length > 0"
      id="people-roles-list"
      :class="{
        'section-container': !isSummaryView,
        'invalid-section': isComponentInvalid
      }"
    >
      <!-- List Headers -->
      <v-row
        v-if="showHeaders"
        class="people-roles-list-header list-item__subtitle pt-0 pb-4"
        no-gutters
      >
        <v-col
          v-for="(title, index) in tableHeaders"
          :key="index"
          cols="12"
          sm="3"
          :class="{'summary-cols': isSummaryView}"
        >
          <span>{{ title }}</span>
        </v-col>
        <!-- Spacer Column for Actions -->
        <v-col class="actions-width" />
      </v-row>

      <!-- List Content -->
      <template v-for="(orgPerson, index) in currentPeopleAndRoles">
        <v-row
          v-if="!(wasReplaced(orgPerson) && wasRemoved(orgPerson))"
          :key="index"
          class="people-roles-content section-container"
          :class="{ 'summary-view': isSummaryView, }"
          no-gutters
        >
          <!-- conditionally render edit component instead of table row -->
          <v-expand-transition>
            <div
              v-if="renderOrgPersonForm && (index === activeIndex)"
              id="people-roles-edit"
            >
              <OrgPerson
                :currentOrgPerson="currentOrgPerson"
                :activeIndex="activeIndex"
                @addEdit="emitAddEdit($event)"
                @remove="emitRemove($event)"
                @reset="emitReset()"
              />
            </div>
          </v-expand-transition>

          <!-- normal table row -->
          <template v-if="!renderOrgPersonForm || (index != activeIndex)">
            <!-- Name + Badge -->
            <v-col
              class="pr-2"
              cols="12"
              sm="3"
            >
              <v-row no-gutters>
                <v-col
                  cols="1"
                  class="mt-n1 ml-n1 mr-3 badges"
                  :class="{ 'removed': wasRemoved(orgPerson)}"
                >
                  <v-icon
                    v-if="isPartyTypePerson(orgPerson)"
                    color="gray9"
                  >
                    mdi-account
                  </v-icon>
                  <v-icon
                    v-if="isPartyTypeOrg(orgPerson)"
                    color="gray9"
                  >
                    mdi-domain
                  </v-icon>
                </v-col>
                <v-col class="overflow-hidden">
                  <p
                    class="people-roles-title mb-1"
                    :class="{ 'removed': wasRemoved(orgPerson)}"
                  >
                    {{ formatName(orgPerson) }}
                  </p>
                  <p
                    v-if="showEmailUnderName"
                    class="info-text mb-1 people-roles-email"
                    :class="{ 'removed': wasRemoved(orgPerson)}"
                  >
                    {{ orgPerson.officer.email }}
                  </p>
                  <p
                    v-if="orgPerson.officer.identifier"
                    class="info-text mb-1 people-roles-inc-number"
                    :class="{ 'removed': wasRemoved(orgPerson)}"
                  >
                    Incorporation Number: {{ orgPerson.officer.identifier }}
                  </p>

                  <div v-if="!isSummaryView && orgPerson.actions">
                    <!-- show DELETED item as black on light gray -->
                    <v-chip
                      v-if="wasRemoved(orgPerson)"
                      x-small
                      label
                      color="grey lighten-2"
                    >
                      REMOVED
                    </v-chip>

                    <!-- show REPLACED item separately (since item also has ADDED action) -->
                    <v-chip
                      v-else-if="wasReplaced(orgPerson)"
                      x-small
                      label
                      color="primary"
                      text-color="white"
                    >
                      CHANGED
                    </v-chip>

                    <!-- all other states -->
                    <template v-else>
                      <v-chip
                        v-for="(action, i) in orgPerson.actions"
                        :key="`action-chip-${i}`"
                        x-small
                        label
                        color="primary"
                        text-color="white"
                      >
                        {{ action }}
                      </v-chip>
                    </template>
                  </div>
                </v-col>
              </v-row>
            </v-col>

            <!-- Mailing Address -->
            <v-col
              cols="12"
              :sm="isSummaryView ? 4 : 3"
              :class="{ 'removed': wasRemoved(orgPerson)}"
            >
              <MailingAddress
                class="peoples-roles-mailing-address"
                :address="orgPerson.mailingAddress"
              />
            </v-col>

            <!-- Delivery Address -->
            <v-col
              v-if="showDeliveryAddressColumn"
              cols="12"
              sm="3"
              :class="{ 'removed': wasRemoved(orgPerson)}"
            >
              <template
                v-if="hasRoleDirector(orgPerson) || hasRoleProprietor(orgPerson) || hasRolePartner(orgPerson)"
              >
                <template v-if="IsSame(orgPerson.mailingAddress, orgPerson.deliveryAddress, ['id'])">
                  <span class="peoples-roles-delivery-address info-text">Same as Mailing Address</span>
                </template>
                <template v-else>
                  <DeliveryAddress
                    class="peoples-roles-delivery-address"
                    :address="orgPerson.deliveryAddress"
                  />
                </template>
              </template>
            </v-col>

            <!-- Email Address -->
            <v-col
              v-if="showEmailColumn"
              cols="12"
              sm="3"
              :class="{ 'removed': wasRemoved(orgPerson)}"
            >
              <p>{{ orgPerson.officer.email }}</p>
            </v-col>

            <!-- Roles -->
            <v-col
              v-if="showRolesColumn"
              cols="12"
              sm="2"
              :class="{ 'removed': wasRemoved(orgPerson)}"
            >
              <!-- Warning if orgPerson has no roles -->
              <div v-if="orgPerson.roles.length > 0">
                <v-col
                  v-for="(role, key) in orgPerson.roles"
                  :key="key"
                  class="col-roles"
                >
                  <span class="info-text small-text break-spaces pr-2">{{ role.roleType }}</span>
                </v-col>
              </div>
              <div v-else>
                <v-icon color="red darken-3">
                  mdi-alert
                </v-icon>
                <span class="warning-text small-text">Missing Role</span>
              </div>
            </v-col>

            <!-- Actions Buttons -->
            <v-col
              v-if="!isSummaryView"
              class="actions"
            >
              <!-- org-person that was replaced: -->
              <div
                v-if="wasReplaced(orgPerson)"
                class="actions pr-5"
              >
                <span class="undo-action">
                  <v-btn
                    :id="`officer-${index}-undo-btn`"
                    text
                    color="primary"
                    :disabled="renderOrgPersonForm"
                    @click="emitUndo(index); dropdown[index]=false"
                  >
                    <v-icon small>mdi-undo</v-icon>
                    <span>Undo</span>
                  </v-btn>
                </span>
              </div>

              <!-- org-person that was added: -->
              <div
                v-else-if="wasAdded(orgPerson)"
                class="actions"
              >
                <span class="edit-action">
                  <v-btn
                    :id="`officer-${index}-edit-btn`"
                    text
                    color="primary"
                    :disabled="renderOrgPersonForm"
                    @click="emitInitEdit(index)"
                  >
                    <v-icon small>mdi-pencil</v-icon>
                    <span>Edit</span>
                  </v-btn>
                </span>

                <!-- More Actions Menu (Remove action) -->
                <span
                  class="dropdown-action mr-4"
                  :class="`more-actions-${index}`"
                >
                  <v-menu
                    v-model="dropdown[index]"
                    offset-y
                    left
                    nudge-bottom="4"
                    :attach="`#list-people-roles .more-actions-${index}`"
                  >
                    <template #activator="{ on }">
                      <v-btn
                        text
                        small
                        color="primary"
                        class="more-actions-btn"
                        :disabled="renderOrgPersonForm"
                        v-on="on"
                      >
                        <v-icon>{{ dropdown[index] ? 'mdi-menu-up' : 'mdi-menu-down' }}</v-icon>
                      </v-btn>
                    </template>

                    <v-list>
                      <v-list-item
                        :id="`officer-${index}-remove-btn`"
                        @click="emitRemove(index); dropdown[index]=false"
                      >
                        <v-list-item-subtitle>
                          <v-icon small>mdi-delete</v-icon>
                          <span class="ml-1">Remove</span>
                        </v-list-item-subtitle>
                      </v-list-item>
                    </v-list>
                  </v-menu>
                </span>
              </div>

              <!-- org-person that was removed: -->
              <div
                v-else-if="wasRemoved(orgPerson)"
                class="actions pr-5"
              >
                <span class="undo-action">
                  <v-btn
                    :id="`officer-${index}-undo-btn`"
                    text
                    color="primary"
                    :disabled="renderOrgPersonForm"
                    @click="emitUndo(index); dropdown[index]=false"
                  >
                    <v-icon small>mdi-undo</v-icon>
                    <span>Undo</span>
                  </v-btn>
                </span>
              </div>

              <!-- org-person that was edited/corrected/changed: -->
              <div
                v-else-if="wasEdited(orgPerson) || wasCorrected(orgPerson) || wasChanged(orgPerson)"
                class="actions"
              >
                <span class="undo-action">
                  <v-btn
                    :id="`officer-${index}-undo-btn`"
                    text
                    color="primary"
                    :disabled="renderOrgPersonForm"
                    @click="emitUndo(index); dropdown[index]=false"
                  >
                    <v-icon small>mdi-undo</v-icon>
                    <span>Undo</span>
                  </v-btn>
                </span>

                <!-- More Actions Menu (Change + Remove actions) -->
                <span
                  class="dropdown-action mr-4"
                  :class="`more-actions-${index}`"
                >
                  <v-menu
                    v-model="dropdown[index]"
                    offset-y
                    left
                    nudge-bottom="4"
                    :attach="`#list-people-roles .more-actions-${index}`"
                  >
                    <template #activator="{ on }">
                      <v-btn
                        text
                        small
                        color="primary"
                        class="more-actions-btn"
                        :disabled="renderOrgPersonForm"
                        v-on="on"
                      >
                        <v-icon>{{ dropdown[index] ? 'mdi-menu-up' : 'mdi-menu-down' }}</v-icon>
                      </v-btn>
                    </template>

                    <v-list>
                      <v-list-item
                        :id="`officer-${index}-edit-btn`"
                        @click="emitInitEdit(index); dropdown[index]=false"
                      >
                        <v-list-item-subtitle>
                          <v-icon small>mdi-pencil</v-icon>
                          <span class="ml-1">Change</span>
                        </v-list-item-subtitle>
                      </v-list-item>

                      <v-list-item
                        v-if="canRemove(orgPerson)"
                        :id="`officer-${index}-remove-btn`"
                        @click="emitRemove(index); dropdown[index]=false"
                      >
                        <v-list-item-subtitle>
                          <v-icon small>mdi-delete</v-icon>
                          <span class="ml-1">Remove</span>
                        </v-list-item-subtitle>
                      </v-list-item>
                    </v-list>
                  </v-menu>
                </span>
              </div>

              <!-- org-person we haven't touched: -->
              <div
                v-else
                class="actions mr-4"
              >
                <span
                  v-if="showChangeButtonForSoleProps"
                  class="edit-action"
                >
                  <v-btn
                    :id="`officer-${index}-edit-btn`"
                    text
                    color="primary"
                    :disabled="renderOrgPersonForm"
                    @click="emitInitEdit(index)"
                  >
                    <v-icon small>mdi-pencil</v-icon>
                    <span>{{ getEditLabel }}</span>
                  </v-btn>
                </span>

                <!-- More Actions Menu (Remove action) -->
                <span
                  v-if="canRemove(orgPerson)"
                  class="dropdown-action"
                  :class="`more-actions-${index}`"
                >
                  <v-menu
                    v-model="dropdown[index]"
                    offset-y
                    left
                    nudge-bottom="4"
                    :attach="`#list-people-roles .more-actions-${index}`"
                  >
                    <template #activator="{ on }">
                      <v-btn
                        text
                        small
                        color="primary"
                        class="more-actions-btn"
                        :disabled="renderOrgPersonForm"
                        v-on="on"
                      >
                        <v-icon>{{ dropdown[index] ? 'mdi-menu-up' : 'mdi-menu-down' }}</v-icon>
                      </v-btn>
                    </template>

                    <v-list>
                      <v-list-item
                        :id="`officer-${index}-remove-btn`"
                        @click="emitRemove(index)"
                      >
                        <v-list-item-subtitle>
                          <v-icon small>mdi-delete</v-icon>
                          <span class="ml-1">Remove</span>
                        </v-list-item-subtitle>
                      </v-list-item>
                    </v-list>
                  </v-menu>
                </span>

                <!-- More Actions Menu (Replace action) -->
                <span
                  v-if="canReplace(orgPerson)"
                  class="dropdown-action"
                  :class="`more-actions-${index}`"
                >
                  <v-menu
                    v-model="dropdown[index]"
                    offset-y
                    left
                    nudge-bottom="4"
                    :attach="`#list-people-roles .more-actions-${index}`"
                  >
                    <template #activator="{ on }">
                      <v-btn
                        text
                        small
                        color="primary"
                        class="more-actions-btn"
                        :disabled="renderOrgPersonForm"
                        v-on="on"
                      >
                        <v-icon>{{ dropdown[index] ? 'mdi-menu-up' : 'mdi-menu-down' }}</v-icon>
                      </v-btn>
                    </template>

                    <v-list>
                      <v-list-item
                        v-if="canReplace(orgPerson)"
                        :id="`officer-${index}-replace-btn`"
                        @click="emitReplace(index); dropdown[index]=false"
                      >
                        <v-list-item-subtitle>
                          <v-icon
                            size="24"
                            class="my-n1"
                          >mdi-swap-horizontal</v-icon>
                          <span class="ml-1">Replace</span>
                        </v-list-item-subtitle>
                      </v-list-item>
                    </v-list>
                  </v-menu>
                </span>
              </div>
            </v-col>
            <v-col
              v-else
              cols="1"
            />
          </template>
        </v-row>
      </template>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Emit, Mixins, Prop } from 'vue-property-decorator'
import { Getter } from 'pinia-class'
import { BaseAddress } from '@bcrs-shared-components/base-address'
import OrgPerson from './OrgPerson.vue'
import { CommonMixin, OrgPersonMixin } from '@/mixins/'
import { IsSame } from '@/utils/'
import { OrgPersonIF } from '@/interfaces/'
import { useStore } from '@/store/store'
import { AuthorizedActions } from '@/enums'

@Component({
  components: {
    DeliveryAddress: BaseAddress,
    MailingAddress: BaseAddress,
    OrgPerson
  }
})
export default class ListPeopleAndRoles extends Mixins(CommonMixin, OrgPersonMixin) {
  // Declaration for template
  readonly IsSame = IsSame

  /** Whether to render the OrgPersonForm (for edit or add). */
  @Prop() readonly renderOrgPersonForm!: boolean

  /** The current org/person to edit or add. */
  @Prop() readonly currentOrgPerson!: OrgPersonIF

  /** The index of the org/person to edit, or NaN to add. */
  @Prop() readonly activeIndex!: number

  @Prop({ default: false }) readonly isSummaryView!: boolean

  /** Whether to perform validation. */
  @Prop({ default: false }) readonly validate!: boolean

  /** Whether OrgPersons list is valid. */
  @Prop({ default: true }) readonly validOrgPersons!: boolean

  @Prop({ default: true }) readonly showDeliveryAddressColumn!: boolean
  @Prop({ default: true }) readonly showRolesColumn!: boolean
  @Prop({ default: false }) readonly showEmailColumn!: boolean
  @Prop({ default: true }) readonly showEmailUnderName!: boolean

  // Store getter
  @Getter(useStore) getEditLabel!: string
  @Getter(useStore) getOrgPeople!: OrgPersonIF[]
  @Getter(useStore) showChangeButtonForSoleProps!: boolean
  @Getter(useStore) isAlterationFiling!: boolean
  @Getter(useStore) isCoopCorrectionFiling!: boolean
  @Getter(useStore) isCorpCorrectionFiling!: boolean
  @Getter(useStore) isCorrectionFiling!: boolean
  @Getter(useStore) isFirmChangeFiling!: boolean
  @Getter(useStore) isFirmConversionFiling!: boolean
  @Getter(useStore) isFirmCorrectionFiling!: boolean
  @Getter(useStore) isLimitedRestorationExtension!: boolean
  @Getter(useStore) isLimitedRestorationToFull!: boolean

  /** V-model for dropdown menus. */
  dropdown: Array<boolean> = []

  /** Headers for the person table. */
  get tableHeaders (): Array<string> {
    const headers = ['Name', 'Mailing Address']
    if (this.showDeliveryAddressColumn) headers.push('Delivery Address')
    if (this.showRolesColumn) headers.push('Roles')
    if (this.showEmailColumn) headers.push('Email Address')
    return headers
  }

  /** This component's validity state (for error styling). */
  get isComponentInvalid (): boolean {
    // if prompted by app or if this is a single-page correction filing
    // and
    // if form is open or if list is invalid
    return (this.validate || this.isCorrectionFiling) && (this.renderOrgPersonForm || !this.validOrgPersons)
  }

  /** The current orgPersons list. */
  get currentPeopleAndRoles (): Array<OrgPersonIF> {
    if (this.isSummaryView) {
      // return list without REMOVED org-persons
      return this.getOrgPeople.filter(orgPerson => !this.wasRemoved(orgPerson))
    }
    return this.getOrgPeople
  }

  /** Whether to show table headers. */
  get showHeaders (): boolean {
    // true if there are any org-persons to display
    // include REMOVED as per currentPeopleAndRoles above
    // exclude replaced-removed
    return this.currentPeopleAndRoles.some(orgPerson => !(this.wasReplaced(orgPerson) && this.wasRemoved(orgPerson)))
  }

  /** Returns True if the specified org-person can be removed. */
  canRemove (orgPerson: OrgPersonIF): boolean {
    if (this.isAlterationFiling) {
      // alterations don't use this component
      return false
    }
    if (this.isFirmChangeFiling) {
      // can only remove partner
      return this.hasRolePartner(orgPerson)
    }
    if (this.isFirmConversionFiling) {
      return true
    }
    if (this.isCorpCorrectionFiling) {
      return true
    }
    if (this.isCoopCorrectionFiling) {
      return true
    }
    if (this.isFirmCorrectionFiling) {
      // cannot remove proprietor/partner
      return false
    }
    if (this.isLimitedRestorationToFull || this.isLimitedRestorationExtension) {
      return true
    }
    return false // should never happen
  }

  /** Returns True if the specified org-person can be replaced. */
  canReplace (orgPerson: OrgPersonIF): boolean {
    // need to be authorized
    if (this.IsAuthorized(AuthorizedActions.FIRM_REPLACE_PERSON)) {
      // change filing only
      if (this.isFirmChangeFiling) {
        // proprietor-org only
        if (this.hasRoleProprietor(orgPerson) && this.isPartyTypeOrg(orgPerson)) {
          // don't show for the replaced-added item
          // (we shouldn't see the replaced-removed item in this component)
          if (!this.wasReplaced(orgPerson)) {
            return true
          }
        }
      }
    }
    return false
  }

  /**
   * Returns the org or person full name for display.
   * @param orgPerson the org/person object
   * @returns the formatted name
   */
  formatName (orgPerson: OrgPersonIF): string {
    if (orgPerson?.officer?.organizationName) {
      return orgPerson.officer.organizationName
    }
    if (orgPerson?.officer) {
      return this.formatFullName(orgPerson.officer)
    }
    return ''
  }

  /**
   * Emits an event and index to the parent to handle undoing.
   * @param index the index of the org/person to undo
   */
  @Emit('undo')
  // eslint-disable-next-line @typescript-eslint/no-unused-vars
  emitUndo (index: number): void {}

  /**
   * Emits an event and index to the parent to start editing.
   * @param index the index of the org/person to edit
   */
  @Emit('initEdit')
  // eslint-disable-next-line @typescript-eslint/no-unused-vars
  emitInitEdit (index: number): void {}

  /**
   * Emits an event and index to the parent to handle removal.
   * @param index the index of the org/person to remove
   */
  @Emit('remove')
  // eslint-disable-next-line @typescript-eslint/no-unused-vars
  emitRemove (index: number): void {}

  /**
   * Emits an event and index to the parent to handle replacement.
   * @param index the index of the org/person to repalce
   */
  @Emit('replace')
  // eslint-disable-next-line @typescript-eslint/no-unused-vars
  emitReplace (index: number): void {}

  /**
   * Emits an event and org/person object to the parent handle adding or editing.
   * @param person the data object of the org/person to add or edit
   */
  @Emit('addEdit')
  // eslint-disable-next-line @typescript-eslint/no-unused-vars
  emitAddEdit (person: OrgPersonIF): void {}

  /**
   * Emits an event to the parent to reset the state.
   */
  @Emit('reset')
  emitReset (): void {}
}
</script>

<style lang="scss" scoped>
@import '@/assets/styles/theme.scss';

.people-roles-invalid-message {
  padding: 1.25rem;
  font-weight: bold;
  color: $BCgovABlue2;
}

.people-roles-list-header {
  // NB: same styles as v-data-table header
  color: $gray9;
  font-size: $px-14;
  font-weight: bold;

  // Apply cols="4" styling on looped elements
  .summary-cols:not(:first-child) {
    -webkit-box-flex: 0;
    -ms-flex: 0 0 33.3333333333%;
    flex: 0 0 33.3333333333%;
    max-width: 33.3333333333%;
  }
}

.people-roles-content {
  margin: 0 -28px;
  border-top: 1px solid $gray3;

  p {
    font-size: $px-14;
  }

  .people-roles-title {
    color: $gray9;
    font-weight: bold;
  }

  .peoples-roles-mailing-address,
  .peoples-roles-delivery-address {
    font-size: $px-15;
  }

  .actions {
    position: absolute;
    right: 0;
    margin-top: -0.25rem;

    @extend .actions-width;

    .v-btn + .v-btn {
      margin-left: 0.5rem;
    }

    .more-actions-btn {
      padding: 0;
      min-width: 28px;
    }
  }

  .dropdown-action {
    border-left: 1px solid $gray3;
  }
}

.actions-width {
  min-width: 150px;
  max-width: 150px;
}

.summary-view {
  margin: 0 !important;
  padding: 0.75rem 0rem 0.75rem !important;
}

.v-list-item {
  min-height: 0;
  padding: 0.5rem 1rem;
}

.v-list-item__subtitle {
  color: var(--v-primary-base) !important;

  .v-icon {
    color: var(--v-primary-base) !important;
  }
}

.col {
  .col-roles {
    padding: 0!important;
    max-width: 6rem;
  }
}

.warning-text {
  position: relative;
  top: 2px;
  left: 2px;
  color: $BCgovInputError;
}

.removed {
  opacity: .4;
}

:deep() {
  .v-chip {
    opacity: 1 !important;
  }

  // align badge icon with text
  .badges .v-icon {
    margin-top: 4px;
  }
}
</style>
