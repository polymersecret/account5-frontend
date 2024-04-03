<template>
  <div class="app-container">
    <div class="app-head">
      <div>
        <button v-show="account === ''" @click="connectWallet">
          Connect metamask
        </button>
        <span v-show="account !== ''">
          {{ account }}
        </span>
      </div>
    </div>
    <div class="app-body">
      <div class="app-body-item">
        <p>optimism to base</p>
        <input class="input-field" v-model="amountOp">
        <button style="margin-top:30px;width: 150px;height:30px;" @click="handleOpBtnClick()">
          Send to base
        </button>
      </div>
      <span style="margin:0 20px;">----------></span>
      <div class="app-body-item">
        <p>base to optimism</p>
        <input class="input-field" v-model="amountBase">
        <button style="margin-top:30px;width: 150px;height:30px;" @click="handleBaseBtnClick()">
          Send to optimism
        </button>
      </div>

    </div>
    <h3>
      <span>
        op avalible balanceOp: {{ balanceOp }}
      </span>
      <span>
        base avalible balanceOp: {{ balanceBase }}
      </span>
    </h3>
  </div>
</template>

<script>
import Web3 from 'web3';
import BigNumber from "bignumber.js";
import { abi, opBridge, baseBridge } from './constant.js';

export default {
  name: 'App',
  data() {
    return {
      account: '',
      amountOp: '',
      amountBase: '',
      balanceOp: '',
      balanceBase: '',
      symbol: '',
    }
  },
  async mounted() {
    if (window.ethereum.isConnected()) {
      const accounts = await ethereum.request({ method: 'eth_accounts' })
      if (accounts && accounts.length > 0) {
        this.account = accounts[0];
        this.queryBalance();
      }
    }
  },
  methods: {
    mutiDecimal(number) {
      return new BigNumber(number).multipliedBy(new BigNumber(10).pow(18)).toNumber();
    },
    async queryBalance() {
      const contractOp = new (new Web3('https://sepolia.optimism.io')).eth.Contract(abi, opBridge);
      const contractBase = new (new Web3('https://sepolia.base.org')).eth.Contract(abi, baseBridge);
      const balanceOp = await contractOp.methods.balanceOf(this.account).call();
      const balanceBase = await contractBase.methods.balanceOf(this.account).call();
      this.symbol = await contractOp.methods.symbol().call();
      this.balanceOp = new BigNumber(Number(balanceOp)).dividedBy('1e18');
      this.balanceBase = new BigNumber(Number(balanceBase)).dividedBy('1e18');
    },
    async handleOpBtnClick() {
      try {
        const web3 = new Web3(window.ethereum);
        await window.ethereum.enable();
        await new web3.eth.Contract(
          abi,
          opBridge
        ).methods.ibcBridgeSend(
          baseBridge,
          web3.utils.padRight(web3.utils.asciiToHex('channel-10'), 64),
          this.mutiDecimal(this.amountOp)
        ).send({ from: this.account });
        setTimeout(() => {
          this.queryBalance()
        }, 35000)
      } catch (error) {
        console.error(error);
      }
    },
    async handleBaseBtnClick() {
      try {
        const chainId = await ethereum.request({ method: 'eth_chainId' });
        if (chainId !== '0x14a34') {
          await window.ethereum.request({
            method: 'wallet_switchEthereumChain',
            params: [{ chainId: '0x14a34' }],
          });
        }
        const web3 = new Web3(window.ethereum);
        await window.ethereum.enable();
        await new web3.eth.Contract(
          abi,
          baseBridge
        ).methods.ibcBridgeSend(
          opBridge,
          web3.utils.padRight(web3.utils.asciiToHex('channel-11'), 64),
          this.mutiDecimal(this.amountBase)
        ).send({ from: this.account });
        setTimeout(() => {
          this.queryBalance()
        }, 35000)
      } catch (error) {
        console.error(error);
      }
    },


    async connectWallet() {
      try {
        await window.ethereum.enable();
        const accountList = await new Web3(window.ethereum).eth.getAccounts();
        this.account = accountList[0]
        this.queryBalance();
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

.app-container {
  width: 100%;
  height: 100%;
  background: #181818;
  padding: 50px;
  display: flex;
  flex-direction: column;
  align-items: center;
  color: #fff;
  box-sizing: border-box;

  .app-head {
    width: 100%;
    display: flex;
    justify-content: flex-end;
  }

  .app-body {
    width: 100%;
    display: flex;
    justify-content: center;
    align-items: center;

    .app-body-item {
      border: 1px solid greenyellow;
      margin-top: 40px;
      display: flex;
      flex-direction: column;
      position: relative;
      width: 300px;
      height: 300px;
      justify-content: center;
      overflow: hidden;
      padding: 10px 20px;
      box-sizing: border-box;
    }
  }

  h3 {
    span {
      display: inline-block;
      margin-right: 30px;
    }
  }
}

.input-field {
  width: 100%;
  padding: 5px 10px;
  border: 1px solid #d9d9d9;
  border-radius: 2px;
  box-sizing: border-box;
  transition: all 0.3s;
}

.input-field:focus {
  border-color: #40a9ff;
  outline: none;
  box-shadow: 0 0 0 2px rgba(24, 144, 255, 0.2);
}
</style>