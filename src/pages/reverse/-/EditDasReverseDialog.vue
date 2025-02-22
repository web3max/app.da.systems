<template>
  <Dialog
    class="edit-das-reverse-dialog"
    :showing="showing"
    :title="$t('Edit reverse record')"
    closeButton
    @close="onClose"
  >
    <label class="edit-das-reverse-dialog__label">DAS</label>
    <SelectDas
      v-model="account"
      :placeholder="$t('Select or enter the DAS account')"
      :errorMessages="selectDasErrors"
    />
    <template v-slot:action>
      <Button
        class="edit-das-reverse-dialog__button"
        block
        success
        :disabled="!account"
        :loading="confirmLoading"
        @click="onConfirm"
      >
        {{ $t('Save') }}
      </Button>
    </template>
  </Dialog>
</template>

<script lang="ts">
import Vue from 'vue'
import { mapState, mapGetters } from 'vuex'
import Dialog from '~/components/Dialog.vue'
import SelectDas from '~/pages/reverse/-/SelectDas.vue'
import { IConnectedAccount, ME_KEYS } from '~/store/me'
import { IReverseLatestRes } from '~/services/DasReverse'
import { fromSatoshi, mmJsonHashAndChainIdHex, sleep, thousandSplit } from '~/modules/tools'
import { REVERSE_KEYS } from '~/store/reverse'
import Button from '~/components/Button.vue'
import { NEW_LOCK_SCRIPT_TYPE } from '~/constant/chain'
import errno from '~/constant/errno'
import { ACCOUNT_STATUS, ORDER_ACTION_TYPE } from '~/constant'
import { COMMON_KEYS } from '~/store/common'

export default Vue.extend({
  name: 'EditDasReverseDialog',
  components: {
    Dialog,
    SelectDas,
    Button
  },
  model: {
    prop: 'showing',
    event: 'close'
  },
  props: {
    showing: {
      type: Boolean,
      required: false
    }
  },
  data () {
    return {
      confirmLoading: false,
      account: '',
      selectDasErrors: [] as string[]
    }
  },
  computed: {
    ...mapState({
      me: ME_KEYS.namespace,
      reverse: REVERSE_KEYS.namespace,
      common: COMMON_KEYS.namespace
    }),
    ...mapGetters({
      computedChainId: ME_KEYS.computedChainId,
      computedEvmChainId: ME_KEYS.computedEvmChainId
    }),
    connectedAccount (): IConnectedAccount {
      return this.me.connectedAccount
    },
    freezeCKB (): string {
      return fromSatoshi(this.common.config.reverse_record_capacity)
    },
    dasReverse (): IReverseLatestRes {
      return this.reverse.dasReverse
    }
  },
  watch: {
    account () {
      this.selectDasErrors = []
    }
  },
  methods: {
    thousandSplit,
    onClose () {
      this.account = ''
      this.$emit('close', false)
    },
    async onConfirm () {
      this.confirmLoading = true

      try {
        const accountInfo = await this.$services.account.accountInfo(this.account)

        if (accountInfo) {
          if ([ACCOUNT_STATUS.notOpenRegister, ACCOUNT_STATUS.unavailableAccount, ACCOUNT_STATUS.reservedAccount].includes(accountInfo.status)) {
            this.selectDasErrors = [(this.$t('The account is not registered and does not support the reverse record') as string)]
            return
          }
        }

        const res = await this.$services.dasReverse.editDasReverse({
          evm_chain_id: this.computedEvmChainId,
          chain_type: this.computedChainId,
          address: this.connectedAccount.address,
          account: this.account
        })

        if (!res) {
          return
        }

        for (const item of res.sign_list) {
          if (item.sign_msg) {
            if (item.sign_type === NEW_LOCK_SCRIPT_TYPE.eip712) {
              const mmJson = JSON.parse(JSON.stringify(res.mm_json))
              mmJson.message.digest = item.sign_msg
              const signDataRes = await this.$walletSdk.signData(mmJson, true)
              item.sign_msg = signDataRes + mmJsonHashAndChainIdHex(mmJson, mmJson.domain.chainId)
            }
            else {
              const signDataRes = await this.$walletSdk.signData(item.sign_msg)
              item.sign_msg = (signDataRes as string)
            }
            // sometimes metamask need a duration to receive next signing request
            await sleep(1000)
          }
        }

        await this.$services.account.sendTrx(res)

        this.$emit('trxSubmitted', ORDER_ACTION_TYPE.editDasReverse)
        this.onClose()
      }
      catch (err: any) {
        console.error(err)
        if (![errno.metaMaskUserRejectedAccountAccess, errno.metaMaskUserDeniedMessageSignature].includes(err.code) && err !== errno.tronLinkConfirmationDeclinedByUser && err.message !== errno.walletConnectUserRejectedTheTransaction) {
          if (err.code === errno.rpcApiErrAccountFrequencyLimit) {
            this.$alert({
              title: this.$t('Tips'),
              message: this.$t('If the operation is too frequent, please try again after {timeInterval} minutes', { timeInterval: 3 })
            })
          }
          else if (err.code === errno.apiErrorCodeResolveFailed) {
            this.$alert({
              title: this.$t('Tips'),
              message: this.$t('Frequent operations. There are still transactions being processed in your wallet address, please try again after 30s.')
            })
          }
          else if (err.code === errno.apiErrorCodeReverseAlreadyExist) {
            this.$alert({
              title: this.$t('Tips'),
              message: this.$t('You have already set this account.')
            })
          }
          else if (err.code === errno.metaMaskWalletRequestPermissions) {
            this.$alert({
              title: this.$t('Tips'),
              message: this.$t('Other requests for the wallet are not processed, please try again after processing')
            })
          }
          else if (err.code === errno.rpcApiErrAccountNotExist) {
            this.selectDasErrors = [(this.$t('The account is not registered and does not support the reverse record') as string)]
          }
          else {
            this.$alert({
              title: this.$t('Error'),
              message: err.code ? `${err.code}: ${err.message}` : err
            })
          }
        }
      }
      finally {
        this.confirmLoading = false
      }
    }
  }
})
</script>

<style lang="scss" scoped>
@import "src/assets/variables";

.edit-das-reverse-dialog {

}

.edit-das-reverse-dialog__label {
  margin-top: 24px;
  margin-bottom: 8px;
  display: block;
  font-size: 14px;
  font-weight: 600;
  color: $primary-font-color;
  line-height: 16px;
}

.edit-das-reverse-dialog__insufficient-balance__tips {
  font-size: 16px;
  font-weight: 400;
  color: $primary-font-color;
  line-height: 24px;
}

.edit-das-reverse-dialog__button {
  margin-top: 140px;
}
</style>
