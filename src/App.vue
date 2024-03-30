<template>
  <div id="app">
    <h2 style="font-weight:900;color:#fff">Polymer Bridge</h2>
    <div class="wallet-connect-container">
      <span class="btn" v-show="!metamaskConnected" @click="connectWallet">
        Connect metamask
      </span>
      <span v-show="metamaskConnected" style="color:#fff;font-size:18px;">
        {{ address }}
      </span>
    </div>
    <div
      style="margin-bottom: 30px;display:flex;width:100%;flex-direction: column;align-items: flex-end;padding-right: 40px;font-size:18px;color:#ffffff;">
      <span style="width:260px">
        Op balance: {{ opBalance }} {{ symbol }}
      </span>
      <span style="width:260px">
        Base balance: {{ baseBalance }} {{ symbol }}
      </span>
    </div>
    <div class="content-wrap">
      <span style="margin:0 0 10px 0;">From op to base</span>
      <div style="display:flex;width:100%;">
        <input v-show="metamaskIsInstalled" type="text" v-model="opAmount" class="bridge-input">
        <span style="margin-left:10px;" class="btn" @click="transfer('op')">Send</span>
      </div>
      <span style="margin:20px 0 10px 0;">From base to op</span>
      <div style="display:flex;width:100%;">
        <input v-show="metamaskIsInstalled" type="text" v-model="baseAmount" class="bridge-input">
        <span style="margin-left:10px;" class="btn" @click="transfer('base')">Send</span>
      </div>
      <div class="loading" v-show="isLoading">
        <span class="spin"></span>
      </div>
    </div>
  </div>
</template>

<script>
import Web3 from 'web3';
import BigNumber from "bignumber.js";
import { CONTRACT_ABI } from './abi.js';
const OP_PORT_ADDRESS = '0xB49a654EcED1CF6FabE4C347faD2089490889fCE', BASE_PORT_ADDRESS = '0xB49a654EcED1CF6FabE4C347faD2089490889fCE';

export default {
  name: 'App',
  data() {
    return {
      address: '',
      //metamaskIsInstalled: false,
      metamaskConnected: false,
      opAmount: '',
      baseAmount: '',
      isLoading: false,
      opBalance: '',
      baseBalance: '',
      symbol: '',
    }
  },
  async mounted() {
    if (window.ethereum.isConnected()) {
          const accounts = await ethereum.request({ method: 'eth_accounts' })
          if (accounts && accounts.length > 0) {
            this.address = accounts[0];
            this.metamaskConnected = true;
            this.getBalance();
          }
        }
  },
  computed:{
    metamaskIsInstalled(){
      return typeof window.ethereum !== 'undefined';
    }
  },
  methods: {
    async getBalance() {
      try {
        const opContract = new (new Web3('https://sepolia.optimism.io')).eth.Contract(CONTRACT_ABI, OP_PORT_ADDRESS);
        const opBalance = await opContract.methods.balanceOf(
          this.address,
        ).call();
        const baseContract = new (new Web3('https://sepolia.base.org')).eth.Contract(CONTRACT_ABI, BASE_PORT_ADDRESS);
        const baseBalance = await baseContract.methods.balanceOf(
          this.address,
        ).call();
        this.symbol = await opContract.methods.symbol().call();
        this.opBalance = new BigNumber(Number(opBalance)).dividedBy('1e18');
        this.baseBalance = new BigNumber(Number(baseBalance)).dividedBy('1e18');
      } catch (error) {
        console.error(error);
      }
    },
    async transfer(chain) {
      console.log(this.opAmount)
      console.log(this.baseAmount)
      try {
        const chainId = await ethereum.request({ method: 'eth_chainId' });
        if (chain === 'op' && chainId !== '0xaa37dc') {
          await window.ethereum.request({
            method: 'wallet_switchEthereumChain',
            params: [{ chainId: '0xaa37dc' }],
          });
        } else if (chain === 'base' && chainId !== '0x14a34') {
          await window.ethereum.request({
            method: 'wallet_switchEthereumChain',
            params: [{ chainId: '0x14a34' }],
          });
        }
        const web3 = new Web3(window.ethereum);
        await window.ethereum.enable();
        this.isLoading = true
        await new web3.eth.Contract(
          CONTRACT_ABI,
          chain === 'op' ? OP_PORT_ADDRESS : BASE_PORT_ADDRESS
        ).methods.bridgePMT(
          chain === 'op' ? BASE_PORT_ADDRESS : OP_PORT_ADDRESS,
          web3.utils.padRight(web3.utils.asciiToHex(chain === 'op' ? 'channel-10' : 'channel-11'), 64),
          this.transfer2Erc20Decimal(
            chain === 'op' ? this.opAmount : this.baseAmount
          )
        ).send({ from: this.address });
        setTimeout(() => {
          this.isLoading = false;
          this.getBalance()
        }, 40000)
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
        this.address = accountList[0]
        this.metamaskConnected = true;
        this.getBalance();
      } catch (error) {
        console.error(error);
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

#app {
  width: 100%;
  height: 100%;
  background: linear-gradient(140deg, #6a3093, #005792);
  padding-top: 20px;
  display: flex;
  flex-direction: column;
  align-items: center;
  color: #000;

  .btn {
    background-color: #3598db;
    color: #fff;
    border-radius: 8px;
    height: 36px;
    line-height: 36px;
    text-align: center;
    cursor: pointer;
    padding: 0 15px;

    &:hover {
      background-color: #66b1ff;
    }
  }

  .wallet-connect-container {
    margin-bottom: 10px;
    width: 100%;
    display: flex;
    justify-content: flex-end;
    padding-right: 40px;
    height: 40px;
  }

  .content-wrap {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    height: 280px;
    width: 400px;
    color: #101112;
    background: #f5f5d5;
    border-radius: 20px;
    padding: 40px;
    position: relative;
    overflow: hidden;
    box-sizing: border-box;
    align-items: flex-start;

    input {
      width: 100%;
      height: 22px;
      border: none;
      background: #f2f4f5;
      border-radius: 8px;
      height: 36px;
      text-indent: 10px;
      outline: none;
    }

    .loading {
      width: 100%;
      height: 100%;
      z-index: 100;
      position: absolute;
      background: rgba(0, 0, 0, 0.86);
      top: 0;
      left: 0;
      display: flex;
      justify-content: center;
      align-items: center;

      .spin {
        border: 16px solid #f3f3f3;
        /* Light grey background */
        border-top: 16px solid #3498db;
        /* Blue color */
        border-radius: 50%;
        width: 30px;
        height: 30px;
        animation: spin 2s linear infinite;
      }

      @keyframes spin {
        0% {
          transform: rotate(0deg);
        }

        100% {
          transform: rotate(360deg);
        }
      }

    }
  }

}
</style>
