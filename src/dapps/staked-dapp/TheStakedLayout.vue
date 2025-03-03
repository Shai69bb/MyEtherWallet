<template>
  <!--
    ===================================================
    The Staked Layout
    ===================================================
    -->
  <the-wrapper-dapp
    :is-new-header="true"
    :dapp-img="headerImg"
    :banner-text="header"
    :tab-items="tabs"
    :active-tab="activeTab"
    external-contents
    :on-tab="tabChanged"
    :valid-networks="validNetworks"
  >
    <!--
    ===================================================
    The Staked Header/Banner
    ===================================================
    -->
    <template #HeaderBody>
      <v-divider class="textMedium my-3" />
      <div class="d-flex flex-wrap align-center justify-center">
        <div class="text-uppercase textMedium--text font-weight-bold">
          Total Staked:
          <span class="greenPrimary--text">{{ totalStaked }}</span>
        </div>
        <v-icon color="textMedium">mdi-circle-medium</v-icon>
        <div class="text-uppercase textMedium--text font-weight-bold">
          Current APR:
          <span class="greenPrimary--text">{{ currentAprFormatted }}</span>
        </div>
      </div>
      <!--
    ===================================================
    Menu tabs
    ===================================================
    -->
      <v-btn-toggle
        v-model="activeTab"
        class="d-flex align-center justify-center mt-3 white--text"
        mandatory
        borderless
        active-class="expandHeader font-weight-bold"
        background-color="transparent"
      >
        <v-btn
          class="px-md-9 white--text text-transform--initial"
          :class="activeTab === 0 ? 'staked-tab-active' : 'staked-tab-inactive'"
          color=""
        >
          New stake
        </v-btn>
        <v-btn
          :class="[
            'px-md-9 white--text text-transform--initial d-flex  flex-column align-center',
            activeTab === 1 ? 'staked-tab-active' : 'staked-tab-inactive'
          ]"
          color=""
        >
          <div>
            <div
              :class="[
                'white--text',
                activeTab === 1 ? 'font-weight-medium' : ''
              ]"
            >
              My stake
            </div>
            <div v-if="!loadingValidators" class="mew-caption textLight--text">
              {{ !loadingValidators ? myETHTotalStaked : '--' }}
            </div>
          </div>
        </v-btn>
      </v-btn-toggle>
    </template>
    <!--
    ===================================================
    Info Link
    ===================================================
    -->
    <template #HeaderRight>
      <div class="text-right">
        <a
          :href="getArticle('stake-eth2-mew-web')"
          target="_blank"
          class="greenPrimary--text font-weight-medium text-right"
        >
          New to staking? Learn more
          <v-icon class="ml-1" small color="greenPrimary"
            >mdi-open-in-new</v-icon
          >
        </a>
      </div>
    </template>

    <!--
    ===================================================
    New Stake Tab
    ===================================================
    -->
    <template #tabContent1>
      <v-sheet
        min-height="500px"
        max-width="700px"
        color="transparent"
        class="mx-auto"
      >
        <staked-stepper
          ref="stakedStepper"
          :current-apr="handlerStaked.apr"
          :start-provision="startProvision"
          :polling-status="pollingStatus"
          @readyToStake="sendTransaction"
        />
      </v-sheet>
    </template>
    <!--
    ===================================================
    My Stake tab
    ===================================================
    -->
    <template #tabContent2>
      <v-sheet
        min-height="500px"
        max-width="700px"
        color="transparent"
        class="py-13 mx-auto"
      >
        <staked-status
          :tx-receipt="handlerStaked.txReceipt"
          :pending-hash="pendingTxHash"
          :validators="validators"
          :loading="loadingValidators"
          :amount="amount"
          :refetch-validators="refetchValidators"
        />
      </v-sheet>
    </template>
  </the-wrapper-dapp>
</template>

<script>
import { mapGetters, mapState } from 'vuex';
import { SUPPORTED_NETWORKS } from './handlers/supportedNetworks';
import { STAKED_ROUTE } from './configsRoutes';
import {
  formatPercentageValue,
  formatFloatingPointValue
} from '@/core/helpers/numberFormatHelper';

import handlerStaked from './handlers/handlerStaked';
export default {
  name: 'TheStakedLayout',
  components: {
    TheWrapperDapp: () => import('@/dapps/TheWrapperDapp.vue'),
    StakedStepper: () => import('./components/staked-stepper/StakedStepper'),
    StakedStatus: () => import('./components/StakedStatus')
  },
  data() {
    return {
      validNetworks: SUPPORTED_NETWORKS,
      headerImg: require('@/assets/images/icons/dapps/icon-dapp-stake.svg'),
      amount: 0,
      header: {
        title: 'Ethereum 2.0 staking',
        subtext:
          'Stake on Ethereum 2.0 and earn continuous rewards for providing a public good to the community.',
        subtextClass: 'textMedium--text'
      },
      activeTab: 0,
      handlerStaked: {},
      tabs: [
        {
          name: 'Stake',
          route: { name: STAKED_ROUTE.STAKED.NAME },
          id: 0
        },
        {
          name: 'Status',
          route: {
            name: STAKED_ROUTE.STATUS.NAME,
            path: STAKED_ROUTE.STATUS.PATH
          },
          id: 1
        }
      ]
    };
  },
  computed: {
    ...mapState('wallet', ['web3', 'address']),
    ...mapGetters('global', ['network']),
    ...mapGetters('article', ['getArticle']),
    /**
     * Total staked by user
     * @returns string
     */
    myETHTotalStaked() {
      return (
        formatFloatingPointValue(this.handlerStaked.myETHTotalStaked).value +
        ' ETH'
      );
    },
    /**
     * Total Staked
     * @returns string
     */
    totalStaked() {
      return (
        formatFloatingPointValue(this.handlerStaked.totalStaked).value + ' ETH'
      );
    },
    /**
     * Current APR Formatted
     * @returns string
     */
    currentAprFormatted() {
      if (this.handlerStaked.apr > 0) {
        return formatPercentageValue(this.handlerStaked.apr).value;
      }
      return '--';
    },
    /**
     * Gets the status after polling (happens on step4)
     * @returns object
     */
    pollingStatus() {
      return this.handlerStaked.pollingStatus;
    },
    /**
     * Gets the clients validators
     * @returns array
     */
    validators() {
      return this.handlerStaked.myValidators;
    },
    /**
     * Checks if validators are loading
     * @returns boolean
     */
    loadingValidators() {
      return this.handlerStaked.loadingValidators;
    },
    /**
     * Checks for pending tx hash
     * @returns string
     */
    pendingTxHash() {
      return this.handlerStaked.pendingTxHash;
    },
    isValidNetwork() {
      const chainID = this.network.type.chainID;
      const validChain = this.validNetworks.filter(
        item => item.chainID === chainID
      );
      return validChain.length > 0;
    }
  },
  watch: {
    $route() {
      this.detactUrlChangeTab();
    },
    /**
     * @watches pendingTxHash (comes after send transaction)
     * if it gets set then go to staked status
     */
    pendingTxHash(newVal) {
      if (newVal !== '') {
        this.activeTab = 1;
      }
      if (this.$refs.stakedStepper) {
        this.$refs.stakedStepper.reset();
      }
    },
    /*
    - watches for address state change
    - updates handlerStaked with new address
    - if user is currently onStep within the stakeStepper component, it will run the reset function
    */
    address(newVal) {
      this.handlerStaked.address = newVal;
      if (this.$refs.stakedStepper) {
        this.$refs.stakedStepper.reset();
      }
    }
  },
  mounted() {
    /**
     * Check url and change tab on load
     */
    this.detactUrlChangeTab();

    /**
     * Initiate Stake Handler
     */
    if (this.isValidNetwork) {
      this.handlerStaked = new handlerStaked(
        this.web3,
        this.network,
        this.address
      );
    }
  },
  methods: {
    detactUrlChangeTab() {
      const currentRoute = this.$route.name;
      if (currentRoute === STAKED_ROUTE.STATUS.NAME) {
        this.activeTab = this.tabs[1].id;
      } else {
        this.activeTab = this.tabs[0].id;
      }
    },
    tabChanged(tab) {
      this.activeTab = tab;
    },
    /**
     * Start provisioning
     */
    startProvision(params) {
      return this.handlerStaked.startProvision(params);
    },
    /**
     * Send transaction to confirm staking
     * and set amount value for staked status
     */
    sendTransaction(amountETH) {
      this.handlerStaked.sendTransaction();
      this.amount = amountETH;
    },
    /**
     * refetch validators
     */
    refetchValidators() {
      this.handlerStaked.getValidators();
    }
  }
};
</script>

<style lang="scss" scoped>
.staked-tab-inactive {
  background-color: rgba(0, 0, 0, 0.24) !important;
}
.staked-tab-active::before {
  opacity: 0 !important;
}
</style>
