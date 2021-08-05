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
        <div class="row">
          <div class="col-5">
            <b-select class="m-auto col-12" v-model="column_select">
              <b-select-option value="all">All columns</b-select-option>
              <b-select-option value="no-blanks">Only non-blank columns</b-select-option>
            </b-select>
            <div class="overflow-auto">
              <b-select
                  select-size=10
                  class="m-auto col-12 overflow-scroll"
                  :options="filteredColumnNames" v-model="selectedColumnName">
              </b-select>
            </div>
            <p>
              {{filteredColumnNames.length}} columns displayed.
              <a target="_blank" href="https://www.npmjs.com/package/jsonpath">JSONPath</a> format used above.
            </p>
          </div>
          <div class="col-3">
            Selected rows
            <b-textarea rows=10 readonly v-model="selectedColumnInfo"></b-textarea>
          </div>
        </div>
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
        <b-form-textarea rows=20 readonly v-model="transformedJSONAsText"></b-form-textarea>
      </b-card>
  </div>
</template>

<script>
import jsonpath from 'jsonpath'
import lifter from 'jsonpath-lifter'

export default {
  name: 'App',
  data: function () {
    return {
      inputData: {},
      inputURL: "./examples/gdc-head-and-mouth.json",
      loadError: undefined,
      transformError: undefined,
      selectedColumnName: "",
      column_select: "all",
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
    columnNames: function () {
      // We generate a name in the form '.{key}.{key}...' for every column in this file.
      function get_column_names(prefix, dict) {
        if (Array.isArray(dict)) {
          return [...new Set(dict.flatMap((row) => get_column_names(prefix + '[*]', row)))]
              .sort()
        } else if (dict instanceof Object) {
          const prefixed_names = [];
          Object.keys(dict).forEach(key => {
            let key_name = `${prefix}.${key}`;
            prefixed_names.push(key_name);
            get_column_names(`${prefix}.${key}`, dict[key]).forEach(rowName => {
              prefixed_names.push(rowName);
            });
          });

          return [`${prefix}`, ...prefixed_names].sort();
        } else {
          return [];
        }
      }

      return get_column_names('$', this.inputData);
    },
    filteredColumnNames: function() {
      let cols = this.columnNames

      if(this.column_select == 'all') {
        return cols
      } else if(this.column_select == 'no-blanks') {
        return cols.filter(colName => {
          return new Set(this.getColumnValues(colName)).size > 0;
        })
      }

      return cols
    },
    selectedColumnInfo: function() {
      return this.getColumnDescription(this.selectedColumnName);
    },
    transformedJSONAsText: function() {
      try {
        let transformedJSON = lifter(this.lifterInstructions)(this.inputData);
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
    },
    getColumnDescription: function(colName) {
      if (colName == '') return '';
      console.log(colName);
      let result = JSON.stringify([...new Set(this.getColumnValues(colName))], null, 2);
      return `${colName}: ${result || ''}`
    },
    getColumnValues: function(colName) {
      let results = jsonpath.query(this.inputData, colName)
          .filter(r => r !== null)
          .flatMap(v => {
            if (Array.isArray(v)) return v;
            return [v];
          });
      return results;
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
