<template>
  <div id="app">
    <div class="content-top">
      <div class="content-top-title">
        Optimism and Base bridge(Sepolia testnet)
      </div>
      <div class="content-top-account-container">
        <span class="wallet-connect-button" v-show="!metamaskConnected" @click="connectWallet">
          Connect metamask
        </span>
        <span class="content-top-account-address" v-show="metamaskConnected">
          {{ account }}
        </span>
      </div>
    </div>
    <div class="content-bottom">
      <div class="content-bottom-body">
        <div class="loading" v-show="isLoading">
          Waiting...
        </div>
        <div class="content-bottom-body-top">
          <p class="content-bottom-body-top-title">Optimism Base Bridge</p>
          <div class="content-bottom-body-top-item">
            <img v-show="isOp" src="https://optimism-sepolia.blockscout.com/favicon/favicon.ico"
              class="content-bottom-body-top-item-logo">
            <img v-show="!isOp" src="https://base-sepolia.blockscout.com/favicon/favicon.ico"
              class="content-bottom-body-top-item-logo">
            <input type="text" v-model="amount" ref="input">
          </div>
          <span class="content-bottom-body-top-item-balance">
            Avalible balance: {{ topBalance }}
          </span>
          <div class="content-bottom-body-top-arrow">
            <svg @click="toggleChain" viewBox="0 0 24 24" width="24" height="24" fill="none"
              xmlns="http://www.w3.org/2000/svg">
              <path fill-rule="evenodd" clip-rule="evenodd"
                d="M19.3582 15.2117L13.4999 21.2307L13.4999 3.00011L14.4999 3.00011L14.4999 18.7695L18.6416 14.5143L19.3582 15.2117Z"
                fill="currentColor"></path>
              <path fill-rule="evenodd" clip-rule="evenodd"
                d="M4.6416 8.7885L10.4999 2.76953L10.4999 21.0001L9.49991 21.0001L9.4999 5.23069L5.35821 9.48597L4.6416 8.7885Z"
                fill="currentColor"></path>
            </svg>
          </div>
          <div class="content-bottom-body-top-item">
            <img v-show="isOp" src="https://base-sepolia.blockscout.com/favicon/favicon.ico"
              class="content-bottom-body-top-item-logo">
            <img v-show="!isOp" src="https://optimism-sepolia.blockscout.com/favicon/favicon.ico"
              class="content-bottom-body-top-item-logo">
            <input placeholder="You will receive amount" disabled type="text"
              style="cursor: not-allowed;background:rgba(0, 0, 0, 0.04)" v-model="amount">

          </div>
          <span class="content-bottom-body-top-item-balance">
            Avalible balance: {{ bottomBalance }}
          </span>
        </div>
        <span class="content-bottom-body-top-bottom" @click="transfer()">Confirm</span>
      </div>

    </div>
  </div>
</template>

<script>
import Web3 from 'web3';
import BigNumber from "bignumber.js";
import { bridgeAbi, opBridge, baseBridge } from './constant.js';

export default {
  name: 'App',
  data() {
    return {
      account: '',
      metamaskConnected: false,
      amount: '',
      isLoading: false,
      opBalance: '',
      baseBalance: '',
      symbol: '',
      isOp: true,
    }
  },
  async mounted() {
    this.$refs.input.focus();
    if (window.ethereum.isConnected()) {
      const accounts = await ethereum.request({ method: 'eth_accounts' })
      this.metamaskConnected = true;
      if (accounts && accounts.length > 0) {
        this.account = accounts[0];
        this.getBalance();
      }
    }
  },
  computed: {
    topBalance() {
      return `${this.isOp ? this.opBalance : this.baseBalance} ${this.symbol}`
    },

    bottomBalance() {
      return `${this.isOp ? this.baseBalance : this.opBalance} ${this.symbol}`
    }
  },
  methods: {
    toggleChain() {
      this.isOp = !this.isOp;
    },
    async getBalance() {
      const opContract = new (new Web3('https://sepolia.optimism.io')).eth.Contract(bridgeAbi, opBridge);
      const opBalance = await opContract.methods.balanceOf(this.account).call();
      const baseContract = new (new Web3('https://sepolia.base.org')).eth.Contract(bridgeAbi, baseBridge);
      const baseBalance = await baseContract.methods.balanceOf(this.account).call();
      this.symbol = await opContract.methods.symbol().call();
      this.opBalance = new BigNumber(Number(opBalance)).dividedBy('1e18');
      this.baseBalance = new BigNumber(Number(baseBalance)).dividedBy('1e18');
    },
    async transfer() {
      try {
        const chainId = await ethereum.request({ method: 'eth_chainId' });
        if (this.isOp && chainId !== '0xaa37dc') {
          await window.ethereum.request({
            method: 'wallet_switchEthereumChain',
            params: [{ chainId: '0xaa37dc' }],
          });
        } else if (!this.isOp && chainId !== '0x14a34') {
          await window.ethereum.request({
            method: 'wallet_switchEthereumChain',
            params: [{ chainId: '0x14a34' }],
          });
        }
        const web3 = new Web3(window.ethereum);
        await window.ethereum.enable();
        this.isLoading = true
        await new web3.eth.Contract(
          bridgeAbi,
          this.isOp ? opBridge : baseBridge
        ).methods.sendTx(
          this.isOp ? baseBridge : opBridge,
          web3.utils.padRight(web3.utils.asciiToHex(this.isOp ? 'channel-10' : 'channel-11'), 64),
          this.transfer2Erc20Decimal(this.amount)
        ).send({ from: this.account });
        setTimeout(() => {
          this.isLoading = false;
          this.getBalance()
        }, 38000)
      } catch (error) {
        this.isLoading = false;
        console.error(error);
      }
    },

    transfer2Erc20Decimal(number) {
      return new BigNumber(number).multipliedBy(new BigNumber(10).pow(18)).toNumber();
    },
    async connectWallet() {
      try {
        await window.ethereum.enable();
        const accountList = await new Web3(window.ethereum).eth.getAccounts();
        this.account = accountList[0]
        this.metamaskConnected = true;
        this.getBalance();
      } catch (error) {
        console.error(error);
        alert(error)
      }

    },

  }

}
</script>

<style lang="scss">
html,
body {
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0;
}



input {
  width: 100%;
  height: 22px;
  border: 1px solid #d9d9d9;
  background: #fff;
  border-radius: 8px;
  height: 36px;
  text-indent: 10px;
  outline: none;
}

#app {
  width: 100%;
  height: 100%;
  background: #fff;
  padding: 20px;
  display: flex;
  flex-direction: column;
  align-items: center;
  color: #000;
  box-sizing: border-box;

  .content-top {
    height: 100px;
    flex: 0 0 100px;
    display: flex;
    flex-direction: column;
    width: 100%;

    .content-top-title {
      font-size: 44px;
      font-weight: 900;
      text-align: center;
      height: 44px;
      line-height: 44px;
      margin-bottom: 20px;
    }

    .content-top-account-container {
      display: flex;
      justify-content: flex-end;

      .wallet-connect-button {
        background: #fff;
        color: #3598db;
        border: 1px solid #3598db;
        border-radius: 8px;
        height: 36px;
        line-height: 36px;
        text-align: center;
        cursor: pointer;
        padding: 0 15px;
      }

      .content-top-account-address {
        font-size: 20px;
        color: #ccc;
      }
    }
  }

  .content-bottom {
    width: 100%;
    display: flex;
    flex: 1;
    justify-content: center;
    align-items: center;

    .content-bottom-body {
      border: 1px solid #ccc;
      border-radius: 20px;
      width: 400px;
      padding: 15px;
      box-sizing: border-box;
      display: flex;
      flex-direction: column;
      position: relative;
      overflow: hidden;

      .loading {
        width: 100%;
        height: 100%;
        z-index: 100;
        position: absolute;
        background: rgba(0, 0, 0, 0.7);
        top: 0;
        left: 0;
        display: flex;
        justify-content: center;
        align-items: center;
        color: #fff;
        font-size: 20px;
      }

      .content-bottom-body-top {
        width: 100%;
        display: flex;
        flex-direction: column;

        .content-bottom-body-top-title {
          width: 100%;
          text-align: center;
          font-size: 24px;
          font-weight: 600;
          margin-bottom: 30px;
        }

        .content-bottom-body-top-item {
          width: 100%;
          display: flex;
          align-items: center;

          &:last-child {
            margin-bottom: 50px;
          }

          .content-bottom-body-top-item-logo {
            width: 30px;
            height: 30px;
            flex: 0 0 30px;
            margin-right: 10px;
          }

          input {
            flex: 1;
          }
        }

        .content-bottom-body-top-item-balance {
          width: 100%;
          display: flex;
          justify-content: flex-end;
          margin-top: 10px;
          font-size: 12px;
        }

        .content-bottom-body-top-arrow {
          display: flex;
          justify-content: center;
          margin: 20px 0;

          svg {
            cursor: pointer;
          }

        }
      }

      .content-bottom-body-top-bottom {
        width: 100%;
        height: 40px;
        line-height: 40px;
        text-align: center;
        border-radius: 10px;
        font-weight: 600;
        font-size: 18px;
        background: #FF0420;
        color: #fff;
        cursor: pointer;
        margin-top: 40px;
      }
    }
  }
}
</style>