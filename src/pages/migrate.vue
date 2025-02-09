<template>
  <div class="container">
    <div class="page-head fs-container">
      <span class="title">Associated Token Account Migrator</span>
    </div>

    <div class="card">
      <div class="card-body">
        <p>
          If you have token accounts that need to be migrated to associated token accounts (ATA), they will show below.
          To learn more about ATAs,
          <a href="https://raydium.gitbook.io/raydium/updates/associated-token-account-migration" target="_blank"
            >click here</a
          >. <br /><br />
          Click the "Migrate Token Accounts" button below to migrate your balances. If there are several tokens below,
          you may need to do this more than once.
        </p>

        <Button v-if="!wallet.connected" size="large" ghost @click="$accessor.wallet.openModal">
          Connect Wallet
        </Button>
        <div v-else>
          <table class="ata-table">
            <thead>
              <tr>
                <th class="address">Address</th>
                <th class="balance">Balance</th>
                <th style="text-align: right">Token</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="accountInfo in wallet.auxiliaryTokenAccounts" :key="accountInfo.pubkey.toBase58()">
                {{
                  void ((address = accountInfo.pubkey.toBase58()), (info = accountInfo.account.data.parsed.info))
                }}
                <td class="flex">
                  <span>{{ address.substr(0, 4) }}...{{ address.substr(address.length - 4, 4) }}</span>
                  <div class="actions">
                    <a :href="`${url.explorer}/account/${address}`" target="_blank">
                      <Icon type="link" />
                    </a>
                  </div>
                </td>
                <td class="balance">{{ info.tokenAmount.uiAmount }}</td>
                <td class="token flex">
                  <span>{{ getTokenSymbolByMint(info.mint) }}</span>
                  <div class="actions">
                    <a :href="`${url.explorer}/token/${info.mint}`" target="_blank">
                      <Icon type="link" />
                    </a>
                  </div>
                </td>
              </tr>
            </tbody>
          </table>

          <Button
            class="migrate-token-button"
            ghost
            :loading="migrating"
            :disabled="wallet.auxiliaryTokenAccounts.length === 0 || migrating"
            @click="migrateTokens"
          >
            Migrate Token Accounts
          </Button>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'
import { mapState } from 'vuex'
import { Button, Icon } from 'ant-design-vue'

import { get, cloneDeep } from 'lodash-es'
import { mergeTokens } from '@/utils/migrate'
import { TOKENS, getTokenSymbolByMint } from '@/utils/tokens'
import { removeLiquidity } from '@/utils/liquidity'
import { getUnixTs } from '@/utils'
import { getBigNumber } from '@/utils/layouts'

export default Vue.extend({
  components: {
    Button,
    Icon
  },

  data() {
    return {
      TOKENS,
      farms: [] as any,
      liquids: [] as any,

      migrating: false,
      creating: false,
      unstaking: false,
      removing: false
    }
  },

  head: {
    title: 'Raydium Migrate'
  },

  computed: {
    ...mapState(['wallet', 'liquidity', 'farm', 'url'])
  },

  watch: {
    'farm.stakeAccounts': {
      handler() {
        this.updateFarms()
      },
      deep: true
    },
    'wallet.tokenAccounts': {
      handler() {
        this.updateLiquids()
      },
      deep: true
    }
  },

  mounted() {
    this.updateFarms()

    this.updateLiquids()
  },

  methods: {
    getBigNumber,
    get,
    getTokenSymbolByMint,

    migrateTokens() {
      this.migrating = true

      const conn = this.$web3
      const wallet = (this as any).$wallet

      const key = getUnixTs().toString()
      this.$notify.info({
        key,
        message: 'Making transaction...',
        description: '',
        duration: 0
      })

      mergeTokens(conn, wallet, this.$accessor.wallet.auxiliaryTokenAccounts, this.$accessor.wallet.tokenAccounts)
        .then((txid) => {
          this.$notify.info({
            key,
            message: 'Transaction has been sent',
            description: (h: any) =>
              h('div', [
                'Confirmation is in progress.  Check your transaction on ',
                h('a', { attrs: { href: `${this.url.explorer}/tx/${txid}`, target: '_blank' } }, 'here')
              ])
          })

          const description = `Merge token accounts`
          this.$accessor.transaction.sub({ txid, description })
        })
        .catch((error) => {
          this.$notify.error({
            key,
            message: 'Merge token accounts failed',
            description: error.message
          })
        })
        .finally(() => {
          this.migrating = false
        })
    },

    updateFarms() {
      const farms: any = []

      for (const [poolId, farmInfo] of Object.entries(this.farm.infos)) {
        // @ts-ignore
        if (!farmInfo.isStake) {
          let userInfo = get(this.farm.stakeAccounts, poolId)

          if (userInfo) {
            userInfo = cloneDeep(userInfo)

            const { depositBalance } = userInfo

            if (!depositBalance.isNullOrZero()) {
              farms.push({
                farmInfo,
                depositBalance
              })
            }
          }
        }
      }

      this.farms = farms
    },

    // createUsdtAccount() {
    //   this.creating = true

    //   const conn = this.$web3
    //   const wallet = (this as any).$wallet

    //   const key = getUnixTs().toString()
    //   this.$notify.info({
    //     key,
    //     message: 'Making transaction...',
    //     description: '',
    //     duration: 0
    //   })

    //   createTokenAccount(conn, wallet, TOKENS.USDT.mintAddress)
    //     .then((txid) => {
    //       this.$notify.info({
    //         key,
    //         message: 'Transaction has been sent',
    //         description: (h: any) =>
    //           h('div', [
    //             'Confirmation is in progress.  Check your transaction on ',
    //             h('a', { attrs: { href: `${this.url.explorer}/tx/${txid}`, target: '_blank' } }, 'here')
    //           ])
    //       })

    //       const description = `Create USDT account`
    //       this.$accessor.transaction.sub({ txid, description })
    //     })
    //     .catch((error) => {
    //       this.$notify.error({
    //         key,
    //         message: 'Create USDT account failed',
    //         description: error.message
    //       })
    //     })
    //     .finally(() => {
    //       this.creating = false
    //     })
    // },

    // unstakeWUSDT(farmInfo: any, amount: string) {
    //   this.unstaking = true

    //   const conn = this.$web3
    //   const wallet = (this as any).$wallet

    //   const lpAccount = get(this.wallet.tokenAccounts, `${farmInfo.lp.mintAddress}.tokenAccountAddress`)
    //   const rewardAccount = get(this.wallet.tokenAccounts, `${farmInfo.reward.mintAddress}.tokenAccountAddress`)
    //   const infoAccount = get(this.farm.stakeAccounts, `${farmInfo.poolId}.stakeAccountAddress`)

    //   const key = getUnixTs().toString()
    //   this.$notify.info({
    //     key,
    //     message: 'Making transaction...',
    //     description: '',
    //     duration: 0
    //   })

    //   withdraw(conn, wallet, farmInfo, lpAccount, rewardAccount, infoAccount, amount)
    //     .then((txid) => {
    //       this.$notify.info({
    //         key,
    //         message: 'Transaction has been sent',
    //         description: (h: any) =>
    //           h('div', [
    //             'Confirmation is in progress.  Check your transaction on ',
    //             h('a', { attrs: { href: `${this.url.explorer}/tx/${txid}`, target: '_blank' } }, 'here')
    //           ])
    //       })

    //       const description = `Unstake ${amount} ${farmInfo.lp.name}`
    //       this.$accessor.transaction.sub({ txid, description })
    //     })
    //     .catch((error) => {
    //       this.$notify.error({
    //         key,
    //         message: 'Stake failed',
    //         description: error.message
    //       })
    //     })
    //     .finally(() => {
    //       this.unstaking = false
    //     })
    // },

    updateLiquids() {
      const liquids = [] as any

      for (const [mintAddress, tokenAccount] of Object.entries(this.wallet.tokenAccounts)) {
        const poolInfo = get(this.liquidity.infos, mintAddress)

        if (poolInfo) {
          const userLpBalance = cloneDeep((tokenAccount as any).balance)

          if (!userLpBalance.isNullOrZero()) {
            liquids.push({
              poolInfo,
              userLpBalance
            })
          }
        }
      }

      this.liquids = liquids
    },

    remove(poolInfo: any, lpBalance: any) {
      this.removing = true

      const conn = this.$web3
      const wallet = (this as any).$wallet

      const lpAccount = get(this.wallet.tokenAccounts, `${poolInfo.lp.mintAddress}.tokenAccountAddress`)
      const fromCoinAccount = get(this.wallet.tokenAccounts, `${poolInfo.coin.mintAddress}.tokenAccountAddress`)
      const toCoinAccount = get(this.wallet.tokenAccounts, `${poolInfo.pc.mintAddress}.tokenAccountAddress`)

      const key = getUnixTs().toString()
      this.$notify.info({
        key,
        message: 'Making transaction...',
        description: '',
        duration: 0
      })

      removeLiquidity(conn, wallet, poolInfo, lpAccount, fromCoinAccount, toCoinAccount, lpBalance.fixed())
        .then((txid) => {
          this.$notify.info({
            key,
            message: 'Transaction has been sent',
            description: (h: any) =>
              h('div', [
                'Confirmation is in progress.  Check your transaction on ',
                h('a', { attrs: { href: `${this.url.explorer}/tx/${txid}`, target: '_blank' } }, 'here')
              ])
          })

          const description = `Remove liquidity for ${lpBalance.format()} ${poolInfo.lp.name}`
          this.$accessor.transaction.sub({ txid, description })
        })
        .catch((error) => {
          this.$notify.error({
            key,
            message: 'Remove liquidity failed',
            description: error.message
          })
        })
        .finally(() => {
          this.removing = false
        })
    }
  }
})
</script>

<style lang="less" scoped>
.container {
  max-width: 450px;

  h6 {
    margin-bottom: 0;
  }

  .action {
    display: grid;
    grid-gap: 4px;
  }
}

.ata-table {
  width: 100%;

  .balance {
    text-align: center;
  }

  .token {
    justify-content: flex-end;
  }

  .actions {
    margin-left: 4px;

    i {
      cursor: pointer;

      &:hover {
        color: @primary-color;
      }
    }

    a {
      color: @text-color;
    }
  }
}

.migrate-token-button {
  margin-top: 10px;
  width: 100%;
}
</style>

<style lang="less">
.ant-steps-item-content {
  width: auto !important;
}

.ant-steps-item-title {
  color: #f1f1f2 !important;
}

.ant-steps-item-description {
  color: #85858d !important;
}
</style>
