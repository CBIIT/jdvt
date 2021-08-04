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
        <div class="col-3">
          <b-select multiple class="m-auto col-12" :options="columnNames">
          </b-select>
        </div>
        <div class="col-9">

        </div>
      </b-card>
  </div>
</template>

<script>
export default {
  name: 'App',
  data: function () {
    return {
      inputData: {},
      inputURL: "./examples/gdc-head-and-mouth.json",
      loadError: undefined,
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
    columnNames: function () {
      // We generate a name in the form '.{key}.{key}...' for every column in this file.
      function get_column_names(prefix, dict) {
        if (Array.isArray(dict)) {
          return [...new Set(dict.flatMap((row) => get_column_names(prefix, row)))]
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
