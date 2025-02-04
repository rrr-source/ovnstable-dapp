<template>
    <v-col class="ma-0 pa-0">
        <v-row dense>
            <v-card flat class="main-card">
                <v-card-text>
                    <v-row>
                        <v-col>
                            <v-row>
                                <label class="title-row-label ml-5 mt-1">From</label>
                            </v-row>
                            <v-row align="center">
                                <v-text-field placeholder="0.00"
                                              @keypress="isNumber($event)"
                                              flat
                                              solo
                                              class="ml-2 field-sum"
                                              hide-details
                                              dark
                                              background-color="transparent"
                                              v-model="sum">
                                </v-text-field>
                                <v-spacer></v-spacer>
                                <div>
                                    <label class="max mr-5" @click="max">Max</label>
                                    <ItemSelector :selected-item.sync="currency" :items="currencies"/>
                                </div>
                            </v-row>
                            <v-row>
                                <label class="balance-label ml-5 mb-1">Balance: {{ maxResult }}</label>
                            </v-row>
                        </v-col>
                    </v-row>
                </v-card-text>
            </v-card>
        </v-row>

        <v-row class="mb-2">
            <v-col>
                <v-row justify="center" align="center">
                    <v-btn @click="showUnWrapView" class="swap-btn" icon>
                        <img width="24" height="24" :src="require('@/assets/icon/filter-exchange.svg')">
                    </v-btn>
                </v-row>
            </v-col>
        </v-row>

        <v-row dense>
            <v-card flat class="main-card">
                <v-card-text>
                    <v-row>
                        <v-col>
                            <v-row>
                                <label class="title-row-label ml-5 mt-1">To</label>
                            </v-row>
                            <v-row align="center">
                                <v-text-field placeholder="0.00"
                                              flat
                                              readonly
                                              solo
                                              class="ml-2 field-sum"
                                              hide-details
                                              dark
                                              background-color="transparent"
                                              v-model="sumResult">
                                </v-text-field>
                                <v-spacer></v-spacer>
                                <div class="mr-8">
                                    <ItemSelector :readonly="true" :selected-item="buyCurrency" :items="buyCurrencies"/>
                                </div>
                            </v-row>
                            <v-row>
                                <label class="balance-label ml-5 mb-1">Balance:
                                    {{ $utils.formatMoney(balance.wUsdPlus, 2) }}</label>
                            </v-row>
                        </v-col>
                    </v-row>
                </v-card-text>
            </v-card>
        </v-row>

        <v-row class="main-btn-row mb-2" align="center">
            <label class="exchange-label ml-5">Current index = {{ $utils.formatMoney(index, 2) }}</label>
            <v-spacer></v-spacer>
            <label class="exchange-label mr-5">1 USD+ = {{ $utils.formatMoney(Number.parseFloat(amountPerUsdPlus), 2) }} wUSD+ <img @click="showUnWrapView" class="exchange-label-icon" width="24" height="24" :src="require('@/assets/icon/filter-exchange.svg')"/></label>
        </v-row>

        <v-row class="main-btn-row" justify="center">
            <div class="mb-4" style="width: 96%;">
                <v-row justify="center">
                    <label class="action-info-label">{{ textInfoLabel }}</label>
                </v-row>
            </div>

            <div style="width: 96%;" v-if="!this.account">
                <v-btn dark
                       height="56"
                       class='buy disabled-buy'
                       @click="connectWallet">
                    {{ buttonLabel }}
                </v-btn>
            </div>

            <div style="width: 96%;" v-else>
                <v-btn dark
                       v-if="approved"
                       height="56"
                       class="buy"
                       :class="isBuy ? 'enabled-buy' : 'disabled-buy'"
                       :disabled="!isBuy"
                       @click="wrapAction">
                    Wrap
                </v-btn>

                <v-btn dark
                       v-else
                       height="56"
                       class="buy"
                       :class="isBuy ? 'enabled-buy' : 'disabled-buy'"
                       :disabled="!isBuy"
                       @click="approveAction">
                    Approve {{ currency.title }}
                </v-btn>
            </div>
        </v-row>

        <WaitingModal/>
        <SuccessModal/>
        <ErrorModal/>
    </v-col>
</template>

<script>
import {mapActions, mapGetters} from "vuex";
import ItemSelector from "../common/element/ItemSelector";
import ErrorModal from "@/components/common/modal/action/ErrorModal";
import WaitingModal from "@/components/common/modal/action/WaitingModal";
import SuccessModal from "@/components/common/modal/action/SuccessModal";
import BN from "bn.js";

export default {
    name: "Wrap",

    components: {
        ItemSelector,
        ErrorModal,
        WaitingModal,
        SuccessModal,
    },

    data: () => ({
        menu: false,
        tab: null,
        currency: {id: 'usdc'},

        currencies: [],

        sum: null,
        sumResult: null,

        buyCurrency: null,
        buyCurrencies: [{
            id: 'wUsdPlus',
            title: 'wUSD+',
            image: require('../../assets/wUsdPlus.svg')
        }],
    }),


    watch: {

        currency: function() {
            this.previewWrap();
        },

        sum: function (){
            this.previewWrap();
        }
    },


    computed: {

        ...mapGetters('accountData', ['balance', 'account']),
        ...mapGetters('accountUI', ['loadingBalance']),

        ...mapGetters('wrapData', ['index', 'amountPerUsdPlus']),
        ...mapGetters('wrapUI', ['usdcApproved', 'usdPlusApproved']),
        ...mapGetters("gasPrice", ["gasPriceGwei", "gasPrice", "gasPriceStation"]),

        ...mapGetters("transaction", ['transactions' ]),
        ...mapGetters("web3", ["web3", 'contracts']),


        estimateResult: function () {
            return this.sum * 0.9996;
        },

        estimateFee: function () {
            return this.sum * 0.0004;
        },


        tokenContract(){
            if (this.currency.id === 'usdc')
                return this.contracts.usdc;
            else
                return this.contracts.usdPlus;
        },


        buttonLabel: function () {

            if (!this.account) {
                return 'Connect to a wallet';
            } else if (this.isBuy) {
                if (this.usdcApproved || this.usdPlusApproved) {
                    return 'Wrap'
                } else {
                    if (this.currency.id === 'usdc')
                        return 'Approve USDC';
                    else if (this.currency.id === 'usdPlus')
                        return 'Approve USD+';
                }
            } else if (this.sum > parseFloat(this.balance.usdc)) {
                return 'Invalid amount'
            } else {
                return 'Enter an amount';
            }
        },


        maxResult: function () {
            return this.$utils.formatMoney(this.balance[this.currency.id], 2);
        },


        numberRule: function () {

            let v = this.sum;

            if (!v)
                return false;

            if (!v.trim()) return false;

            v = parseFloat(v.trim().replace(/\s/g, ''));

            if (!isNaN(parseFloat(v)) && v >= 0 && v <= parseFloat(this.balance[this.currency.id])) return true;

            return false;
        },

        textInfoLabel: function () {

            if (!this.account) {
                return 'You need to connect to a wallet';
            } else if (this.isBuy) {
                if (this.approved) {
                    return 'Press Wrap button to complete the process'
                } else {
                    return '';
                }
            } else if (this.sum > parseFloat(this.balance[this.currency.id])) {
                return 'Invalid amount'
            } else {
                return 'Enter an amount';
            }
        },

        approved: function () {
            if (this.currency.id === 'usdc') {
                return this.usdcApproved;
            } else if (this.currency.id === 'usdPlus') {
                return this.usdPlusApproved;
            } else {
                return false;
            }
        },

        isBuy: function () {
            return this.account && this.sum > 0 && this.numberRule;
        },
    },


    created() {
        this.currencies.push({
            id: 'usdc',
            title: 'USDC',
            image: require('../../assets/currencies/usdc.png')
        });
        this.currencies.push({
            id: 'usdPlus',
            title: 'USD+',
            image: require('../../assets/currencies/usdPlus.svg')
        });

        this.currency = this.currencies[0];

        this.buyCurrency = this.buyCurrencies[0];
    },

    methods: {

        ...mapActions("wrapData", ['refreshWrap']),
        ...mapActions("wrapUI", ['showUnWrapView', 'approveUsdc', 'approveUsdPlus']),

        ...mapActions("transaction", ['putTransaction']),
        ...mapActions("web3", ['connectWallet']),
        ...mapActions("gasPrice", ['refreshGasPrice']),

        ...mapActions("errorModal", ['showErrorModal']),
        ...mapActions("waitingModal", ['showWaitingModal', 'closeWaitingModal']),
        ...mapActions("successModal", ['showSuccessModal']),


        isNumber: function(evt) {
            evt = (evt) ? evt : window.event;
            let charCode = (evt.which) ? evt.which : evt.keyCode;

            if ((charCode > 31 && (charCode < 48 || charCode > 57)) && charCode !== 46) {
                evt.preventDefault();
            } else {
                if (charCode === 46 && (!this.sum || this.sum.includes('.'))) {
                    evt.preventDefault();
                } else {
                    return true;
                }
            }
        },


        async previewWrap() {

            if (!this.sum || this.sum === 0 || this.sum === '0') {
                this.sumResult = '0.00';
                return;
            }

            let sum = this.web3.utils.toWei(this.sum, 'mwei');
            let address = this.tokenContract.options.address;

            let value = await this.contracts.market.methods.previewWrap(address, sum).call();
            value = this.web3.utils.fromWei(value, 'mwei');
            this.sumResult = this.$utils.formatMoney(Number.parseFloat(value), 2);

        },

        async wrapAction() {
            try {
                let sum = this.web3.utils.toWei(this.sum, 'mwei');

                this.showWaitingModal();


                let estimatedGasValue = await this.estimateGas(sum);
                if (estimatedGasValue === -1) {
                    this.closeWaitingModal();
                    this.showErrorModal('estimateGas');
                } else {

                    await this.refreshGasPrice();

                    this.estimatedGas = estimatedGasValue;

                    /* adding 10% to estimated gas */
                    this.gas = new BN(Number.parseFloat(this.estimatedGas) * 1.1);
                    this.gasAmountInMatic = this.web3.utils.fromWei(this.gas.muln(Number.parseFloat(this.gasPrice)), "gwei");
                    this.gasAmountInUsd = this.web3.utils.fromWei(this.gas.muln(Number.parseFloat(this.gasPrice) * Number.parseFloat(this.gasPriceStation.usdPrice)), "gwei");


                    let wrapParams = {from: this.account, gasPrice: this.gasPriceGwei, gas: this.gas};
                    let wrapResult = await this.contracts.market.methods.wrap(this.tokenContract.options.address, sum, this.account).send(wrapParams);

                    this.closeWaitingModal();
                    this.showSuccessModal(wrapResult.transactionHash)
                }
            } catch (e) {
                console.log(e)
                this.closeWaitingModal();
                this.showErrorModal('estimateGas');
            }
        },

        async estimateGas(sum) {

            let contracts = this.contracts;
            let from = this.account;

            let result;


            try {
                let estimateOptions = {from: from, "gasPrice": this.gasPriceGwei};

                await contracts.market.methods.wrap(this.tokenContract.options.address, sum, this.account).estimateGas(estimateOptions)
                  .then(function (gasAmount) {
                      result = gasAmount;
                  })
                  .catch(function (error) {
                      console.log(error);
                      result = -1;
                  });
            } catch (e) {
                console.log(e);
                return -1;
            }

            return result;
        },


        approveAction: async function (){

            try {
                this.showWaitingModal('Approving in process');

                let approveSum = "10000000";

                let sum = this.web3.utils.toWei(approveSum, 'mwei');

                let allowApprove = await this.checkAllowance(sum);
                if (!allowApprove) {
                    this.closeWaitingModal();
                    this.showErrorModal('approve');
                    return;
                } else {

                    if (this.currency.id === 'usdc')
                        this.approveUsdc();
                    else if (this.currency.id === 'usdPlus')
                        this.approveUsdPlus();
                    else
                        throw new Error('Unknown currency');

                    this.closeWaitingModal();
                }
            } catch (e) {
                console.log(e)
                this.showErrorModal('approve');
            }

        },

        async checkAllowance(sum) {

            let contracts = this.contracts;
            let from = this.account;
            let self = this;

            let text = 'Approve ' + await this.tokenContract.methods.symbol().call();

            let allowanceValue = await this.tokenContract.methods.allowance(from, contracts.market.options.address).call();

            if (allowanceValue < sum) {
                try {
                    await this.refreshGasPrice();
                    let approveParams = {gasPrice: this.gasPriceGwei, from: from};

                    let tx = await this.tokenContract.methods.approve(contracts.market.options.address, sum).send(approveParams);

                    let minted = true;
                    while (minted) {
                        await new Promise(resolve => setTimeout(resolve, 2000));
                        let receipt = await this.web3.eth.getTransactionReceipt(tx.transactionHash);

                        if (receipt) {
                            if (receipt.status) {
                                let tx = {
                                    text: text,
                                    hash: receipt.transactionHash,
                                    pending: true,
                                };

                                self.putTransaction(tx);

                                return true;
                            } else {
                                return false;
                            }
                        }
                    }

                    return true;
                } catch (e) {
                    console.log(e)
                    return false;
                }
            }

            return true;
        },

        setSum(value) {
            this.sum = value;
        },

        max() {
            let balanceElement = this.balance[this.currency.id];
            this.sum = balanceElement + "";
        },
    }
}
</script>

<style scoped>

/* mobile */
@media only screen and (max-width: 1400px) {
    .max {
        display: none !important;
    }

    .exchange-row {
        padding-left: 8px;
        padding-right: 8px;
    }

    .buy {
        height: 46px !important;
    }

    .main-btn-row {
        margin-top: 28px;
    }
}

@media only screen and (min-width: 1400px) {
    .exchange-row {
        padding-left: 28px;
        padding-right: 28px;
    }

    .buy {
        height: 56px;
    }

    .main-btn-row {
        margin-top: 40px;
    }
}

.main-card {
    background: none !important;
    width: 100% !important;
    border: 1px solid #1D2029 !important;
    border-radius: 16px !important;
}

.title-row-label {
    color: white;
    font-style: normal;
    font-weight: 600;
    font-size: 14px;
    line-height: 24px;
}

.v-text-field >>> input {
    min-height: 42px !important;
    max-height: 42px !important;
    height: 42px !important;
}

.v-text-field >>> input::placeholder {
    color: #353E4C !important;
}

.v-text-field >>> input, .v-text-field >>> label, .v-text-field >>> button {
    font-family: 'Lato', sans-serif;
    font-style: normal !important;
    font-weight: 700 !important;
    font-size: 34px !important;
    line-height: 42px !important;

    color: #FE7F2D !important;
}

.field-sum {
    width: 45%;
}

.balance-label {
    color: white;
    font-family: 'Lato', sans-serif;
    font-style: normal;
    font-weight: 400;
    font-size: 14px;
    line-height: 24px;
}

.max {
    color: var(--link);
    font-style: normal;
    font-weight: 600;
    font-size: 14px;
    line-height: 24px;
    cursor: pointer;
}

.swap-btn {
}

.enabled-buy {
    background: var(--orange-gradient) !important;
}

.disabled-buy {
    background: #181A21 !important;
    box-shadow: none !important;
    border: 1px solid #1D2029 !important;
}

.buy {
    width: 100%;
    border-radius: 40px;
    color: white !important;
    font-family: 'Lato', sans-serif;
    font-style: normal;
    font-weight: 600;
    font-size: 16px !important;
    line-height: 24px !important;
    text-transform: none !important;
}

.container_body {
    border-radius: 24px;
    background-color: var(--secondary) !important;
}

.container_header {
    background-color: var(--secondary) !important;
}

.from-to-label {
    color: white;
    font-style: normal;
    font-weight: 600;
    font-size: 16px;
    line-height: 24px;
}

.from-to-sub-label {
    color: #6A84A0;
    font-style: normal;
    font-weight: 400;
    font-size: 14px;
    line-height: 16px;
}

.from-to-sub-sum {
    color: white;
    font-style: normal;
    font-weight: 700;
    font-size: 14px;
    line-height: 16px;
}

.currency-icon {
    width: 24px;
    height: 24px;
}

.add-info-label {
    color: white;
    font-style: normal;
    font-weight: 400;
    font-size: 16px;
    line-height: 24px;
}

.exchange-label {
    font-family: 'Lato', sans-serif;
    font-style: normal !important;
    font-weight: 400 !important;
    font-size: 14px !important;
    line-height: 24px !important;
    color: white !important;
}

.exchange-label-icon {
    cursor: pointer !important;
    vertical-align: bottom !important;
}

.action-info-label {
    font-family: 'Roboto', sans-serif;
    font-style: normal;
    font-weight: 400;
    font-size: 12px;
    line-height: 14px;
    color: #FE7F2D;
}
</style>
