<template>
  <div id='correction'>

    <confirm-dialog
      ref="confirm"
      attach="#correction"
    />

     <load-correction-dialog
      :dialog="loadCorrectionDialog"
      @exit="navigateToDashboard"
      attach="#correction"
    />

    <resume-error-dialog
      :dialog="resumeErrorDialog"
      @exit="navigateToDashboard"
      attach="#correction"
    />

    <save-error-dialog
      filing="Correction"
      :dialog="saveErrorDialog"
      :disableRetry="busySaving"
      :errors="saveErrors"
      :warnings="saveWarnings"
      @exit="navigateToDashboard"
      @retry="onClickFilePay"
      @okay="resetErrors"
      attach="#correction"
    />

    <payment-error-dialog
      :dialog="paymentErrorDialog"
      @exit="navigateToDashboard"
      attach="#correction"
    />

   <v-container id="standalone-office-address-container" class="view-container" >
      <v-row>
        <v-col cols="12" lg="9">
          <article id="standalone-office-address-article">

            <section>
              <header>
                <h1 id="filing-header">Address Correction</h1>
                <P></P>
              </header>

              <header>
                <h2 id="correction-step-1-header">1. Detail</h2>
                <p>Enter a detail that will appear on the ledger for this entity.</p>
                <p class="black--text mb-0">{{defaultComment}}</p>
              </header>
              <detail-comment
                v-model="detailComment"
                placeholder="Add a Detail that will appear on the ledger for this entity."
                :maxLength="maxDetailCommentLength"
                @valid="detailCommentValid=$event"
              />
            <P></P>
            </section>

            <header>
              <h2 id="filing-header">2. Correct Address</h2>

              <!-- Address change-->
              <p>
                <span v-if="isCoop()">Please change your Registered Office Address.</span>
                <span v-if="isBComp()">Please change your Registered Office Address and Records Address.</span>
              </p>

              <v-alert type="info" outlined
                v-if="isBComp()"
                icon="mdi-information"
                class="white-background"
              >

                <span>Any address update will be effective tomorrow.</span>
              </v-alert>
            </header>

            <!-- Office Addresses -->
            <section>
              <office-addresses
                :addresses.sync="addresses"
                :registeredAddress.sync="registeredAddress"
                :recordsAddress.sync="recordsAddress"
                @modified="officeModifiedEventHandler($event)"
                @valid="officeAddressFormValid = $event"
              />
            </section>

            <!-- Certify -->
            <section>
              <header>
                <h2 id="AR-step-4-header">3. Certify</h2>
                <p>Enter the legal name of the person authorized to complete and submit this
                  Address Change.</p>
              </header>
              <certify
                :isCertified.sync="isCertified"
                :certifiedBy.sync="certifiedBy"
                :entityDisplay="displayName()"
                :message="certifyText(FilingCodes.ADDRESS_CHANGE_OT)"
                @valid="certifyFormValid=$event"
              />
            </section>

            <!-- Staff Payment -->
            <section v-if="isRoleStaff">
              <header>
                <h2 id="AR-step-5-header">4. Staff Payment</h2>
              </header>
              <staff-payment
                :routingSlipNumber.sync="routingSlipNumber"
                :isPriority.sync="isPriority"
                :isWaiveFees.sync="isWaiveFees"
                @valid="staffPaymentFormValid=$event"
              />
            </section>

          </article>
        </v-col>

        <v-col cols="12" lg="3" style="position: relative">
          <!-- Total Fee Display -->
          <aside>
            <affix
              relative-element-selector="#correction-article"
              :offset="{ top: 120, bottom: 40 }"
            >
              <sbc-fee-summary
                :filingData="[...filingData]"
                :payURL="payAPIURL"
                @total-fee="totalFee=$event"
              />
            </affix>
          </aside>
          </v-col>

      </v-row>
   </v-container>

    <!-- FUTURE: this container should have some container class not 'list-item' class -->
    <v-container id="standalone-office-address-buttons-container" class="list-item">

      <div class="buttons-right">
        <v-tooltip top color="#3b6cff">
          <template v-slot:activator="{ on }">
            <div v-on="on" class="d-inline">
            <v-btn
              id="coa-file-pay-btn"
              color="primary"
              large
              :disabled="!validated || busySaving"
              :loading="filingPaying"
              @click="onClickFilePay()"
            >
              <span>{{isPayRequired ? "File &amp; Pay" : "File"}}</span>
            </v-btn>
            </div>
          </template>
          <span>Ensure all of your information is entered correctly before you File.<br>
            There is no opportunity to change information beyond this point.</span>
        </v-tooltip>

        <v-btn
          id="coa-cancel-btn"
          large
          :disabled="busySaving"
          @click="navigateToDashboard()"
        >
          <span>Cancel</span>
        </v-btn>
      </div>

    </v-container>

  </div>
</template>

<script lang="ts">
// Libraries
import { Component, Mixins, Vue, Prop, Watch, Emit } from 'vue-property-decorator'
import axios from '@/axios-auth'
import { mapActions, mapState, mapGetters } from 'vuex'
import { required, maxLength } from 'vuelidate/lib/validators'
import { BAD_REQUEST, PAYMENT_REQUIRED } from 'http-status-codes'

// Dialogs
import {
  ConfirmDialog,
  PaymentErrorDialog,
  ResumeErrorDialog,
  LoadCorrectionDialog,
  SaveErrorDialog,
  WarningPopover
} from '@/components/dialogs'

// Components
import BaseAddress from 'sbc-common-components/src/components/BaseAddress.vue'

// Common components
import { Certify, DetailComment, OfficeAddresses, StaffPayment } from '@/components/common'
import SbcFeeSummary from 'sbc-common-components/src/components/SbcFeeSummary.vue'

// Mixins
import { CommonMixin, DateMixin, FilingMixin, ResourceLookupMixin } from '@/mixins'

// Enums
import { EntityTypes, FilingCodes, FilingNames, FilingStatus, FilingTypes } from '@/enums'

export default {
  name: 'CorrectOfficeAddress',

  components: {
    DetailComment,
    OfficeAddresses,
    SbcFeeSummary,
    Certify,
    StaffPayment,
    ConfirmDialog,
    LoadCorrectionDialog,
    PaymentErrorDialog,
    ResumeErrorDialog,
    SaveErrorDialog
  },
  mixins: [CommonMixin, FilingMixin, ResourceLookupMixin],

  data () {
    return {
      // Filing data
      addresses: null,
      officeAddressFormValid: true,
      filingId: null,
      // filingId: 0, // id of this correction filing
      correctedFilingId: 0, // id of filing to correct
      origFiling: null, // copy of original filing

      // properties for DetailComment component
      detailComment: '',
      detailCommentValid: null,

      // properties for Certify component
      certifiedBy: '',
      isCertified: false,
      certifyFormValid: null,

      // properties for StaffPayment component
      routingSlipNumber: '',
      isPriority: false,
      isWaiveFees: false,
      staffPaymentFormValid: null,
      totalFee: 0,

      // flags for displaying dialogs
      loadCorrectionDialog: false,
      resumeErrorDialog: false,
      saveErrorDialog: false,
      paymentErrorDialog: false,

      // other program state
      dataLoaded: false,
      showLoading: false,
      loadingMessage: 'Loading...', // initial generic message

      saving: false,
      savingResuming: false,
      filingPaying: false,
      haveChanges: false,
      saveErrors: [],
      saveWarnings: [],

      // enums
      EntityTypes,
      FilingCodes,
      FilingNames,
      FilingStatus,
      FilingTypes
    }
  },

  computed: {
    ...mapState(['currentDate', 'entityType', 'entityName', 'entityIncNo',
      'entityFoundingDate', 'registeredAddress', 'recordsAddress', 'filingData']),

    ...mapGetters(['isRoleStaff']),

    // Returns default comment (ie, the first line of the detail comment). */
    defaultComment (): string {
      return `Correction for ${this.title}. Filed on ${this.originalFilingDate}.`
    },

    validated (): boolean {
      const staffPaymentValid = (!this.isRoleStaff || !this.isPayRequired || this.staffPaymentFormValid)
      const filingDataValid = (this.filingData.length > 0)

      return (
        staffPaymentValid &&
        this.detailCommentValid &&
        this.certifyFormValid &&
        this.officeAddressFormValid &&
        filingDataValid)
    },

    busySaving () {
      return (this.saving || this.savingResuming || this.filingPaying)
    },

    // Returns maximum length of detail comment. */
    maxDetailCommentLength (): number {
      // = (max size in db) - (default comment length) - (Carriage Return)
      return 4096 - this.defaultComment.length - 1
    },

    payAPIURL () {
      return sessionStorage.getItem('PAY_API_URL')
    },

    isPayRequired () {
      // FUTURE: modify rule here as needed
      return (this.totalFee > 0)
    },

    // Returns title of original filing.
    title (): string {
      if (this.origFiling && this.origFiling.header && this.origFiling.header.name) {
        return this.typeToTitle(this.origFiling.header.name, this.agmYear)
      }
      return ''
    },

    /** Returns True if loading container should be shown, else False. */
    showLoadingContainer (): boolean {
      return !this.dataLoaded && !this.loadCorrectionDialog
    },

    /** Returns AGM Year of original filing (AR only). */
    agmYear (): number | null {
      if (this.origFiling && this.origFiling.annualReport && this.origFiling.annualReport.annualReportDate) {
        const date: string = this.origFiling.annualReport.annualReportDate
        return +date.slice(0, 4)
      }
      return null
    },

    /** Returns date of original filing in format "yyyy-mm-dd". */
    originalFilingDate (): string | null {
      if (this.origFiling && this.origFiling.header && this.origFiling.header.date) {
        const localDateTime: string = this.convertUTCTimeToLocalTime(this.origFiling.header.date)
        return localDateTime.split(' ')[0]
      }
      return null
    }
  },

  /** Called when component is mounted. */
  mounted (): void {
    // always include correction code
    // use default Priority and Waive Fees flags
    // this.updateFilingData('add', FilingCodes.CORRECTION, this.isPriority, this.isWaiveFees)
    // OfficeAddresses.call('editAddress')
  },

  methods: {
    ...mapActions(['setFilingData']),

    formatAddress (address) {
      return {
        'actions': address.actions || [],
        'addressCity': address.addressCity || '',
        'addressCountry': address.addressCountry || '',
        'addressRegion': address.addressRegion || '',
        'addressType': address.addressType || '',
        'deliveryInstructions': address.deliveryInstructions || '',
        'postalCode': address.postalCode || '',
        'streetAddress': address.streetAddress || '',
        'streetAddressAdditional': address.streetAddressAdditional || ''
      }
    },

    /** Fetches the draft correction filing. */
    async fetchDraftFiling (): Promise<void> {
      const url: string = this.entityIncNo + '/filings/' + this.filingId
      await axios.get(url).then(res => {
        if (res && res.data) {
          const filing: any = res.data.filing
          try {
            // verify data
            if (!filing) throw new Error('missing filing')
            if (!filing.header) throw new Error('missing header')
            if (!filing.business) throw new Error('missing business')
            if (!filing.correction) throw new Error('missing correction')
            if (filing.header.name !== FilingTypes.CORRECTION) throw new Error('invalid filing type')
            if (filing.header.status !== FilingStatus.DRAFT) throw new Error('invalid filing status')
            if (filing.business.identifier !== this.entityIncNo) throw new Error('invalid business identifier')
            if (filing.business.legalName !== this.entityName) throw new Error('invalid business legal name')

            // load Certified By but not Date
            this.certifiedBy = filing.header.certifiedBy

            // load Staff Payment properties
            this.routingSlipNumber = filing.header.routingSlipNumber
            this.isPriority = filing.header.priority
            this.isWaiveFees = filing.header.waiveFees

            // load Detail Comment, removing the first line (default comment)
            const comment: string = filing.correction.comment || ''
            this.detailComment = comment.split('\n').slice(1).join('\n')
          } catch (err) {
            // eslint-disable-next-line no-console
            console.log(`fetchDraftFiling() error - ${err.message}, filing = ${filing}`)
            this.resumeErrorDialog = true
          }
        } else {
          // eslint-disable-next-line no-console
          console.log('fetchDraftFiling() error - invalid response =', res)
          this.resumeErrorDialog = true
        }
      }).catch(err => {
        // eslint-disable-next-line no-console
        console.error('fetchDraftFiling() error =', err)
        this.resumeErrorDialog = true
      })
    },

    /** Fetches the original filing to correct. */
    async fetchOrigFiling (): Promise<void> {
      const url: string = this.entityIncNo + '/filings/' + this.correctedFilingId
      await axios.get(url).then(res => {
        if (res && res.data) {
          this.origFiling = res.data.filing
          try {
            // verify data
            if (!this.origFiling) throw new Error('missing filing')
            if (!this.origFiling.header) throw new Error('missing header')
            if (!this.origFiling.business) throw new Error('missing business')
            if (this.origFiling.header.status !== FilingStatus.COMPLETED) throw new Error('invalid filing status')
            if (this.origFiling.business.identifier !== this.entityIncNo) throw new Error('invalid business identifier')
            if (this.origFiling.business.legalName !== this.entityName) throw new Error('invalid business legal name')

            // FUTURE:
            // use original Certified By name
            // this.certifiedBy = this.origFiling.header.certifiedBy || ''
          } catch (err) {
            // eslint-disable-next-line no-console
            console.log(`fetchOrigFiling() error - ${err.message}, origFiling =${this.origFiling}`)
            this.loadCorrectionDialog = true
          }
        } else {
          // eslint-disable-next-line no-console
          console.log('fetchOrigFiling() error - invalid response =', res)
          this.loadCorrectionDialog = true
        }
      }).catch(error => {
        // eslint-disable-next-line no-console
        console.error('fetchOrigFiling() error =', error)
        this.loadCorrectionDialog = true
      })
    },

    /** Handler for Save click event. */
    async onClickSave (): Promise<void> {
      // prevent double saving
      if (this.busySaving) return

      this.saving = true
      const filing: any = await this.saveFiling(true)
      if (filing) {
        // save Filing ID for future PUTs
        this.filingId = +filing.header.filingId // number
      }
      this.saving = false
    },

    /** Handler for Save & Resume click event. */
    async onClickSaveResume (): Promise<void> {
      // prevent double saving
      if (this.busySaving) return

      this.savingResuming = true
      const filing: any = await this.saveFiling(true)
      // on success, route to Home URL
      if (filing) {
        this.$router.push('/')
      }
      this.savingResuming = false
    },

    /** Handler for File & Pay click event. */
    async onClickFilePay (): Promise<void> {
      // prevent double saving
      if (this.busySaving) return

      this.filingPaying = true
      const filing: any = await this.saveFiling(false) // not a draft

      // on success, redirect to Pay URL
      if (filing && filing.header) {
        const filingId: number = +filing.header.filingId

        // if this is a regular user, redirect to Pay URL
        if (!this.isRoleStaff) {
          const paymentToken: string = filing.header.paymentToken
          const baseUrl: string = sessionStorage.getItem('BASE_URL')
          const returnURL: string = encodeURIComponent(baseUrl + 'dashboard?filing_id=' + filingId)
          const authUrl: string = sessionStorage.getItem('AUTH_URL')
          const payURL: string = authUrl + 'makepayment/' + paymentToken + '/' + returnURL

          // assume Pay URL is always reachable
          // otherwise, user will have to retry payment later
          window.location.assign(payURL)
        } else {
          // route directly to dashboard
          this.$router.push('/dashboard?filing_id=' + filingId)
        }
      }
      this.filingPaying = false
    },

    /** Actually saves the filing. */
    async saveFiling (isDraft): Promise<any> {
      this.resetErrors()

      const hasPendingFilings: boolean = await this.hasTasks(this.entityIncNo)
      if (hasPendingFilings) {
        this.saveErrors = [
          { error: 'Another draft filing already exists. Please complete it before creating a new filing.' }
        ]
        this.saveErrorDialog = true
        return null
      }

      const header: any = {
        header: {
          name: 'correction',
          certifiedBy: this.certifiedBy,
          email: 'no_one@never.get',
          date: this.currentDate
        }
      }
      // only save Routing Slip Number if it's valid
      if (this.routingSlipNumber && !this.isWaiveFees) {
        header.header['routingSlipNumber'] = this.routingSlipNumber
      }
      // only save Priority it it's valid
      if (this.isPriority && !this.isWaiveFees) {
        header.header['priority'] = true
      }
      // only save Waive Fees if it's valid
      if (this.isWaiveFees) {
        header.header['waiveFees'] = true
      }

      const business: any = {
        business: {
          foundingDate: this.entityFoundingDate,
          identifier: this.entityIncNo,
          legalName: this.entityName,
          legalType: this.entityType
        }
      }

      const correction: any = {
        correction: {
          correctedFilingId: this.correctedFilingId,
          correctedFilingType: this.origFiling.header.name,
          correctedFilingDate: this.originalFilingDate,
          comment: `${this.defaultComment}\n${this.detailComment}`
        }
      }

      // build filing data
      // NB: a correction to a correction is to the original data
      const data: any = {
        filing: Object.assign(
          {},
          header,
          business,
          correction
          // FUTURE: don't enable this until API is ready for it
          // this.origFiling.annualReport || {},
          // this.origFiling.changeOfDirectors || {},
          // this.origFiling.changeOfAddress || {}
        )
      }

      if (this.filingId > 0) {
        // we have a filing id, so we are updating an existing filing
        let url: string = this.entityIncNo + '/filings/' + this.filingId
        if (isDraft) {
          url += '?draft=true'
        }
        let filing: any = null
        await axios.put(url, data).then(res => {
          if (!res || !res.data || !res.data.filing) {
            throw new Error('invalid API response')
          }
          filing = res.data.filing
          this.haveChanges = false
        }).catch(error => {
          if (error && error.response && error.response.status === PAYMENT_REQUIRED) {
            this.paymentErrorDialog = true
          } else if (error && error.response && error.response.status === BAD_REQUEST) {
            if (error.response.data.errors) {
              this.saveErrors = error.response.data.errors
            }
            if (error.response.data.warnings) {
              this.saveWarnings = error.response.data.warnings
            }
            this.saveErrorDialog = true
          } else {
            this.saveErrorDialog = true
          }
        })
        return filing
      } else {
        // filing id is 0, so we are saving a new filing
        let url: string = this.entityIncNo + '/filings'
        if (isDraft) {
          url += '?draft=true'
        }
        let filing: any = null
        await axios.post(url, data).then(res => {
          if (!res || !res.data || !res.data.filing) {
            throw new Error('invalid API response')
          }
          filing = res.data.filing
          this.haveChanges = false
        }).catch(error => {
          if (error && error.response && error.response.status === PAYMENT_REQUIRED) {
            this.paymentErrorDialog = true
          } else if (error && error.response && error.response.status === BAD_REQUEST) {
            if (error.response.data.errors) {
              this.saveErrors = error.response.data.errors
            }
            if (error.response.data.warnings) {
              this.saveWarnings = error.response.data.warnings
            }
            this.saveErrorDialog = true
          } else {
            this.saveErrorDialog = true
          }
        })
        return filing
      }
    },

    /** Reset all error flags/states. */
    resetErrors (): void {
      this.saveErrorDialog = false
      this.saveErrors = []
      this.saveWarnings = []
    },

    /** Handler for dialog Exit click events. */
    navigateToDashboard (): void {
      this.haveChanges = false
      this.dialog = false
      this.$router.push('/dashboard')
    },

    /**
    * Callback method for the "modified" event from OfficeAddresses component.
    *
    * @param modified a boolean indicating whether or not the office address(es) have been modified from their
    * original values.
    */
    officeModifiedEventHandler (modified: boolean): void {
      this.haveChanges = true
      // when addresses change, update filing data
      // use default Priority and Waive Fees flags
      this.updateFilingData(modified ? 'add' : 'remove', FilingCodes.ADDRESS_CHANGE_OT,
        this.isPriority, this.isWaiveFees)
    }
  }
}

</script>

<style lang="scss" scoped>
@import '@/assets/styles/theme.scss';

article {
  .v-card {
    line-height: 1.2rem;
    font-size: 0.875rem;
  }
}

header p,
section p {
  color: $gray6;
}

section + section {
  margin-top: 3rem;
}

h1 {
  margin-bottom: 1.25rem;
  line-height: 2rem;
  letter-spacing: -0.01rem;
}

h2 {
  margin-bottom: 0.25rem;
  margin-top: 3rem;
  font-size: 1.125rem;
}

// Save & Filing Buttons
#standalone-office-address-buttons-container {
  padding-top: 2rem;
  border-top: 1px solid $gray5;

  .buttons-left {
    width: 50%;
  }

  .buttons-right {
    margin-left: auto;
  }

  .v-btn + .v-btn {
    margin-left: 0.5rem;
  }

  #coa-cancel-btn {
    margin-left: 0.5rem;
  }
}
</style>
