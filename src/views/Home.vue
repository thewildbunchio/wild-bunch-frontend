<template>
  <div class="container text-center">
    <img class="img-fluid main-img rounded" src="../assets/wildbunch-logo.jpeg" />

    <div class="text-align-header mt-3">
      <h1>Mint your {{getTldName}} domain!</h1>
    </div>

    <div class="d-flex justify-content-center domain-input-container mb-3 mt-5">
      <div class="input-group domain-input input-group-lg input-sizing">
        <input
          v-model="chosenDomainName" 
          placeholder="enter domain name"
          type="text" 
          class="form-control text-end border-2 border-end-0 border-dark"
          aria-label="Text input with dropdown button"
        >

        <span class="input-group-text tld-addon border-2 border-dark">
          <span v-if="loading" class="spinner-border spinner-border-sm mx-1" role="status" aria-hidden="true"></span>
          <span style="color:#1B0D23;">{{getTldName}}</span>
        </span>
      </div>
    </div>

    <p class="error">
      <small v-if="buyNotValid(chosenDomainName).invalid">
        <em>{{ buyNotValid(chosenDomainName).message }}</em>
      </small>
    </p>

    <div class="text-align-header">
      <p class="mt-5 price-text">
          Domain price: {{getMinterTldPrice}} {{getPaymentTokenName}}
      </p>
    </div>

    <!-- Minter contract paused -->
    <button v-if="isActivated && getMinterPaused" class="btn btn-primary btn-lg mt-3 buy-button" :disabled="true">
      <span v-if="getMinterPaused">Buying paused</span>
    </button>

    <!-- Not eligible -->
    <button 
      v-if="isActivated && isNetworkSupported && !getMinterPaused && !getCanUserBuy" 
      class="btn btn-primary btn-lg mt-3 buy-button" 
      :disabled="waiting || buyNotValid(chosenDomainName).invalid || !hasUserEnoughTokens"
    >
      <span>Not TWB NFT holder</span>
    </button>

    <!-- Too low ETH balance -->
    <button 
      v-if="isActivated && isNetworkSupported && !getMinterPaused && !hasUserEnoughTokens && getCanUserBuy" 
      class="btn btn-primary btn-lg mt-3 buy-button" 
      :disabled="waiting || buyNotValid(chosenDomainName).invalid || !hasUserEnoughTokens"
    >
      <span>Your {{getPaymentTokenName}} balance is too low</span>
    </button>

    <!-- Buy domain -->
    <button 
      v-if="isActivated && isNetworkSupported && getCanUserBuy && !getMinterPaused && hasUserEnoughTokens" 
      class="btn btn-primary btn-lg mt-3 buy-button" 
      @click="buyDomain" 
      :disabled="waiting || buyNotValid(chosenDomainName).invalid || !hasUserEnoughTokens"
    >
      <span v-if="waiting" class="spinner-border spinner-border-sm mx-1" role="status" aria-hidden="true"></span>
      <span>Buy domain</span>
    </button>

    <!-- Connect Wallet -->
    <button v-if="!isActivated" class="btn btn-primary btn-lg mt-3 btn-Disconnected" @click="open">Connect wallet</button>

    <div v-if="isActivated && !isNetworkSupported" class="mt-4">
      <button class="btn btn-primary btn-lg btn-Disconnected" @click="changeNetwork(this.getTldChainName)">Switch to {{getTldChainName}}</button>
    </div>
    
  </div>

  <Referral v-if="isActivated" />

</template>

<script>
import { ethers } from 'ethers';
import { useBoard, useEthers } from 'vue-dapp';
import { mapActions, mapGetters, mapMutations } from 'vuex';
import { useToast, TYPE } from "vue-toastification";
import WaitingToast from "../components/toasts/WaitingToast.vue";
import Referral from '../components/Referral.vue';
import useDomainHelpers from "../hooks/useDomainHelpers";
import useChainHelpers from "../hooks/useChainHelpers";
import MinterAbi from "../abi/Minter.json";
import erc20Abi from '../abi/Erc20.json';

export default {
  name: "Home",

  data() {
    return {
      chosenDomainName: null,
      chosenAllowance: null,
      loading: false, // loading data
      waiting: false, // waiting for TX to complete
      minterContract: null
    }
  },

  components: {
    Referral
  },

  computed: {
    ...mapGetters("user", ["getPaymentTokenAddress", "getPaymentTokenName", "getPaymentTokenAllowance", "getUserBalance", "getCanUserBuy", "getDiscountEligible"]),
    ...mapGetters("network", ["getBlockExplorerBaseUrl"]),
    ...mapGetters("tld", ["getTldChainId", "getTldChainName", "getMinterAddress", "getTldContract", "getMinterTldPrice", "getMinterPaused", "getMinterDiscountPercentage", "getTldName"]),

    domainLowerCase() {
      return this.chosenDomainName.toLowerCase();
    },

    hasUserEnoughTokens() {
      if (this.address && Number(this.getMinterTldPrice) > 0 && Number(this.getUserBalance) > 0) {
        if (Number(this.getUserBalance) >= Number(this.getMinterTldPrice)) {
          return true;
        }
      }

      return false;
    },

    isNetworkSupported() {
      if (this.isActivated) {
        if (this.chainId === this.getTldChainId) {
          return true;
        }
      }

      return false;
    }
  },

  methods: {
    ...mapActions("user", ["fetchCanUserBuy", "getPaymentTokenDecimals"]),
    ...mapMutations("user", ["addDomainManually", "setPaymentTokenAllowance"]),

    async buyDomain() {
      this.waiting = true;
      const fullDomainName = this.domainLowerCase + this.getTldName;

      // check if domain already minted
      const existingHolder = await this.getTldContract.getDomainHolder(this.domainLowerCase);

      if (existingHolder !== ethers.constants.AddressZero) {
        this.toast("Sorry, but this domain name is already taken...", {type: TYPE.ERROR});
        this.waiting = false;
        return;
      }

      // wrapper contract (with signer)
      const wrapperIntfc = new ethers.utils.Interface(MinterAbi);
      const minterContractSigner = new ethers.Contract(this.getMinterAddress, wrapperIntfc, this.signer);

      try {
        let referral = localStorage.getItem("referral");

        if (!referral || !ethers.utils.isAddress(referral)) {
          referral = ethers.constants.AddressZero;
        }

        const tx = await minterContractSigner.mint(
          this.domainLowerCase,
          this.address,
          referral,
          {
            value: ethers.utils.parseEther(this.getMinterTldPrice)
          }
        );

        const toastWait = this.toast(
          {
            component: WaitingToast,
            props: {
              text: "Please wait for your transaction to confirm. Click on this notification to see transaction in the block explorer."
            }
          },
          {
            type: TYPE.INFO,
            onClick: () => window.open(this.getBlockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
          }
        );

        const receipt = await tx.wait();

        if (receipt.status === 1) {
          this.toast.dismiss(toastWait);
          this.toast("You have successfully bought the domain!", {
            type: TYPE.SUCCESS,
            onClick: () => window.open(this.getBlockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
          });
          this.addDomainManually(fullDomainName);
          this.waiting = false;
        } else {
          this.toast.dismiss(toastWait);
          this.toast("Transaction has failed.", {
            type: TYPE.ERROR,
            onClick: () => window.open(this.getBlockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
          });
          console.log(receipt);
          this.waiting = false;
        }

      } catch (e) {
        console.log(e)
        this.waiting = false;
        this.toast(e.message, {type: TYPE.ERROR});
      }

      this.waiting = false;
    },

    changeNetwork(networkName) {
      const networkData = this.switchNetwork(networkName); 

      window.ethereum.request({ 
        method: networkData.method, 
        params: networkData.params
      });
    },

  },

  setup() {
    const { open } = useBoard();
    const { address, chainId, isActivated, signer } = useEthers();
    const toast = useToast();
    const { buyNotValid } = useDomainHelpers();
    const { switchNetwork } = useChainHelpers();

    return { address, buyNotValid, chainId, isActivated, open, signer, switchNetwork, toast }
  }
}
</script>


<style scoped>
.and {
  font-size: 1.7em;
  vertical-align: bottom;
  padding-left: 0.2em;
  padding-right: 0.1em;
}

.buy-button {
  margin-bottom: 20px;
}

.container {
  padding-top: 80px;
  padding-bottom: 50px;
}

.domain-input {
  width: 50%;
}

.domain-input-container {
  margin-top: 30px;
}

.error {
  color: #DBDFEA;
}

.main-img {
  width: 10em;
}

.tld-addon {
  background-color: white;
}

@media only screen and (max-width: 767px) {
  .domain-input {
    width: 100%;
  }
}
</style>