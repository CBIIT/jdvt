<template>
  <b-card body-class="text-center"
          class="col-10 m-auto mt-2"
          no-body>
    <b-tabs card>
      <b-tab :title="'Display ' + dataName + ' as JSON'" active>
        <b-textarea rows=20 v-model="dataAsText"></b-textarea>
      </b-tab>
      <b-tab :title="'Display ' + dataName + ' as fields'">
        <div class="row">
          <div class="col-5">
            <b-select class="m-auto col-12" v-model="field_select">
              <b-select-option value="all">All fields</b-select-option>
              <b-select-option value="no-blanks">Only non-blank fields</b-select-option>
            </b-select>
            <div class="overflow-auto">
              <b-select
                  select-size=10
                  class="m-auto col-12 overflow-scroll"
                  :options="filteredFieldNames" v-model="selectedFieldName">
              </b-select>
            </div>
            <p>
              {{filteredFieldNames.length}} fields displayed.
              <a target="_blank" href="https://www.npmjs.com/package/jsonpath">JSONPath</a> format used above.
            </p>
          </div>
          <div class="col-7">
            <div v-if="!selectedFieldName">
              <p>Select a field for more information.</p>
            </div>
            <div v-else-if="getFieldValues(selectedFieldName).length === 0">
              <p>No values present for field {{selectedFieldName}}.</p>
            </div>
            <p v-if="recordCount > 0">The input contains {{recordCount}} top-level records.</p>
            <b-table :items="getFieldValueCounts(selectedFieldName)" v-if="selectedFieldName" bordered striped>
            </b-table>
          </div>
        </div>
      </b-tab>
      <b-tab :title="'Display ' + dataName + ' as RDF'">
      </b-tab>
    </b-tabs>
  </b-card>
</template>

<script>
import jsonpath from "jsonpath";

export default {
  name: "JSONExplorer",
  props: {
    data: {
      type: Object,
      required: true
    },
    dataName: {
      type: String,
      default: "data"
    }
  },
  data: function () { return {
    selectedFieldName: "",
    field_select: "all",
  }},
  computed: {
    fieldNames: function () {
      // We generate a name in the form '.{key}.{key}...' for every field in this file.
      function get_field_names(prefix, dict) {
        if (Array.isArray(dict)) {
          return [...new Set(dict.flatMap((row) => get_field_names(prefix + '[*]', row)))]
              .sort()
        } else if (dict instanceof Object) {
          const prefixed_names = [];
          Object.keys(dict).forEach(key => {
            let key_name = `${prefix}.${key}`;
            prefixed_names.push(key_name);
            get_field_names(`${prefix}.${key}`, dict[key]).forEach(rowName => {
              prefixed_names.push(rowName);
            });
          });

          return [`${prefix}`, ...prefixed_names].sort();
        } else {
          return [];
        }
      }

      return get_field_names('$', this.data);
    },

    filteredFieldNames: function() {
      let cols = this.fieldNames;

      if(this.field_select === 'all') {
        return cols;
      } else if(this.field_select === 'no-blanks') {
        return cols.filter(colName => {
          return new Set(this.getFieldValues(colName)).size > 0;
        })
      }

      return cols
    },

    recordCount() {
      if (Array.isArray(this.data)) {
        return this.data.length;
      }
      return undefined;
    },

    dataAsText() { return JSON.stringify(this.data, null, 2); },
  },
  methods: {
    getFieldValues: function(colName) {
      if(!colName) return [];

      return jsonpath.query(this.data, colName)
          .filter(r => r !== null)
          .flatMap(v => {
            if (Array.isArray(v)) return v;
            return [v];
          });
    },
    getFieldValueCounts: function(colName) {
      let result = this.getFieldValues(colName);
      let uniqs = result.reduce((acc, val) => {
        if (typeof(val) === 'object') {
          val = JSON.stringify(val, undefined, 2);
        }

        acc[val] = acc[val] === undefined ? 1 : acc[val] += 1;
        return acc;
      }, {});
      console.log(uniqs);
      return Object.keys(uniqs).map(key => {
        let row = {
          value: key,
          count: uniqs[key],
        };

        return row;
      }).sort(v => v.count);
    }
  }
}
</script>

<style scoped>

</style>
