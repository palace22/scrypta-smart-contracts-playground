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
          <b-input
            placeholder="Insert parameters"
            v-model="params"
            type="textarea"
          ></b-input>
          <br />
          <b-button size="is-medium" v-on:click="run" expanded type="is-primary"
            >RUN</b-button
          >
        </b-tab-item>

        <b-tab-item label="" icon="view-dashboard">
          <h1>Deployed contracts</h1>
        </b-tab-item>

        <b-tab-item label="" icon="help-circle">
          <h1>Informations</h1>
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
    selected: "eachBlock",
    isLogging: true,
    params: "",
    showdebug: true,
    debug: "> Playground ready, please run a function first.",
  }),
  async mounted() {
    const app = this;
    app.wallet = await app.scrypta.importBrowserSID();
    app.wallet = await app.scrypta.returnDefaultIdentity();
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
  },
  methods: {
    highlighter(code) {
      return highlight(code, languages.js);
    },
    selectfunction(what) {
      const app = this;
      app.selected = what;
    },
    async run() {
      const app = this;
      try {
        let codehex = Buffer.from(app.code).toString('hex')
        let result = await axios.post("http://localhost:4498/run", {
          address: "code:" + codehex,
          request: {
            function: app.selected,
            params: app.params
          },
        })
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
      if (newCode !== oldCode) {
        this.contract = await this.v001.compiler(newCode);
        delete this.contract.code;
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
  height: 30vh;
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
  height: calc(70vh - 55px);
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