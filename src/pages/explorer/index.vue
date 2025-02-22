<template>
  <div class="explorer">
    <PublicBetaTips v-if="!config.isProdData" />
    <div class="explorer__lang-switcher">
      <LangSwitcher v-if="isMobile" />
    </div>
    <img class="explorer__logo" src="/images/explorer/das-logo.png" alt="logo">
    <div class="explorer__full-name">
      {{ $t('Decentralized Account Systems') }}
      <span
        v-if="!config.isProdData"
        class="explorer__full-name__beta"
      >
        {{ $t('(Public Beta)') }}
      </span>
    </div>
    <ExplorerSearch
      v-model.trim="searchWord"
      :placeholder="$t('Find your perfect DAS account')"
      @input="onInput"
      @search="onSearch"
    />
    <div
      v-if="showIncorrectAccountFormat"
      class="explorer__error-tip"
    >
      {{ $t('Account names can only contain lowercase letters, numbers and partial Emoji') }}
      <a
        class="explorer__rules-details"
        :href="$i18n.locale === 'zh' ? 'https://docs.da.systems/docs/v/chinese-1/zhu-ce-das/charsets' : 'https://docs.da.systems/docs/register-das/charsets'"
        :target="isMobile ? '_self' : '_blank'"
      >
        {{ $t('[Rules Details]') }}
      </a>
    </div>
    <div
      v-else-if="showIllegalStringLength"
      class="explorer__error-tip"
    >
      {{ $t('Account name length must be {min} to {max} characters', { min: commonConfig.min_account_len, max: commonConfig.max_account_len }) }}
    </div>
    <div
      v-else-if="notOpenForRegistrationShowing"
      class="explorer__error-tip"
    >
      {{ $t('Try another one. This account is not open for registration yet') }}
    </div>
    <div class="explorer__search_result">
      <ul
        v-if="searchResult.account"
        class="explorer__search_result__list"
      >
        <AccountStatus
          :account="searchResult"
          :loading="loading"
        />
      </ul>
    </div>
    <div class="explorer__open-registration-rules">
      <div class="explorer__open-registration-rules__title">
        {{ $t('Updates for the release:') }}
      </div>
      <div>{{ $t('10 characters and above: All released;') }}</div>
      <div>{{ $t('4 ~ 9 characters: Randomly release 35%;') }}</div>
      <div>
        {{ $t('1 ~ 3 characters: Not released yet.') }}
        <a
          class="explorer__rules-details"
          :href="$i18n.locale === 'zh' ? 'https://docs.da.systems/docs/v/chinese-1/zhu-ce-das/open-registration-rules' : 'https://docs.da.systems/docs/register-das/open-registration-rules'"
          :target="isMobile ? '_self' : '_blank'"
        >
          {{ $t('Why?') }}
        </a>
      </div>
      <i18n
        path="{joinDiscord} to keep updated about the release."
        tag="div"
      >
        <a
          slot="joinDiscord"
          class="explorer__rules-details"
          href="https://discord.gg/dascommunity"
          :target="isMobile ? '_self' : '_blank'"
        >
          {{ $t('Join Discord') }}
        </a>
      </i18n>
    </div>
    <a
      v-if="!searchWord"
      class="explorer__faq"
      href="https://da.systems"
      :target="isMobile ? '_self' : '_blank'"
    >
      {{ $t('What is DAS?') }}
    </a>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'
import debounce from 'lodash.debounce'
import { mapState, mapGetters } from 'vuex'
import uts46 from 'idna-uts46-hx'
import LangSwitcher from '~/components/LangSwitcher.vue'
import ExplorerSearch from '~/pages/explorer/-/ExplorerSearch.vue'
import { ACCOUNT_STATUS, ACCOUNT_SUFFIX, CHAR_TYPE, DEBOUNCE_WAIT_TIME } from '~/constant'
import { ISearchAccount } from '~/services/Explorer'
import AccountStatus from '~/pages/explorer/-/AccountStatus.vue'
import { IConnectedAccount, ME_KEYS } from '~/store/me'
import { isMobile, splitAccount } from '~/modules/tools'
import { COMMON_KEYS } from '~/store/common'
import { IConfig } from '~/services/Common'
import PublicBetaTips from '~/components/PublicBetaTips.vue'
import config from '~~/config'
import errno from '~/constant/errno'

export default Vue.extend({
  name: 'Explorer',
  components: {
    LangSwitcher,
    ExplorerSearch,
    AccountStatus,
    PublicBetaTips
  },
  data () {
    return {
      config,
      searchWord: '',
      searchResult: {} as ISearchAccount,
      loading: false,
      showIncorrectAccountFormat: false,
      showIllegalStringLength: false,
      notOpenForRegistrationShowing: false
    }
  },
  computed: {
    isMobile,
    ...mapState({
      me: ME_KEYS.namespace,
      common: COMMON_KEYS.namespace
    }),
    ...mapGetters({
      computedChainId: ME_KEYS.computedChainId
    }),
    connectedAccount (): IConnectedAccount {
      return this.me.connectedAccount
    },
    commonConfig (): IConfig {
      return this.common.config
    }
  },
  mounted () {
    this.$store.dispatch(COMMON_KEYS.fetchConfig)
  },
  methods: {
    checkSearchWord (value: string) {
      try {
        value = uts46.toAscii(value, { useStd3ASCII: true, transitional: false, verifyDnsLength: false })
        value = uts46.toUnicode(value, { useStd3ASCII: true })
      }
      catch (err) {
        this.showIncorrectAccountFormat = true
      }

      const splitArr = splitAccount(value)
      const nonComplianceChar = splitArr.find((item: { char_set_name: number, char: string }) => {
        return item.char_set_name === CHAR_TYPE.notKnown
      })
      const accountLength = splitArr.length

      if (accountLength < this.commonConfig.min_account_len || accountLength > this.commonConfig.max_account_len) {
        this.showIllegalStringLength = true
      }
      this.showIncorrectAccountFormat = !!nonComplianceChar
    },
    onInput () {
      this.searchResult = {} as ISearchAccount
      this.showIncorrectAccountFormat = false
      this.showIllegalStringLength = false
      this.notOpenForRegistrationShowing = false
      try {
        let _searchWord = this.searchWord.toLowerCase()
        _searchWord = uts46.toAscii(_searchWord, { useStd3ASCII: true, transitional: false, verifyDnsLength: false })
        this.searchWord = uts46.toUnicode(_searchWord, { useStd3ASCII: true })
      }
      catch (err) {
        this.searchWord = this.searchWord.toLowerCase()
      }
    },
    onSearch: debounce(async function (this: any, value: string) {
      value = value.replace(/\s+/g, '')
      this.searchWord = value
      value = value.replace(/\.bit$/, '')

      this.searchResult = {}
      this.showIllegalStringLength = false
      this.showIncorrectAccountFormat = false
      this.notOpenForRegistrationShowing = false

      if (!value) {
        return
      }

      this.$ga.event('explorer', 'search', 'search account')

      this.checkSearchWord(value)

      if (this.showIncorrectAccountFormat || this.showIllegalStringLength) {
        return
      }

      this.loading = true
      this.searchResult = {
        account: this.searchWord + ACCOUNT_SUFFIX
      }

      try {
        const res = await this.$services.explorer.searchAccount({
          account: value,
          chain_type: this.computedChainId,
          address: this.connectedAccount.address
        })

        if (!res.is_self && [ACCOUNT_STATUS.registeringPaymentConfirm].includes(res.status)) {
          this.searchResult = {
            account: this.searchWord,
            ...res,
            status: ACCOUNT_STATUS.registerable
          }
        }
        else if (!res.is_self && [ACCOUNT_STATUS.registeringLockedAccount, ACCOUNT_STATUS.registering, ACCOUNT_STATUS.registeringIncludeProposal, ACCOUNT_STATUS.registeringConfirmProposal].includes(res.status)) {
          this.searchResult = {
            account: this.searchWord,
            ...res,
            status: ACCOUNT_STATUS.othersRegistering
          }
        }
        else if (res) {
          if (res.status === ACCOUNT_STATUS.notOpenRegister) {
            this.searchResult = {}
            this.notOpenForRegistrationShowing = true
          }
          else {
            this.searchResult = {
              account: this.searchWord,
              ...res
            }
          }
        }
      }
      catch (err: any) {
        console.error(err)
        this.searchResult = {}
        if (err.code === errno.rpcApiErrAccountCharsErr) {
          this.showIncorrectAccountFormat = true
        }
        else {
          this.$alert({
            title: this.$t('Error'),
            message: `${err.code}: ${err.message}`
          })
        }
      }
      finally {
        this.loading = false
      }
    }, DEBOUNCE_WAIT_TIME)
  }
})
</script>

<style lang="scss" scoped>
@import "src/assets/variables";

.explorer {
  position: relative;
  flex: 1;
  padding: 16px 16px 0 16px;
  text-align: center;
}

.explorer__lang-switcher {
  min-height: 32px;
  text-align: right;
}

.explorer__logo {
  width: 129px;
  min-height: 44px;
  margin-top: 24px;
}

.explorer__full-name {
  margin: 10px 0 44px 0;
  color: $primary-font-color;
}

.explorer__full-name__beta {
  color: $error-font-color;
}

.explorer__error-tip {
  margin-top: 4px;
  margin-left: 4px;
  font-size: 12px;
  font-weight: 600;
  color: #DF4A46;
  line-height: 14px;
  text-align: left;
}

.explorer__rules-details {
  color: $link-font-color;
}

.explorer__search_result {
  position: relative;
}

.explorer__search_result__list {
  position: absolute;
  width: 100%;
  margin-top: 12px;
  margin-bottom: 20px;
}

.explorer__faq {
  position: absolute;
  bottom: 10%;
  transform: translateX(-50%);
  color: $assist-font-color;

  &:hover {
    color: $link-font-color
  }
}

.explorer__open-registration-rules {
  margin-top: 20px;
  border-radius: 8px;
  border: 1px solid #EFF2F5;
  padding: 12px;
  text-align: left;
  font-size: 12px;
  font-weight: 400;
  color: #636D85;
  line-height: 17px;
}

.explorer__open-registration-rules__title {
  margin-bottom: 4px;
}
</style>
