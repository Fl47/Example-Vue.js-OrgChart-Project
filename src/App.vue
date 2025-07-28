<template>

  <head>
    <title>Org Chart by David</title>
    <link href="src/output.css" rel="stylesheet">
  </head>

  <div class="fixed top-0 left-0 right-0 z-10">
    <div v-if="showTop" class="bg-gradient-to-b from-white to-sky-50">

      <div class="py-10">
      <h1 class="text-slate-800 font-inter font-bold text-4xl text-center">Upload your org chart and <span class="text-blue-600">bring it to life</span></h1>
      </div>

      <div class="flex justify-center pb-4">
        <div class="flex justify-center rounded-lg border-2 border-dashed border-gray-900/25 max-w-fit">
          <div class="text-center p-2">
            <DocumentArrowUpIcon class="mx-auto h-12 w-12 text-gray-500"/>
            <div class="mt-2 flex text-sm/6 text-gray-600">
              <label for="file" class="mx-auto relative cursor-pointer rounded-md font-semibold text-sky-600 hover:text-sky-500">
                <span>Upload a file</span>
                <input @change='parseCSV' id='file' type='file' class="sr-only" />
              </label>
            </div>
            <p class="text-xs/5 text-gray-600">.csv, .txt or any file you want.</p>
          </div>
        </div>
      </div>
    </div>

    <div class="relative flex justify-center bg-transparent">
      <div class="inline-block mt-2">
      <ChevronDoubleUpIcon @click="showTop = false" v-if="showTop" class="mx-auto h-12 w-12 text-gray-500 rounded-full border-2 border-gray-900/25 bg-sky-50 hover:bg-sky-100"/>
      <ChevronDoubleDownIcon @click="showTop = true" v-else class="mx-auto h-12 w-12 text-gray-500 rounded-full border-2 border-gray-900/25 bg-sky-50 hover:bg-sky-100"/>
      </div>

      <div class="absolute left-4 top-3 bottom-3 flex flex-row gap-4">
      <button @click="$refs.chart.expandAll()" class="inline-flex items-center p-1 rounded-lg border-2 border-gray-900/25 bg-sky-50 hover:bg-sky-100">Expand All</button>
      <button @click="$refs.chart.collapseAll()" class="inline-flex items-center p-1 rounded-lg border-2 border-gray-900/25 bg-sky-50 hover:bg-sky-100">Collapse All</button>
      <button @click="$refs.chart.fitScreen()" class="inline-flex items-center p-1 rounded-lg border-2 border-gray-900/25 bg-sky-50 hover:bg-sky-100">Fit Screen</button>
      </div>
    </div>
  </div>

  <div>
    <OrgChart ref="chart" :data='orgData'/>
  </div>
</template>

<script setup>
// icons
import { DocumentArrowUpIcon, ChevronDoubleDownIcon, ChevronDoubleUpIcon } from "@heroicons/vue/24/outline";

// sample is a cut down version of the csv in json format
import json from '@/sample.json';

import Papa from 'papaparse';
import { ref, onMounted } from 'vue';
import OrgChart from './components/OrgChart.vue';
import * as d3 from 'd3';
import * as acc from 'accounting-js';

var showTop = ref(true);
var orgData = ref(null);
// this is used to avoid updating orgData.value and triggering OrgChart's watch when not needed
var tmpOrgData;

// set up example data (shown before uploading csv)
onMounted(() => {
  updateChart(json);
});

// start parsing when csv file selected
function parseCSV(event) {
    const csv_file = event.target.files[0];
    Papa.parse(csv_file, {header: true, dynamicTyping: true, skipEmptyLines: true, complete: parseComplete});
}

// after parsing, update chart
function parseComplete(results){
  updateChart(results.data);
  showTop = ref(false);
}

function updateChart(data){
  tmpOrgData = data.map(n => processNode(n));
  calculateChartValues(tmpOrgData);
  // Optional step to make money easier to read
  tmpOrgData = tmpOrgData.map(n => beautifyNode(n))
  orgData.value = tmpOrgData;
}

// every node needs 'id' and 'parentID'
function processNode(node){
  node['id'] = node['Employee Id'];
  node['parentId'] = node['Manager'];
  return node;
}

// creates d3 hierarchy from array to recursively calculate values
function calculateChartValues(orgArr){
  const root = d3.stratify()
    .id((d) => d.id)
    .parentId((d) => d.parentId)
  (orgArr);
  calculateNodeValues(root);
}

// Helper function for recursive calculations
// Assuming id is equal to index, if was not the case a map would be used for tmpOrgData instead
function calculateNodeValues(node){
  const children = node.children;

  // Base case (no children)
  if (children === undefined){
    // the d3 definition of "descendants" includes self
    tmpOrgData[node.data.id]['Descendants'] = 1;
    tmpOrgData[node.data.id]['Non Leaf Descendants'] = 0;
    tmpOrgData[node.data.id]['Management Cost'] = '$0';
    tmpOrgData[node.data.id]['IC Cost'] = tmpOrgData[node.data.id]['Salary'];
    tmpOrgData[node.data.id]['Total cost'] = tmpOrgData[node.data.id]['Salary'];
    tmpOrgData[node.data.id]['Management Cost Ratio'] = 0;
  }
  // Recursive Case
  else{
    // ensure all children have the values calculated
    // this is efficient because all nodes have only 1 parent, thus this is called only 1 time per node
    children.map(d => calculateNodeValues(d));

    tmpOrgData[node.data.id]['Descendants'] = 1;
    tmpOrgData[node.data.id]['Non Leaf Descendants'] = 1;
    let tempManagementCost = acc.unformat(tmpOrgData[node.data.id]['Salary']);
    let tempICCost = 0;
    let tempTotalCost = acc.unformat(tmpOrgData[node.data.id]['Salary']);

    children.forEach(child => {
      tmpOrgData[node.data.id]['Descendants'] += tmpOrgData[child.data.id]['Descendants'];
      tmpOrgData[node.data.id]['Non Leaf Descendants'] += tmpOrgData[child.data.id]['Non Leaf Descendants'];
      tempManagementCost += acc.unformat(tmpOrgData[child.data.id]['Management Cost'])
      tempICCost += acc.unformat(tmpOrgData[child.data.id]['IC Cost'])
      tempTotalCost += acc.unformat(tmpOrgData[child.data.id]['Total cost']);
      }
    );
    
    // convert costs to string
    tmpOrgData[node.data.id]['Management Cost'] = acc.format(tempManagementCost, {precision: 0})
    tmpOrgData[node.data.id]['IC Cost'] = acc.format(tempICCost, {precision: 0})
    tmpOrgData[node.data.id]['Total cost'] = acc.format(tempTotalCost, {precision: 0})
    tmpOrgData[node.data.id]['Management Cost Ratio'] = tempManagementCost / tempTotalCost;
  }
}

// there are 3 dollar ammounts to beautify
function beautifyNode(node){
    node['Management Cost'] = beautifyMoney(node['Management Cost'])
    node['IC Cost'] = beautifyMoney(node['IC Cost'])
    node['Total cost'] = beautifyMoney(node['Total cost'])
    node['Salary'] = beautifyMoney(node['Salary'])
    return node
  }

// makes the dollar ammounts easier to read
function beautifyMoney(money){
  let i = acc.unformat(money);
  if (i > 1e10){
    return acc.format(i/1e9, {precision: 0}) + "B"
  }
  else if (i > 1e9){
    return acc.format(i/1e9, {precision: 1}) + "B"
  }
  else if (i > 1e7){
    return acc.format(i/1e6, {precision: 0}) + "M"
  }
  else if (i > 1e6){
    return acc.format(i/1e6, {precision: 1}) + "M"
  }
  else if (i > 1e4){
    return acc.format(i/1e3, {precision: 0}) + "K"
  }
  else if (i > 1e3){
    return acc.format(i/1e3, {precision: 1}) + "K"
  }
  return money
}
</script>


