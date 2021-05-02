<template>
  <div id="app">
    <el-alert
      title=""
      type="success"
      v-if="txnHash.length > 0"
      >
    Success! TxnHash: {{txnHash}}</el-alert>
    <h2>Generate new seed or details</h2>
    <div class="input-form">
      <el-input
              placeholder="12-word seed"
              v-model="seed"
              clearable>
      </el-input>
    </div>
    <div class="buttons">
      <el-button type="primary" @click="generate_addresses" plain round>View Account Details</el-button>
      <el-button type="primary" @click="generate_seed" plain round>Generate New Seed</el-button>
    </div>
    <hr id="hr-top">
    <h2>Addresses, Keys and Balances of the seed</h2>
    <el-card class="box-card" style="margin-bottom: 2rem" v-if="showCard">
      <div v-for="o in addresses.length" :key="o" class="text item">
        <p><b>{{ o }}. Address</b>: {{addresses[o-1]}}</p>
        <p><b>Private Key</b>: {{privateKeys[o-1]}}</p>
        <p><b>Balance</b>: {{balances[o-1]}} ETH</p>
      </div>
    </el-card>
    <hr>
    <h2>Send Ether</h2>
    <div class="input-form">
      <el-input
              placeholder="From address"
              v-model="fromAddress"
              clearable>
      </el-input>
      <el-input
              class="input-form-el"
              placeholder="To address"
              v-model="toAddress"
              clearable>
      </el-input>
      <el-input
              class="input-form-el"
              placeholder="Ether"
              v-model="ether"
              clearable>
      </el-input>
    </div>
    <div class="buttons">
      <el-button type="primary" @click="send_ether" plain round>Send Ether</el-button>
    </div>
  </div>
</template>

<script>
import lightwallet from 'eth-lightwallet'
import Web3 from 'web3'
import HookedWeb3Provider from 'hooked-web3-provider'

export default {
  name: 'App',
  data() {
    return {
      seed: '',
      totalAddresses: 0,
      fromAddress: '',
      toAddress: '',
      ether: '',
      showCard: false,
      txnHash: '',
      finishSeed: '',
      addresses: [],
      privateKeys: [],
      balances: []
    }
  },
  methods: {
    generate_addresses: function() {
      this.addresses = []
      this.privateKeys = []
      this.balances = []
      if(!lightwallet.keystore.isSeedValid(this.seed)) {
        alert('Please enter a valid seed')
        return
      }
      this.totalAddresses = prompt("How many addresses do you want to generate?")
      if(!Number.isInteger(parseInt(this.totalAddresses))) {
        alert("Please enter valid number of addresses")
        return
      }
      var password = Math.random().toString()
      lightwallet.keystore.createVault({
        hdPathString: "m/0'/0'/0'",
        password: password,
        seedPhrase: this.seed
      }, (err, ks) => {
        ks.keyFromPassword(password, (err, pwDerivedKey) => {
          if(err) {
            console.log("ERROR")
          } else {
            ks.generateNewAddress(pwDerivedKey, this.totalAddresses)
            this.addresses = ks.getAddresses()
            var web3 = new Web3(new Web3.providers.HttpProvider("http://127.0.0.1:7545"))
            for(var count=0; count < this.addresses.length; count++) {
              var address = this.addresses[count]
              var private_key = ks.exportPrivateKey(address, pwDerivedKey)
              this.privateKeys.push(private_key)
              var balance = web3.eth.getBalance(address)
              this.balances.push(web3.fromWei(balance, "ether").toString())
              console.log('SEED', this.seed)
              console.log('ADDRESS', address)
              console.log('PRIVATE KEY', private_key)
              console.log('BALANCE', web3.fromWei(balance, "ether").toString())
            }
            this.finishSeed = this.seed
            this.showCard = true
          }
        })
      })
    },
    generate_seed: function() {
      this.seed = lightwallet.keystore.generateRandomSeed()
    },
    send_ether: function() {
      if(!lightwallet.keystore.isSeedValid(this.seed)) {
        alert('Please enter a valid seed')
        return
      }
      var password = Math.random().toString()
      lightwallet.keystore.createVault({
        hdPathString: "m/0'/0'/0'",
        password: password,
        seedPhrase: this.seed
      }, (err, ks) => {
        ks.keyFromPassword(password, (err, pwDerivedKey) => {
          if(err) {
            console.log("ERROR")
          } else {
            ks.generateNewAddress(pwDerivedKey, this.totalAddresses)
            ks.passwordProvider = function (callback) {
              callback(null, password)
            }
            var provider = new HookedWeb3Provider({
              host: 'http://127.0.0.1:7545',
              transaction_signer: ks
            })
            var web3 = new Web3(provider)
            web3.eth.sendTransaction({
              from: this.fromAddress,
              to: this.toAddress,
              value: this.ether,
              gas: 21000
            }, (error, result) => {
              if(error) {
                console.log("ERROR")
              } else {
                this.txnHash = result
                console.log("TXN HASH:", result)
              }
            })
          }
        })
      })
    }
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  width: 40%;
  margin: 60px auto;
}
.buttons {
  margin-top: 20px;
}
#hr-top {
  margin-top: 2rem;
}
.input-form-el {
  margin-top: 1rem;
}
hr {
  color: #D3D3D3;
}
p {
  text-align: left;
}
</style>
