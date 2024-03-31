<template>
  <div
    style="width: 100%;height: 100%;background: #f5f5d5;padding: 50px;display: flex;flex-direction: column;align-items: center;color: #000;box-sizing: border-box;">
    <div>
      <h2>
        Cross chain bridge
      </h2>
      <div>
        <button v-show="!metamaskConnected" @click="connectWallet">
          Connect metamask
        </button>
        <span v-show="metamaskConnected">
          {{ account }}
        </span>
      </div>
    </div>
    <div style="width: 100%;display: flex;justify-content: center;align-items: center;">
      <div
        style="border: 1px solid #ccc;margin-top:40px;display: flex;flex-direction: column;position: relative;width: 300px;height: 300px;justify-content: center;margin-right: 30px;overflow: hidden;">
        <p>From optimism to base</p>
        <span>
          op avalible balance: {{ balance }}
        </span>
        <input style="margin:30px 0;" type="number" v-model="amount">
        <span>
          base avalible balance: {{ balance1 }}
        </span>
        <button style="margin-top:30px;" @click="onConfirmClick()">Confirm</button>
      </div>
      <div
        style="border: 1px solid #ccc;margin-top:40px;display: flex;flex-direction: column;position: relative;width: 300px;height: 300px;justify-content: center;overflow: hidden;">

        <p>From base to optimism</p>
        <span>
          op avalible balance: {{ balance }}
        </span>
        <input style="margin:30px 0;" type="number" v-model="amount1">
        <span>
          base avalible balance: {{ balance1 }}
        </span>
        <button style="margin-top:30px;" @click="onConfirmClick1()">Confirm</button>
      </div>

    </div>
    <h3>wait about 35 seconds</h3>
  </div>
</template>

<script>
import Web3 from 'web3';
import BigNumber from "bignumber.js";
import { cross_chain_bridge_abi, op_cross_chain_bridge, base_cross_chain_bridge } from './constant.js';

export default {
  name: 'App',
  data() {
    return {
      account: '',
      metamaskConnected: false,
      amount: 0,
      amount1: 0,
      balance: '',
      balance1: '',
      symbol: '',
    }
  },
  async mounted() {
    if (window.ethereum.isConnected()) {
      const accounts = await ethereum.request({ method: 'eth_accounts' })
      this.metamaskConnected = true;
      if (accounts && accounts.length > 0) {
        this.account = accounts[0];
        this.getBalance();
      }
    }
  },
  methods: {
    async getBalance() {
      const contract = new (new Web3('https://sepolia.optimism.io')).eth.Contract(cross_chain_bridge_abi, op_cross_chain_bridge);
      const balance = await contract.methods.balanceOf(this.account).call();
      const contract1 = new (new Web3('https://sepolia.base.org')).eth.Contract(cross_chain_bridge_abi, base_cross_chain_bridge);
      const balance1 = await contract1.methods.balanceOf(this.account).call();
      this.symbol = await contract.methods.symbol().call();
      this.balance = new BigNumber(Number(balance)).dividedBy('1e18');
      this.balance1 = new BigNumber(Number(balance1)).dividedBy('1e18');
    },
    async onConfirmClick() {
      try {
        const chainId = await ethereum.request({ method: 'eth_chainId' });
        if (chainId !== '0xaa37dc') {
          await window.ethereum.request({
            method: 'wallet_switchEthereumChain',
            params: [{ chainId: '0xaa37dc' }],
          });
        }
        const web3 = new Web3(window.ethereum);
        await window.ethereum.enable();
        await new web3.eth.Contract(
          cross_chain_bridge_abi,
          op_cross_chain_bridge
        ).methods.sendCrossChainTransaction(
          base_cross_chain_bridge,
          web3.utils.padRight(web3.utils.asciiToHex('channel-10'), 64),
          this.onConfirmClick2Erc20Decimal(this.amount)
        ).send({ from: this.account });
        setTimeout(() => {
          this.getBalance()
        }, 35000)
      } catch (error) {
        console.error(error);
      }
    },
    async onConfirmClick1() {
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
          cross_chain_bridge_abi,
          base_cross_chain_bridge
        ).methods.sendCrossChainTransaction(
          op_cross_chain_bridge,
          web3.utils.padRight(web3.utils.asciiToHex('channel-11'), 64),
          this.onConfirmClick2Erc20Decimal(this.amount1)
        ).send({ from: this.account });
        setTimeout(() => {
          this.getBalance()
        }, 35000)
      } catch (error) {
        console.error(error);
      }
    },

    onConfirmClick2Erc20Decimal(number) {
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

<style>
html,
body {
  width: 100%;
  height: 100%;
}
</style>