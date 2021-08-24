<template>
  <div id="app">
    <h1>JSON Data Visualization and Transformation Tool (JDVT)</h1>

    <p></p>

      <b-card body-class="text-center" header-tag="nav" class="col-10 m-auto" no-body>
        <b-tabs card>
          <b-tab title="Load JSON from URL" active>
            <b-card-text>
                <b-input-group prepend="Load JSON from URL" class="mt-3">
                  <b-form-input
                    type="url"
                    v-model="inputURL"
                  ></b-form-input>
                  <b-input-group-append>
                    <b-button
                        variant="success"
                        @click="loadFromURL(inputURL)"
                    >Load</b-button>
                  </b-input-group-append>
                </b-input-group>
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
          <b-tab title="Import data from GDC">
            <b-card-text>
              <p>GDC queries use a <a
                  target="_blank"
                  href="https://docs.gdc.cancer.gov/API/Users_Guide/Search_and_Retrieval/"
              >JSON-based query format</a>.</p>

              <b-row class="mt-2">
                <textarea rows="10" class="col-12 m-auto" v-model.lazy="gdcQueryAsData"></textarea>
              </b-row>

              <b-row class="mt-2">
                <b-button variant="success"  @click="gdcQueryExecute()">Query</b-button>
              </b-row>
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

      <!-- Input data -->
      <JSONExplorer data-name="input data" :data="inputData" />

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
      <JSONExplorer data-name="transformed data" :data="transformedJSON" />

      <!-- JSON Schema to validate -->
      <b-card id="json-schema-card" header="JSON Schema" class="col-10 m-auto mt-2">
        <p>Include a JSON Schema to be used to validate the transformed data.</p>
        <b-textarea rows=20 v-model="jsonSchemaAsText"></b-textarea>
      </b-card>

      <b-card
        v-show="jsonSchemaError"
        bg-variant="danger"
        text-variant="white"
        class="col-10 m-auto my-2"
        header="Error reading JSON Schema"
      >
        {{ jsonSchemaError }}
      </b-card>

      <b-card id="json-schema-validation-card" header="JSON Schema validation results" class="col-10 m-auto mt-2">
        <b-textarea rows=20 v-model="jsonSchemaValidationResults" readonly></b-textarea>
      </b-card>
  </div>
</template>

<script>
import lifter from 'jsonpath-lifter'
import Ajv from 'ajv'
import JSONExplorer from "@/components/JSONExplorer";

export default {
  name: 'App',
  components: {JSONExplorer},
  data: function () {
    return {
      ajv: new Ajv({
        allErrors: true
      }),
      inputData: {},
      inputURL: "./examples/gdc-head-and-mouth.json",
      loadError: undefined,
      transformError: undefined,
      lifterInstructions: [{
        src: '$[*].diagnoses[*]',
        dst: '$',
        mv: true,
        via: [
          {src: '$.diagnosis_id', dst: '$.id'},
          {src: '$.age_at_diagnosis', dst: '$.age_at_diagnosis.valueDecimal', via: abc => parseInt(abc)},
          {dst: '$.age_at_diagnosis.unit', set: 'days'}
        ]
      }],
      jsonSchemaError: undefined,
      jsonSchema: {
        type: 'array',
        items: {
          type: 'object',
          properties: {
            id: {
              'type': 'string'
            },
            age_at_diagnosis: {
              'properties': {
                'valueDecimal': {
                  'type': 'number'
                },
                'unit': {
                  'type': 'string',
                  'enum': ['days']
                }
              },
            },
          },
          requiredProperties: false
        }
      },

      gdcQuery: {
        filters: {
          op: "in",
          content: {
            field: "cases.primary_site",
            value: [
              "baseoftongue",
              "floorofmouth",
              "gum",
              "hypopharynx",
              "larynx",
              "lip",
              "nasalcavityandmiddleear",
              "nasopharynx",
              "oropharynx",
              "otherandill-definedsitesinlip,oralcavityandpharynx",
              "otherandunspecifiedmajorsalivaryglands",
              "otherandunspecifiedpartsofmouth",
              "otherandunspecifiedpartsoftongue",
              "palate",
              "tonsil",
            ]
          },
        },
        expand: "diagnoses,samples",
        format: "JSON",
        size: 200,
      },
    };
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
    jsonSchemaAsText: {
      get: function() {
        return JSON.stringify(this.jsonSchema, null ,2);
      },
      set: function (text) {
        try {
          this.jsonSchema = JSON.parse(text);
          this.ajv.compile(this.jsonSchema);
          this.jsonSchemaError = undefined;
        } catch(err) {
          this.jsonSchemaError = `Could not parse JSON Schema instructions: ${err}`;
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
    },
    jsonSchemaValidationResults: function() {
      let validator = this.ajv.compile(this.jsonSchema);
      let valid = validator(this.transformedJSON);
      if (valid) return "No validation errors reported.";
      let errors_as_str = this.ajv.errorsText(validator.errors).replaceAll(', ', '\n - ');
      return `Validator errors:\n - ${errors_as_str}`;
    },

    gdcQueryAsData: {
      get() { return JSON.stringify(this.gdcQuery, undefined, 2); },
      set(str) {
        this.gdcQuery = JSON.parse(str);
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
    },
    gdcQueryExecute() {
      this.$http.get('https://api.gdc.cancer.gov/cases', {
        params: this.gdcQuery,
      }).then(response => {
        this.inputData = JSON.parse(response.bodyText);
      }, response => {
        this.loadError = `Could not load JSON from GDC: ${response.statusText}`;
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
