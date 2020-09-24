<template>
  <div class="home">
    <prism-editor
      class="my-editor"
      v-model="code"
      :highlight="highlighter"
      line-numbers
    ></prism-editor>
    <div class="sidebar">
      <h2>Write and test your Smart Contract</h2>
    </div>
  </div>
</template>

<script>
// import Prism Editor
import { PrismEditor } from "vue-prism-editor";
import "vue-prism-editor/dist/prismeditor.min.css"; // import the styles somewhere

// import highlighting library (you can use any library you want just return html string)
import { highlight, languages } from "prismjs/components/prism-core";
import "prismjs/components/prism-clike";
import "prismjs/components/prism-javascript";
import "prismjs/themes/prism-tomorrow.css"; // import syntax highlighting styles

export default {
  components: {
    PrismEditor,
  },
  data: () => ({
    code: `/**
  * NAME: HELLOWORLD
  * DESCRIPTION: SIMPLE HELLO WORLD MODULE
  * AUTHOR: YOUR FABOULOUS NAME
  * VERSION: 1.0.0
  * IMMUTABLE: false
  */

  // DEFINING COMPILER VERSION
  /* Scrypta v0.0.1 */

  const time = new Date()
  let state = {}
  async function constructor() {
      
  }

  async function public: eachBlock(block){
      // THIS FUNCTION WILL BE CALLED FOR EACH NEW BLOCK!
      console.log(block)
  }

  async function public: ifMempool(mempool){
      // THIS FUNCTION WILL BE CALLED IF MEMPOOL IS FULL!
      console.log(mempool)
  }

  async function public: updatestatus(){
      // WORKING WITH DB IN STATELESS MODE
      let read = await db.read({"address": request.address})

      if(read.length === 0){
          await db.insert({ "address": request.address, score: 0 })
          read = await db.read({"address": request.address})
          console.log('CREATED', read)
      }

      let increment = read.score + 1
      await db.update({"address": request.address }, { $set: { score: increment } })
      read = await db.read({"address": request.address})
      console.log('UPDATED', read)
  }

  function private: inner(){
      return true
  }

  function public: helloworld(who) {
      return new Promise(async response => {
          if (who !== undefined && who.length > 0) {
              response('Hello ' + who + '! Your address is ' + request.address + '. Now are ' + time + '!')
          } else {
              response('Hello who?')
          }
      })
  }`,
  }),
  methods: {
    highlighter(code) {
      return highlight(code, languages.js); // languages.<insert language> to return html with markup
    },
  },
};
</script>

<style>
  body, html{
    overflow: hidden;
  }
  .my-editor {
    width:75%;
    float:left;
    background: #2d2d2d;
    color: #ccc;
    height: calc(100vh - 55px);
    /* you must provide font-family font-size line-height. Example: */
    font-family: Fira code, Fira Mono, Consolas, Menlo, Courier, monospace;
    font-size: 14px;
    line-height: 1.5;
    padding: 5px;
  }

  .sidebar {
    float:left;
    width:25%;
    background:#efefef;
    height: calc(100vh - 55px);
    padding:20px;
    text-align: left;
  }

  /* optional class for removing the outline */
  .prism-editor__textarea:focus {
    outline: none;
  }
</style>