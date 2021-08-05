<template>
  <div id="app">
    <h1>JSON Data Visualization and Transformation Tool (JDVT)</h1>

      <b-card body-class="text-center" header-tag="nav" class="col-10 m-auto" no-body>
        <b-tabs card>
          <b-tab title="Load JSON from URL" active>
            <b-card-text>
              <b-form inline>
                <b-input-group prepend="Load JSON from URL" class="mt-3">
                  <b-form-input
                    type="url"
                    v-model="inputURL"
                  ></b-form-input>
                  <b-input-group-append>
                    <b-button
                        variant="outline-success"
                        @click="loadFromURL(inputURL)"
                    >Load</b-button>
                  </b-input-group-append>
                </b-input-group>
              </b-form>
            </b-card-text>
          </b-tab>
          <b-tab title="Paste JSON">
            <b-card-text>
              <p>Paste your JSON content into the field below.</p>

              <b-row class="mt-2">
                <textarea rows="10" class="col-12 m-auto" v-model.lazy="inputDataAsText"></textarea>
              </b-row>
            </b-card-text>
          </b-tab>
          <b-tab title="Import data from CDA">
            <b-card-text>
              <p>We can use <a href="https://cda.cda-dev.broadinstitute.org/api/swagger-ui.html#/query/sqlQuery">the SQL query API request</a>.</p>
            </b-card-text>
          </b-tab>
        </b-tabs>
      </b-card>

      <b-card
          v-show="loadError"
          bg-variant="danger"
          text-variant="white"
          class="col-10 m-auto my-2"
          header="Error reading JSON data"
      >
        {{loadError}}
      </b-card>

      <!-- Columns -->
      <b-card id="columns" header="Columns" class="col-10 m-auto mt-2">
        <JSONExplorer :input-data="inputData" />
      </b-card>

      <!-- Transform instructions -->
      <b-card id="lifter-instructions-card" header="Transform" class="col-10 m-auto mt-2">
        <p>Providing lifting instructions in the format used by <a target="_blank" href="https://github.com/AndyA/jsonpath-lifter#README">jsonpath-lifter</a>.</p>
        <b-textarea rows=20 v-model="lifterInstructionsAsText"></b-textarea>
      </b-card>

      <b-card
            v-show="transformError"
            bg-variant="danger"
            text-variant="white"
            class="col-10 m-auto my-2"
            header="Error reading transform instructions"
        >
          {{transformError}}
        </b-card>

      <!-- Output JSON -->
      <b-card id="transformed-json" header="Transformed JSON" class="col-10 m-auto mt-2">
        <JSONExplorer :input-data="transformedJSON" />
      </b-card>
  </div>
</template>

<script>
import jsonpath from 'jsonpath'
import lifter from 'jsonpath-lifter'
import JSONExplorer from "@/components/JSONExplorer";

export default {
  name: 'App',
  components: {JSONExplorer},
  data: function () {
    return {
      inputData: {},
      inputURL: "./examples/gdc-head-and-mouth.json",
      loadError: undefined,
      transformError: undefined,
      lifterInstructions: [{
        src: '$[*].diagnoses[*]',
        dst: '$',
        mv: true,
        via: [
          { src: '$.diagnosis_id', dst: '$.id' },
          { src: '$.age_at_diagnosis', dst: '$.age_at_diagnosis.valueDecimal' },
          { dst: '$.age_at_diagnosis.unit', set: 'days' }
        ]
      }],
    }
  },
  computed: {
    inputDataAsText: {
      get: function () {
        return JSON.stringify(this.inputData, null, 2);
      },
      set: function (text) {
        try {
          this.inputData = JSON.parse(text);
          this.loadError = undefined;
        } catch(err) {
          this.loadError = `Could not parse JSON: ${err}`;
        }
      }
    },
    lifterInstructionsAsText: {
      get: function() {
        return JSON.stringify(this.lifterInstructions, null ,2);
      },
      set: function (text) {
        try {
          this.lifterInstructions = JSON.parse(text);
          this.transformError = undefined;
        } catch(err) {
          this.transformError = `Could not parse transform instructions: ${err}`;
        }
      }
    },
    transformedJSON: function() {
      let lift = lifter(this.lifterInstructions);
      return lift(this.inputData);
    },
    transformedJSONAsText: function() {
      try {
        let transformedJSON = this.transformedJSON;
        return JSON.stringify(transformedJSON, null, 2);
      } catch(err) {
        return `Could not transform data: ${err}`;
      }
    }
  },
  methods: {
    loadFromURL: function(url) {
      // Load JSON from the provided URL
      this.$http.get(url).then(response => {
        this.inputData = JSON.parse(response.bodyText)
      }, response => {
        this.loadError = `Could not load JSON from url ${url}: ${response.statusText}`;
      });
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
  margin-top: 60px;
}
</style>
