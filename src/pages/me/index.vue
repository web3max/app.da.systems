<template>
  <div class="me">
    <div class="me__header">
      <LoginStatusCard />
      <Tabs
        v-model="currentTab"
        :items="tabs"
        class="me__tabs"
      />
    </div>
    <div class="me__container">
      <MyDas v-show="currentTab === ME_TABS.myDAS" />
      <Reward v-show="currentTab === ME_TABS.reward" />
    </div>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'
import { TranslateResult } from 'vue-i18n'
import { mapState } from 'vuex'
import LoginStatusCard from '~/components/cards/LoginStatusCard.vue'
import Tabs from '~/components/Tabs.vue'
import MyDas from '~/pages/me/-/MyDas.vue'
import Reward from '~/pages/me/-/Reward.vue'
import { COMMON_KEYS } from '~/store/common'
import { ME_KEYS } from '~/store/me'
import { REVERSE_KEYS } from '~/store/reverse'

const ME_TABS = {
  myDAS: 'myDAS',
  myAuction: 'myAuction',
  following: 'following',
  reward: 'reward',
  balance: 'balance'
}

export default Vue.extend({
  name: 'Me',
  components: {
    LoginStatusCard,
    Tabs,
    MyDas,
    Reward
  },
  computed: {
    ...mapState({
      me: ME_KEYS.namespace
    })
  },
  data () {
    return {
      ME_TABS,
      tabs: [
        {
          text: 'My DAS',
          value: ME_TABS.myDAS
        },
        // {
        //   text: this.$t('My Auction'),
        //   value: ME_TABS.myAuction
        // },
        // {
        //   text: this.$t('Favorites'),
        //   value: ME_TABS.following
        // },
        {
          text: 'Rewards',
          icon: 'reward',
          value: ME_TABS.reward
        },
        {
          text: 'Balance',
          value: ME_TABS.balance
        }
      ],
      currentTab: this.$route.query.tab || ME_TABS.myDAS
    }
  },
  head (): { [key: string]: string | TranslateResult } {
    return {
      title: this.$t('My')
    }
  },
  watch: {
    currentTab (newVal) {
      if (newVal) {
        this.$router.replace({
          query: {
            tab: newVal
          }
        })
      }
    }
  },
  mounted () {
    this.$store.dispatch(COMMON_KEYS.fetchConfig)
    this.$store.dispatch(REVERSE_KEYS.fetchDasReverse)
  }
})
</script>

<style lang="scss" scoped>
@import "src/assets/variables";

.me {
  flex: 1;
  background: #F7F8F9;
}

.me__header {
  padding: 8px 12px 12px 12px;
  background: $white;
  box-shadow: 0 2px 5px 0 rgb(0 0 0 / 5%);
  border-radius: 0 0 22px 22px;
}

.me__tabs {
  margin-top: 16px;
}

.me__container {
  position: relative;
  padding: 0 12px;
}
</style>
