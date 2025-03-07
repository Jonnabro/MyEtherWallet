<template>
  <!--
    ===================================================
    Step 4: Review & Stake
    ===================================================
    -->
  <div>
    <div class="mx-auto pb-3" style="max-width: 550px">
      <div class="mew-heading-2 py-8 text-center">Review and stake</div>
      <v-row dense>
        <!--
    ===================================================
    Stake details
    ===================================================
    -->
        <v-col
          v-for="(detail, idx) in details"
          :key="detail + idx"
          cols="12"
          sm="6"
        >
          <div class="greyLight pa-5 d-flex align-center rounded-lg">
            <div class="mr-3 mt-1">
              <img :src="detail.img" height="25" alt="eth icon" />
            </div>
            <div>
              <div
                class="textLight--text text-uppercase detail-subtitle font-weight-medium"
              >
                {{ detail.subtitle }}
              </div>
              <div class="mew-heading-4 detail-title">{{ detail.title }}</div>
              <mew-transform-hash
                v-if="detail.isAddress"
                class="mew-hash-container textLight--text font-weight-medium"
                :hash="detail.desc"
              />
              <div v-if="!detail.isAddress" class="textLight--text">
                {{ detail.desc }}
              </div>
            </div>
          </div>
        </v-col>
      </v-row>
      <!--
    ===================================================
    Fees
    ===================================================
    -->
      <div class="mt-5">
        <div
          v-for="(fee, idx) in fees"
          :key="fee + idx"
          class="d-block d-sm-flex align-center justify-space-between mb-3 mb-sm-2"
        >
          <div class="mew-caption textLight--text">{{ fee.title }}</div>
          <div>
            {{ fee.ethValue }} <span class="textLight--text">ETH /</span>
            {{ fee.fiatValue }}
          </div>
        </div>
      </div>
      <!--
    ===================================================
    Terms & Conditions
    ===================================================
    -->
      <div class="mt-11 mt-sm-10 pa-5 greyLight">
        <mew-checkbox
          v-model="secondCondition"
          dense
          class-name="textMedium--text"
          label="I understand that staking involves slashing risks and my funds can be lost."
        />
        <mew-checkbox
          v-model="thirdCondition"
          dense
          class-name="textMedium--text"
        >
          <template #contentSlot>
            <span
              >I have read and agreed to
              <a href="https://staked.us/terms/" target="_blank">Staked.us</a>
              terms of service. This Staking feature is provided by Staked.us,
              and MEW is not liable for it's services.</span
            >
          </template>
        </mew-checkbox>
      </div>

      <border-block class="mt-6 mx-md-2 text-center rounded-lg pa-6">
        <!--
      ===================================================
      Prepare to stake (Step 1)
      ===================================================
      -->
        <div v-if="stakedStep === 1">
          <div
            :class="[
              'mx-auto',
              stakedStep1Title.error ? 'redPrimary--text' : 'textMedium--text'
            ]"
            style="max-width: 300px"
          >
            {{ stakedStep1Title.message }}
            <a
              v-if="stakedStep1Title.showContactSupport"
              rel="noopener noreferrer"
              class="cursor-pointer greenPrimary--text text-lowercase"
              href="mailto:support@myetherwallet.com"
              target="_blank"
            >
              {{ $t('common.contact-support') }}
            </a>
          </div>
          <mew-button
            btn-size="xlarge"
            class="mt-3"
            title="Prepare for staking"
            :disabled="!secondCondition || !thirdCondition"
            @click.native="prepareToStake"
          ></mew-button>
        </div>
        <!--
    ===================================================
    Preparing validators (Step 2)
    ===================================================
    -->
        <div v-if="stakedStep === 2">
          <div class="mew-heading-4 font-weight-medium mb-2">
            Preparing validators
          </div>
          <div class="textMedium--text mx-auto" style="max-width: 300px">
            This usually takes ~20 seconds, in rare cases it can take up to 10
            min.
          </div>
          <div class="px-5">
            <v-progress-linear
              style="max-width: 350px"
              class="mt-4 mx-auto"
              indeterminate
              color="greenPrimary"
            ></v-progress-linear>
          </div>
        </div>
        <!--
    ===================================================
    Ready to stake (Step 3)
    ===================================================
    -->
        <div v-if="stakedStep === 3">
          <v-icon color="greenPrimary" class="mr-2">mdi-check-circle</v-icon>
          Ready to stake
        </div>
      </border-block>
      <!--
    ===================================================
    Back & Stake buttons
    ===================================================
    -->
      <div
        class="mt-10 d-flex flex-column-reverse flex-md-row align-center justify-center"
      >
        <mew-button
          :has-full-width="$vuetify.breakpoint.smAndDown"
          btn-size="xlarge"
          class="d-block ma-2"
          title="Back"
          btn-style="outline"
          @click.native="onBack"
        />
        <mew-button
          :has-full-width="$vuetify.breakpoint.smAndDown"
          btn-size="xlarge"
          class="d-block ma-2"
          :title="buttonTitle"
          :disabled="!readyToStake"
          @click.native="onReadyToStake"
        >
        </mew-button>
      </div>
    </div>
  </div>
</template>

<script>
import BigNumber from 'bignumber.js';
import { mapState, mapGetters } from 'vuex';

import { ABI_GET_FEES } from '@/dapps/staked-dapp/handlers/handlerStaked';
import { formatBalanceEthValue } from '@/core/helpers/numberFormatHelper';
import iconColorfulETH from '@/assets/images/icons/icon-colorful-eth.svg';
import configNetworkTypes from '@/dapps/staked-dapp/handlers/configNetworkTypes';

export default {
  props: {
    amount: {
      type: Number,
      default: 0
    },
    eth2Address: {
      type: String,
      default: ''
    },
    startProvision: {
      type: Function,
      default: () => {}
    },
    pollingStatus: {
      type: Object,
      default: () => {}
    }
  },
  data() {
    return {
      secondCondition: false,
      thirdCondition: false,
      readyToStake: false,
      serviceFees: {},
      stakedStep: 1,
      stakedStep1Title: {
        message:
          'We will prepare validators for you. After that you can confirm and stake your ETH.'
      }
    };
  },
  computed: {
    ...mapGetters('external', ['fiatValue']),
    ...mapGetters('global', ['network', 'getFiatValue']),
    ...mapState('wallet', ['web3']),
    ...mapGetters('global', ['gasPrice', 'network']),
    buttonTitle() {
      return `Stake ${this.amount} ETH`;
    },
    /**
     * Returns current network eth icon
     * @return string
     */
    networkImg() {
      return this.network.type.icon;
    },
    /**
     * @returns array
     * Staking details to display (including amount and eth2 address)
     */
    details() {
      return [
        {
          img: this.networkImg,
          subtitle: 'Staking',
          title: this.amount + ' ETH',
          desc: this.amountFiat
        },
        {
          img: iconColorfulETH,
          subtitle: 'Ethereum',
          title: 'Your withdrawal address',
          desc: this.eth2Address,
          isAddress: true
        }
      ];
    },
    /**
     * @returns array
     * Includes network, service fees, total
     */
    fees() {
      return [
        {
          title: 'Network fee',
          ethValue: this.networkFees.eth,
          fiatValue: this.networkFees.fiat
        },
        {
          title: 'Service fee',
          ethValue: this.serviceFees.eth,
          fiatValue: this.serviceFees.fiat
        },
        {
          title: 'Total',
          ethValue: this.totalFees.eth,
          fiatValue: this.totalFees.fiat
        }
      ];
    },
    /**
     * @returns object
     * Network fees in ETH and fiat
     */
    networkFees() {
      const gasPriceETH = formatBalanceEthValue(this.gasPrice).value;
      return {
        eth: gasPriceETH,
        fiat: this.getFiatValue(BigNumber(this.fiatValue).times(gasPriceETH))
      };
    },
    /**
     * @returns object
     * Total fees in ETH and fiat
     */
    totalFees() {
      const totalETH = new BigNumber(this.networkFees.eth)
        .plus(this.serviceFees.eth)
        .toFixed();
      return {
        eth: totalETH,
        fiat: this.getFiatValue(
          new BigNumber(this.fiatValue).times(totalETH).toFixed()
        )
      };
    },
    /**
     * @returns how many validators are needed to stake x amount of eth
     */
    validatorsCount() {
      if (this.amount) {
        return new BigNumber(this.amount).dividedBy(32).toFixed();
      }
      return 0;
    },
    /**
     * @returns eth staking amount in fiat
     */
    amountFiat() {
      return this.getFiatValue(
        new BigNumber(this.amount).times(this.fiatValue)
      );
    }
  },
  watch: {
    /**
     * Watches the status of polling
     * if successful -> then its ready to stake
     * else will error out
     */
    pollingStatus(newVal) {
      if (newVal.success) {
        this.stakedStep += 1;
        this.readyToStake = true;
      } else {
        this.stakedStep -= 1;
        if (
          newVal.error.status === 406 ||
          newVal.error.message?.includes('406')
        ) {
          this.stakedStep1Title = {
            message:
              'Oops. We don’t know how you got this far without enough ETH. Please start over and check your funds.',
            error: true
          };
        } else {
          this.stakedStep1Title = {
            message:
              'Something went wrong. Please try again in a few minutes. If the problem persists,',
            error: true,
            showContactSupport: true
          };
        }
      }
    }
  },
  mounted() {
    this.getServiceFees();
  },
  methods: {
    /**
     * @returns object
     * Gets service fees (calls contract and abi to get exact fee)
     */
    async getServiceFees() {
      const batchContract =
        configNetworkTypes.network[this.network.type.name].batchContract;
      const contract = new this.web3.eth.Contract(ABI_GET_FEES, batchContract);
      const feesWEI = await contract.methods.getFees(this.amount / 32).call();
      const feesETH = formatBalanceEthValue(feesWEI).value;
      this.serviceFees = {
        eth: feesETH,
        fiat: this.getFiatValue(BigNumber(this.fiatValue).times(feesETH))
      };
    },
    /**
     * Emits back to go to previous step
     */
    onBack() {
      this.$emit('back');
    },
    /**
     * Start provisioning and polling (step 1: prepare to stake)
     */
    prepareToStake() {
      this.stakedStep += 1;
      this.startProvision({
        eth2Address: this.eth2Address,
        count: this.validatorsCount
      });
    },
    /**
     * Sends the transaction to confirm eth stake
     */
    onReadyToStake() {
      this.$emit('readyToStake');
    }
  }
};
</script>

<style lang="scss" scoped>
.mew-hash-container {
  width: 160px;
}
.detail-subtitle {
  font-size: 0.714rem; //font-size: 10px
}
.detail-title {
  font-weight: 600;
}
</style>
