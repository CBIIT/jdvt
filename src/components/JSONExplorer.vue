<template>
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
</template>

<script>
import jsonpath from "jsonpath";

export default {
  name: "JSONExplorer",
  props: {
    inputData: {
      type: Object,
      required: true
    }
  },
  data: function () { return {
    selectedColumnName: "",
    column_select: "all",
  }},
  computed: {
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
      let cols = this.columnNames;

      if(this.column_select === 'all') {
        return cols;
      } else if(this.column_select === 'no-blanks') {
        return cols.filter(colName => {
          return new Set(this.getColumnValues(colName)).size > 0;
        })
      }

      return cols
    },
    selectedColumnInfo: function() {
      return this.getColumnDescription(this.selectedColumnName);
    },
  },
  methods: {
    getColumnDescription: function(colName) {
      if (colName === '') return '';
      console.log(colName);
      let result = JSON.stringify([...new Set(this.getColumnValues(colName))], null, 2);
      return `${colName}: ${result || ''}`
    },
    getColumnValues: function(colName) {
      return jsonpath.query(this.inputData, colName)
          .filter(r => r !== null)
          .flatMap(v => {
            if (Array.isArray(v)) return v;
            return [v];
          });
    }
  }
}
</script>

<style scoped>

</style>
