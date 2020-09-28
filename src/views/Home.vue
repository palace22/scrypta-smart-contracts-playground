<template>
  <div class="home">
    <b-loading
      :is-full-page="true"
      :active.sync="isLogging"
      :can-cancel="false"
    ></b-loading>
    <div class="sidebar">
      <b-tabs
        class="tabs-custom"
        type="is-toggle"
        :animated="false"
        vertical
        expanded
      >
        <b-tab-item label="" icon="play">
          <div
            style="float: right; cursor: pointer"
            v-on:click="createLocalContract"
          >
            <b-icon icon="plus" size="is-small"> </b-icon>
          </div>
          <div v-if="local.length > 0">
            <b> LOCAL CONTRACTS </b>
            <div
              v-for="contract in local"
              style="font-size: 13px; width: 100%; display: block"
              v-bind:key="contract"
            >
              <span
                style="cursor: pointer"
                v-on:click="selectContract(contract)"
                >{{ contract }}</span
              >
              <div
                style="float: right; cursor: pointer"
                v-on:click="deleteContract(contract)"
              >
                <b-icon icon="delete" size="is-small"> </b-icon>
              </div>
            </div>
            <hr style="margin: 5px 0; padding: 0" />
          </div>
          <h1>
            {{ contract.name }}
            <span style="font-size: 12px">v.{{ contract.version }}</span>
          </h1>
          <i>{{ contract.description }}</i>
          <hr />
          <div v-if="contract.functions" style="text-align: center">
            <div v-for="fn in contract.functions" v-bind:key="fn">
              <b-button
                v-if="selected !== fn"
                size="is-small"
                expanded
                style="text-align: left"
                v-on:click="selectfunction(fn)"
              >
                {{ fn }}
              </b-button>
              <b-button
                v-if="selected === fn"
                size="is-small"
                expanded
                type="is-danger"
                style="text-align: left"
              >
                {{ fn }}
              </b-button>
            </div>
          </div>
          <div style="position: relative">
            <b-input
              v-if="selected === 'eachBlock' || selected === 'ifMempool'"
              placeholder="Insert block to inject"
              v-model="block"
            ></b-input>
            <b-tooltip
              label="Inject last block"
              style="
                cursor: pointer;
                position: absolute;
                top: 7px;
                z-index: 99;
                right: 7px;
              "
              position="is-left"
            >
              <div v-on:click="getLastBlock">
                <b-icon icon="download-network"></b-icon>
              </div>
            </b-tooltip>
          </div>
          <b-input
            v-if="selected !== 'eachBlock' && selected !== 'ifMempool'"
            placeholder="Insert parameters"
            v-model="params"
            type="textarea"
          ></b-input>
          <br />
          <b-button size="is-medium" v-on:click="run" expanded type="is-primary"
            >RUN</b-button
          >
          <b-button
            size="is-medium"
            type="is-success"
            style="border-radius: 0px; position: absolute; bottom: 0; left: 0"
            v-on:click="deploy"
            v-if="!isDeploying"
            expanded
            >DEPLOY</b-button
          >
        </b-tab-item>

        <b-tab-item label="" icon="view-dashboard">
          <h1>Your contracts</h1>
        </b-tab-item>

        <b-tab-item label="" icon="compass">
          <h1>Discovery contracts</h1>
        </b-tab-item>
      </b-tabs>
    </div>
    <prism-editor
      v-bind:class="{ 'editor-debug': showdebug }"
      class="contract-editor"
      v-model="code"
      :highlight="highlighter"
      line-numbers
    ></prism-editor>
    <div v-if="showdebug" class="debugger" id="debugger">
      <div class="debug" v-html="debug"></div>
    </div>
  </div>
</template>

<script>
import { PrismEditor } from "vue-prism-editor";
import "vue-prism-editor/dist/prismeditor.min.css";
import { highlight, languages } from "prismjs/components/prism-core";
import "prismjs/components/prism-clike";
import "prismjs/components/prism-javascript";
import "prismjs/themes/prism-tomorrow.css";
const hello_world = require("../hello_world.js");
const ScryptaCore = require("@scrypta/core");
const ScryptaCompiler = require("@scrypta/compiler");
var CoinKey = require("coinkey");
const axios = require("axios");
const LZUTF8 = require('lzutf8');

export default {
  components: {
    PrismEditor,
  },
  data: () => ({
    code: hello_world.code,
    identity: {},
    scrypta: new ScryptaCore(true),
    v001: ScryptaCompiler.v001,
    coinkey: CoinKey,
    contract: {},
    showSave: false,
    selected: "eachBlock",
    lastContract: "HELLOWORLD",
    isLogging: true,
    params: "",
    block: "",
    showdebug: true,
    isDeploying: false,
    local: [],
    session: {},
    debug: "> Playground ready, please run a function first.",
  }),
  async mounted() {
    const app = this;
    app.wallet = await app.scrypta.importBrowserSID();
    app.wallet = await app.scrypta.returnDefaultIdentity();
    let address = await app.scrypta.createAddress("SESSION");
    app.session = address;
    if (localStorage.getItem("contracts") !== null) {
      let stored = JSON.parse(localStorage.getItem("contracts"));
      if (Object.keys(stored).length > 0) {
        let contracts = Object.keys(stored);
        app.local = contracts;
        app.lastContract = app.local[0];
        app.code = Buffer.from(stored[app.local[0]], "hex").toString("utf-8");
      }
    }
    if (app.wallet.length > 0) {
      let SIDS = app.wallet.split(":");
      app.address = SIDS[0];
      let identity = await app.scrypta.returnIdentity(app.address);
      app.wallet = identity;
      app.isLogging = false;
    } else {
      app.isLogging = false;
    }
    app.contract = await app.v001.compiler(app.code);
    delete app.contract.code;
    this.showSave = false;
  },
  methods: {
    deploy() {
      const app = this;
      if (app.isDeploying === false) {
        app.isDeploying = true;
        app.$buefy.dialog.prompt({
          message: `Inserisci la password del wallet`,
          inputAttrs: {
            type: "password",
          },
          trapFocus: true,
          onConfirm: async (password) => {
            let key = await app.scrypta.readKey(password, app.wallet.wallet);
            if (key !== false) {
              let pubkey = await app.scrypta.getPublicKey(key.prv);
              let address = await app.scrypta.getAddressFromPubKey(pubkey);
              app.log("AUTHOR ADDRESS IS: " + address);
              let balance = await app.scrypta.get("/balance/" + address);
              const hash = await app.scrypta.hash(
                key.prv + ":" + app.contract.name
              );
              let contract = new CoinKey(Buffer.from(hash, "hex"), {
                private: 0xae,
                public: 0x30,
                scripthash: 0x0d,
              });
              app.log("CONTRACT ADDRESS IS: " + contract.publicAddress);
              app.contract.v = 1
              app.contract.address = contract.publicAddress
              let manifest = app.contract
              manifest.code = LZUTF8.compress(app.code, { outputEncoding: 'Base64' })
              let sid = await app.scrypta.buildWallet(
                "TEMPORARY",
                contract.publicAddress,
                {
                  prv: contract.privateWif,
                  key: contract.publicKey.toString("hex"),
                },
                false
              );
              if (balance.balance > 0.011) {
                let genesis_check = await app.scrypta.post("/read", {
                  address: manifest.address,
                  refID: "genesis",
                  protocol: "ida://",
                });
                if (genesis_check.data[0] !== undefined) {
                  let genesis = genesis_check.data[0].data;
                  genesis.contract = JSON.parse(genesis.message);
                  if (
                    genesis.contract.name === manifest.name &&
                    genesis.contract.address === manifest.address
                  ) {
                    app.log("GENESIS EXIST, CHECK IF EXIST VERSION.");
                    let version_check = await app.scrypta.post("/read", {
                      address: manifest.address,
                      refID: manifest.version,
                      protocol: "ida://",
                    });
                    if (
                      version_check.data[0] === undefined &&
                      genesis.contract.version !== manifest.version
                    ) {
                      app.log("PUBLISHING UPDATE " + manifest.version);
                      if (
                        genesis.contract.immutable === undefined ||
                        genesis.contract.immutable === "false"
                      ) {
                        let signed = await app.scrypta.signMessage(
                          key.prv,
                          JSON.stringify(manifest)
                        );
                        let contractBalance = await app.scrypta.get(
                          "/balance/" + manifest.address
                        );
                        let funded = false;
                        if (contractBalance.balance < 0.002) {
                          funded = await app.fundAddress(manifest.address, key.prv);
                        } else {
                          funded = true;
                        }
                        if (funded !== false) {
                          let update_written = await app.scrypta.write(
                            sid,
                            "TEMPORARY",
                            JSON.stringify(signed),
                            "",
                            manifest.version,
                            "ida://"
                          );
                          if (
                            update_written.uuid !== undefined &&
                            update_written.txs !== undefined &&
                            update_written.txs.length > 0
                          ) {
                            app.log("UPDATE TRANSACTION WRITTEN CORRECTLY");
                            app.isDeploying = false;
                          } else {
                            app.isDeploying = false;
                            app.error("ERROR WHILE CREATING TRANSACTION");
                          }
                        } else {
                          app.isDeploying = false;
                          app.log("ERROR WHILE FUNDING ADDRESS");
                        }
                      } else {
                        app.isDeploying = false;
                        app.log("CAN'T UPDATE IMMUTABLE CONTRACT!");
                      }
                    } else {
                      app.isDeploying = false;
                      app.log("VERSION EXIST, PLEASE UPDATE CONTRACT FIRST");
                    }
                  } else {
                    app.isDeploying = false;
                    app.log("GENESIS TRANSACTION DOESN'T MATCH.");
                  }
                } else {
                  app.log("SIGNING GENESIS TRANSACTION.");
                  let signed = await app.scrypta.signMessage(
                    key.prv,
                    JSON.stringify(manifest)
                  );
                  let contractBalance = await app.scrypta.get(
                    "/balance/" + manifest.address
                  );
                  let funded = false;
                  if (contractBalance.balance < 0.01) {
                    funded = await app.fundAddress(manifest.address, key.prv);
                  } else {
                    funded = true;
                  }
                  if (funded !== false) {
                    let genesis_written = await app.scrypta.write(
                      sid,
                      "TEMPORARY",
                      JSON.stringify(signed),
                      "",
                      "genesis",
                      "ida://"
                    );
                    if (
                      genesis_written.uuid !== undefined &&
                      genesis_written.txs !== undefined &&
                      genesis_written.txs.length > 0
                    ) {
                      app.isDeploying = false;
                      app.log("GENESIS TRANSACTION WRITTEN CORRECTLY");
                    } else {
                      app.isDeploying = false;
                      app.error("ERROR WHILE CREATING TRANSACTION");
                    }
                  } else {
                    app.isDeploying = false;
                    app.log("ERROR WHILE FUNDING ADDRESS");
                  }
                }
              } else {
                app.isDeploying = false;
                app.log("NOT ENOUGH BALANCE");
              }
            } else {
              app.isDeploying = false;
              app.log("WRONG PASSWORD!");
            }
          },
        });
      }
    },
    async fundAddress(contractAddress, privkey) {
      const app = this;
      return new Promise(async (response) => {
        let funded = false;
        let success = false;
        let retries = 0;
        while (funded === false) {
          let pubkey = await app.scrypta.getPublicKey(privkey);
          let address = await app.scrypta.getAddressFromPubKey(pubkey);
          let sid = await app.scrypta.buildWallet(
            "TEMPORARY",
            address,
            { prv: privkey, key: pubkey },
            false
          );
          let sent = await app.scrypta.send(
            sid,
            "TEMPORARY",
            contractAddress,
            0.01
          );
          if (sent !== false && sent !== null && sent.length === 64) {
            funded = true;
            success = true;
          }
          retries++;
          if (retries > 10) {
            funded = true;
          }
        }
        if (funded === true && success === true) {
          setTimeout(function () {
            response(true);
          }, 1000);
        } else {
          response(false);
        }
      });
    },
    async getLastBlock() {
      const app = this;
      let info = await app.scrypta.get("/wallet/getinfo");
      app.block = info.blocks;
    },
    highlighter(code) {
      return highlight(code, languages.js);
    },
    async selectContract(contract) {
      const app = this;
      let stored = JSON.parse(localStorage.getItem("contracts"));
      app.lastContract = contract;
      app.code = Buffer.from(stored[app.lastContract], "hex").toString("utf-8");
      app.contract = await app.v001.compiler(app.code);
      delete app.contract.code;
      setTimeout(function () {
        app.showSave = false;
      }, 10);
    },
    createLocalContract() {
      const app = this;
      if (localStorage.getItem("contracts") === null) {
        localStorage.setItem("contracts", JSON.stringify({}));
      }
      let stored = JSON.parse(localStorage.getItem("contracts"));
      let contractName = "NEW CONTRACT";
      stored[contractName] =
        "2f2a2a0a20202a204e414d453a204e455720434f4e54524143540a20202a204445534352495054494f4e3a204120434f4e54524143540a20202a20415554484f523a20594f555220464142554c4f5553204e414d450a20202a2056455253494f4e3a20312e302e300a20202a20494d4d555441424c453a2066616c73650a20202a2f0a0a2f2f20444546494e494e4720434f4d50494c45522056455253494f4e0a2f2a20536372797074612076302e302e31202a2f0a0a6173796e632066756e6374696f6e20636f6e7374727563746f722829207b0a0a7d0a66756e6374696f6e207075626c69633a2065616368426c6f636b28626c6f636b297b0a20200a7d0a66756e6374696f6e207075626c69633a2069664d656d706f6f6c28626c6f636b297b0a20200a7d";
      localStorage.setItem("contracts", JSON.stringify(stored));
      app.selectContract("NEW CONTRACT");
      let contracts = Object.keys(stored);
      app.local = contracts;
    },
    deleteContract(contract) {
      const app = this;
      return new Promise((response) => {
        if (app.contract.name !== undefined) {
          let stored = JSON.parse(localStorage.getItem("contracts"));
          delete stored[contract];
          localStorage.setItem("contracts", JSON.stringify(stored));
          stored = JSON.parse(localStorage.getItem("contracts"));
          let contracts = Object.keys(stored);
          app.local = contracts;
          response(true);
        }
      });
    },
    save() {
      const app = this;
      return new Promise((response) => {
        if (app.contract.name !== undefined) {
          let codehex = Buffer.from(app.code).toString("hex");
          if (localStorage.getItem("contracts") === null) {
            localStorage.setItem("contracts", JSON.stringify({}));
          }
          let stored = JSON.parse(localStorage.getItem("contracts"));
          stored[app.contract.name] = codehex;
          localStorage.setItem("contracts", JSON.stringify(stored));
          stored = JSON.parse(localStorage.getItem("contracts"));
          let contracts = Object.keys(stored);
          app.local = contracts;
          response(true);
        }
      });
    },
    selectfunction(what) {
      const app = this;
      app.params = "";
      app.selected = what;
    },
    async run() {
      const app = this;
      try {
        let codehex = Buffer.from(app.code).toString("hex");
        if (
          (app.selected === "eachBlock" || app.selected === "ifMempool") &&
          app.block !== undefined &&
          parseInt(app.block) > 0
        ) {
          let block = await app.scrypta.get("/analyze/" + app.block);
          app.params = block.data;
        }
        let requesthex = Buffer.from(
          JSON.stringify({ function: app.selected, params: app.params })
        ).toString("hex");
        let request = await app.scrypta.signMessage(
          app.session.prv,
          requesthex
        );

        let result = await axios.post("http://104.248.90.33:4498/run", {
          address: "code:" + codehex,
          request: request,
        });

        app.log(JSON.stringify(result.data));
      } catch (e) {
        app.log(e);
      }
    },
    log(what) {
      this.debug += `<br>> ` + what;
      var objDiv = document.getElementById("debugger");
      setTimeout(function () {
        objDiv.scrollTop = 9999999999999999999999;
      }, 1);
    },
  },
  watch: {
    code: async function (newCode, oldCode) {
      const app = this;
      if (newCode !== oldCode) {
        let oldContract = await app.v001.compiler(oldCode);
        app.contract = await this.v001.compiler(newCode);
        delete this.contract.code;
        if (
          oldContract.name !== app.contract.name &&
          app.lastContract === oldContract.name
        ) {
          await this.deleteContract(oldContract.name);
          app.lastContract = app.contract.name;
        }
        app.save();
      }
    },
  },
};
</script>

<style>
body,
html {
  overflow: hidden;
}
.debugger {
  position: fixed;
  width: 75%;
  height: 25vh;
  background: #000;
  color: #00ff00;
  bottom: 0;
  right: 0;
  text-align: left;
  overflow-y: scroll;
}
.debugger .debug {
  font-size: 12px;
  padding: 10px 10px 30px 10px;
  font-family: Fira code, Fira Mono, Consolas, Menlo, Courier, monospace;
  word-break: break-all;
}
.contract-editor {
  width: 75%;
  float: left;
  background: #2d2d2d;
  color: #ccc;
  height: calc(100vh - 55px);
  font-family: Fira code, Fira Mono, Consolas, Menlo, Courier, monospace;
  font-size: 14px;
  line-height: 1.5;
  padding: 5px;
}
.editor-debug {
  height: calc(75vh - 55px);
}
.icon {
  margin-right: 0px !important;
}
.tabs a {
  padding: 0 10px !important;
  border-radius: 0 !important;
}
.sidebar {
  float: left;
  width: 25%;
  background: #efefef;
  height: calc(100vh - 55px);
  text-align: left;
}

.tabs-custom {
  height: calc(100vh - 55px);
}

/* optional class for removing the outline */
.prism-editor__textarea:focus {
  outline: none;
}
</style>