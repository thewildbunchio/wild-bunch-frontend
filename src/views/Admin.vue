<template>
<div class="container">
  <h1 class="text-center">Admin</h1>
  
  <div class="row mt-3 text-center">
    <div class="col-md-8 offset-md-2">

      <!-- Minter: togglePaused -->
      <div v-if="isUserMinterAdmin">
        <h3 v-if="getMinterPaused">Unpause the minting contract</h3>
        <h3 v-if="!getMinterPaused">Pause the minting contract</h3>

        <button 
          v-if="isActivated" 
          class="btn btn-primary btn-lg mt-3" 
          @click="togglePaused" 
          :disabled="waitingPaused"
        >
          <span v-if="waitingPaused" class="spinner-border spinner-border-sm mx-1" role="status" aria-hidden="true"></span>
          <span v-if="getMinterPaused">Unpause</span>
          <span v-if="!getMinterPaused">Pause</span>
        </button>

        <hr />
      </div>
      <!-- END Minter: togglePaused -->

      <!-- Minter: changeReferralFee -->
      <div v-if="isUserTldAdmin">
        <h3>Change referral fee</h3>

        <p>Important: Referral fees are defined in basis points (bps). Example: 1000 bps = 10%</p>

        <div class="row mt-5">
          <div class="col-md-6 offset-md-3">
            <input 
              v-model="newReferralFee"
              class="form-control text-center border-2 border-dark"
              placeholder="Enter a new referral fee"
            >
          </div>
        </div>

        <button 
          v-if="isActivated" 
          class="btn btn-primary btn-lg mt-3" 
          @click="changeReferralFee" 
          :disabled="waitingRf"
        >
          <span v-if="waitingRf" class="spinner-border spinner-border-sm mx-1" role="status" aria-hidden="true"></span>
          <span>Change referral fee</span>
        </button>

        <hr />
      </div>
      <!-- END Minter: changeReferralFee -->

      <!-- Minter: changePrice -->
      <div v-if="isUserTldAdmin">
        <h3>Change price</h3>

        <p>Current price: {{getMinterTldPrice}} ETH</p>

        <div class="row mt-5">
          <div class="col-md-6 offset-md-3">
            <input 
              v-model="newPrice"
              class="form-control text-center border-2 border-dark"
              placeholder="Enter the new domain price"
            >
          </div>
        </div>

        <button 
          v-if="isActivated" 
          class="btn btn-primary btn-lg mt-3" 
          @click="changePrice" 
          :disabled="waitingPrice"
        >
          <span v-if="waitingPrice" class="spinner-border spinner-border-sm mx-1" role="status" aria-hidden="true"></span>
          <span>Change price</span>
        </button>

        <hr />
      </div>
      <!-- END Minter: changePrice -->

      <!-- Minter: transferOwnership -->
      <div v-if="isUserMinterAdmin">
        <h3>Minter contract: transfer ownership</h3>

        <p>Transfer ownership of the minter contract to another address.</p>

        <div class="row mt-5">
          <div class="col-md-6 offset-md-3">
            <input 
              v-model="newMinterOwner"
              class="form-control text-center border-2 border-dark"
              placeholder="Enter a new owner address"
            >
          </div>
        </div>

        <button 
          v-if="isActivated" 
          class="btn btn-primary btn-lg mt-3" 
          @click="transferMinterOwnership" 
          :disabled="waitingTmo"
        >
          <span v-if="waitingTmo" class="spinner-border spinner-border-sm mx-1" role="status" aria-hidden="true"></span>
          <span>Transfer minter ownership</span>
        </button>

        <hr />
      </div>
      <!-- END Minter: transferOwnership -->

      <!-- TLD: transferOwnership -->
      <div v-if="isUserTldAdmin">
        <h3>TLD contract (.wildbunch): transfer ownership</h3>

        <p>Transfer ownership of the TLD contract to another address.</p>

        <div class="row mt-5">
          <div class="col-md-6 offset-md-3">
            <input 
              v-model="newTldOwner"
              class="form-control text-center border-2 border-dark"
              placeholder="Enter a new owner address"
            >
          </div>
        </div>

        <button 
          v-if="isActivated" 
          class="btn btn-primary btn-lg mt-3" 
          @click="transferTldOwnership" 
          :disabled="waitingTdo"
        >
          <span v-if="waitingTdo" class="spinner-border spinner-border-sm mx-1" role="status" aria-hidden="true"></span>
          <span>Transfer TLD ownership</span>
        </button>

        <hr />
      </div>
      <!-- END TLD: transferOwnership -->

    </div>
  </div>
</div>
</template>

<script lang="ts">
import { ethers } from 'ethers';
import { useEthers } from 'vue-dapp';
import { mapActions, mapGetters } from 'vuex';
import { useToast, TYPE } from "vue-toastification";
import WaitingToast from "../components/toasts/WaitingToast.vue";
import MinterAbi from "../abi/Minter.json";

export default {
  name: "Admin",

  data() {
    return {
      newMinterOwner: null,
      newTldOwner: null,
      newPrice: null,
      newReferralFee: null,
      waitingPaused: false, // waiting for TX to complete
      waitingPrice: false, // waiting for TX to complete
      waitingRf: false, // waiting for TX to complete
      waitingTmo: false, // waiting for TX to complete
      waitingTdo: false, // waiting for TX to complete
    }
  },

  computed: {
    ...mapGetters("punk", ["getTldAbi"]),
    ...mapGetters("network", ["getBlockExplorerBaseUrl"]),
    ...mapGetters("user", ["isUserMinterAdmin", "isUserTldAdmin", "isUserRoyaltyFeeUpdater"]),
    ...mapGetters("tld", ["getMinterAddress", "getTldAddress", "getMinterPaused", "getMinterTldPrice"]),
  },

  methods: {
    ...mapActions("tld", ["fetchMinterContractData"]),

    async changePrice() {
      this.waitingPrice = true;
      this.newPrice;

      // convert price to wei
      const newPriceWei = ethers.utils.parseUnits(this.newPrice, 18);

      // TLD contract (with signer)
      const tldIntfc = new ethers.utils.Interface(this.getTldAbi);
      const tldContractSigner = new ethers.Contract(this.getTldAddress, tldIntfc, this.signer);

      try {
        const tx = await tldContractSigner.changePrice(newPriceWei);

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
          this.toast("You have changed price!", {
            type: TYPE.SUCCESS,
            onClick: () => window.open(this.getBlockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
          });
          this.fetchMinterContractData();
          this.waitingPrice = false;
        } else {
          this.toast.dismiss(toastWait);
          this.toast("Transaction has failed.", {
            type: TYPE.ERROR,
            onClick: () => window.open(this.getBlockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
          });
          console.log(receipt);
          this.waitingPrice = false;
        }

      } catch (e) {
        console.log(e)
        this.waitingPrice = false;
        this.toast(e.message, {type: TYPE.ERROR});
      }

      this.waitingPrice = false;
    },

    async changeReferralFee() {
      this.waitingRf = true;

      const tldIntfc = new ethers.utils.Interface(this.getTldAbi);
      const tldContractSigner = new ethers.Contract(this.getTldAddress, tldIntfc, this.signer);

      try {
        const tx = await tldContractSigner.changeReferralFee(Number(this.newReferralFee));

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
          this.toast("You have changed the referral fee!", {
            type: TYPE.SUCCESS,
            onClick: () => window.open(this.getBlockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
          });
          this.waitingRf = false;
        } else {
          this.toast.dismiss(toastWait);
          this.toast("Transaction has failed.", {
            type: TYPE.ERROR,
            onClick: () => window.open(this.getBlockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
          });
          console.log(receipt);
          this.waitingRf = false;
        }

      } catch (e) {
        console.log(e)
        this.waitingRf = false;
        this.toast(e.message, {type: TYPE.ERROR});
      }

      this.waitingRf = false;
    },

    async togglePaused() {
      this.waitingPaused = true;

      // minter contract (with signer)
      const minterIntfc = new ethers.utils.Interface(MinterAbi);
      const minterContractSigner = new ethers.Contract(this.getMinterAddress, minterIntfc, this.signer);

      try {
        const tx = await minterContractSigner.togglePaused();

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
          this.toast("You have un/paused the minter contract!", {
            type: TYPE.SUCCESS,
            onClick: () => window.open(this.getBlockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
          });
          this.fetchMinterContractData();
          this.waitingPaused = false;
        } else {
          this.toast.dismiss(toastWait);
          this.toast("Transaction has failed.", {
            type: TYPE.ERROR,
            onClick: () => window.open(this.getBlockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
          });
          console.log(receipt);
          this.waitingPaused = false;
        }

      } catch (e) {
        console.log(e)
        this.waitingPaused = false;
        this.toast(e.message, {type: TYPE.ERROR});
      }

      this.waitingPaused = false;
    },

    async transferMinterOwnership() {
      this.waitingTmo = true;

      // minter contract (with signer)
      const minterIntfc = new ethers.utils.Interface(MinterAbi);
      const minterContractSigner = new ethers.Contract(this.getMinterAddress, minterIntfc, this.signer);

      try {
        const tx = await minterContractSigner.transferOwnership(this.newMinterOwner);

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
          this.toast("You have transferred ownership!", {
            type: TYPE.SUCCESS,
            onClick: () => window.open(this.getBlockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
          });
          this.waitingTmo = false;
        } else {
          this.toast.dismiss(toastWait);
          this.toast("Transaction has failed.", {
            type: TYPE.ERROR,
            onClick: () => window.open(this.getBlockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
          });
          console.log(receipt);
          this.waitingTmo = false;
        }

      } catch (e) {
        console.log(e)
        this.waitingTmo = false;
        this.toast(e.message, {type: TYPE.ERROR});
      }

      this.waitingTmo = false;
    },

    async transferTldOwnership() {
      this.waitingTdo = true;

      // TLD contract (with signer)
      const tldIntfc = new ethers.utils.Interface(this.getTldAbi);
      const tldContractSigner = new ethers.Contract(this.getTldAddress, tldIntfc, this.signer);

      try {
        const tx = await tldContractSigner.transferOwnership(this.newTldOwner);

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
          this.toast("You have transferred ownership!", {
            type: TYPE.SUCCESS,
            onClick: () => window.open(this.getBlockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
          });
          this.waitingTdo = false;
        } else {
          this.toast.dismiss(toastWait);
          this.toast("Transaction has failed.", {
            type: TYPE.ERROR,
            onClick: () => window.open(this.getBlockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
          });
          console.log(receipt);
          this.waitingTdo = false;
        }

      } catch (e) {
        console.log(e)
        this.waitingTdo = false;
        this.toast(e.message, {type: TYPE.ERROR});
      }

      this.waitingTdo = false;
    },

  },

  setup() {
    const { address, isActivated, signer } = useEthers();
    const toast = useToast();

    return { address, isActivated, signer, toast }
  }
}
</script>

<style scoped>
p {
  font-size: 1.1em;
}

h3 {
  margin-top: 35px;
}
</style>