<template>
  <div ref="chart" class="fixed top-0 left-0 right-0 bottom-0">
  </div>

  <div v-if="highlightedNode" class="fixed items-center left-0 top-0 bottom-0 flex justify-center">
    <div class="justify-center min-h-[700px] w-[450px] ml-10 bg-sky-50 rounded-lg border-[3px] border-slate-500 ">
      <div class="flex justify-center">
        <img :src="`https://ui-avatars.com/api/?name=${highlightedNode['Name'].replace(' ', '+')}&background=random&rounded=true`"/>
      </div>
      <p class="font-inter text-2xl font-medium text-center">{{highlightedNode['Name']}}</p>
      <p class="font-inter text-xl text-center text-slate-700">{{highlightedNode["Job Title"].length < 40 ? highlightedNode["Job Title"] : highlightedNode["Job Title"].substring(0,37) + "..."}}</p>
      <div class="flex justify-center pt-1">
        <svg height="25" width="150" class="fill-rose-300  bg-green-300 rounded-xl">
        <rect height="25" :width="`${150 * highlightedNode['Management Cost Ratio']}`"/>
        </svg>
      </div>
      <p class="font-inter text-base text-center -mt-6 text-slate-800">Cost Ratio: {{Math.round(highlightedNode["Management Cost Ratio"]*100) / 100}}</p>
      <div class="font-inter text-lg text-left text-slate-500 px-4 pt-1">
        <p>Management Cost: <span class="text-rose-500">{{highlightedNode["Management Cost"]}}</span></p>
        <p>IC Cost: <span class="text-green-600">{{highlightedNode["IC Cost"]}}</span></p>
        <p>Total cost: <span class="text-sky-600">{{highlightedNode["Total cost"]}}</span></p>
        <p>Salary: <span class="text-violet-600">{{highlightedNode["Salary"]}}</span></p>

        <p class="text-xl text-slate-900 font-medium text-center py-1">{{highlightedNode["Department"]}}</p>
        <p>Entity: <span class="text-slate-700">{{highlightedNode["Entity"]}}</span></p>
        <p>Project: <span class="text-slate-700">{{highlightedNode["Project"]}}</span></p>
        <p>Skill: <span class="text-slate-700">{{highlightedNode["Skill"]}}</span></p>
        <p>Performance: <span class="text-slate-700">{{highlightedNode["Performance"]}}</span></p>
        <p>Status: <span :class="`text-${highlightedNode['Status'] == 'Active' ? 'green-600' : 'rose-500'}`">{{highlightedNode["Status"]}}</span></p>

        <p class="text-xl text-slate-900 font-medium text-center py-1">{{highlightedNode["Company Cluster"]}}</p>
        <p>Company: <span class="text-slate-700">{{highlightedNode["Company"]}}</span></p>
        <p>Company Hierarchy: <span class="text-slate-700">{{highlightedNode["Company Hierarchy"]}}</span></p>
        <p>Cost Center: <span class="text-slate-700">{{highlightedNode["Cost Center"]}}</span></p>
        <p>Management Level: <span :class="`text-${highlightedNode['Management Level'] == 'Individual Contributor' ? 'green-600' : 'rose-500'}`">{{highlightedNode["Management Level"]}}</span></p>

      </div>
      
    </div>
  </div>

</template>
  
<script setup>
  import { ref, onMounted, watch} from 'vue';
  import { OrgChart } from 'd3-org-chart';
  import * as d3 from 'd3';
  
  const props = defineProps({
    data: {
      type: Array,
      required: true,
    },
  });
  
  const chart = ref(null);
  var orgChartInstance = null;
  var highlightedNode = ref(null);

    function createChart() {
    // If there's already an instance, clear it before creating a new one
    if (orgChartInstance) orgChartInstance.clear();

    orgChartInstance = new OrgChart()
    .container(chart.value)
    .data(props.data)
    .svgHeight(window.innerHeight)
    .nodeWidth((d) => 250)
    .nodeHeight((d) => 500)
    .childrenMargin((d) => 80)
    .compact(false)
      //clicking on a node toggles highlighting
    .onNodeClick((d) => {
      d.data._highlighted 
      ?
      clearHighlight()
      :
      setHighlight(d.data);
    })
    // the arrow function binds "this" so we use a traditional anonymous function instead
    // remove the default highlight border
    .nodeUpdate(function (d, i, arr) {
      d3.select(this).select(".node-rect").attr("stroke", "none");
    })
    // change default connecting lines
    .linkUpdate(function (d, i, arr) {
      d3.select(this)
        // the color is tailwind slate-400 to keep in theme with the other colors
        .attr("stroke", "#94a3b8")
        .attr("stroke-width", 2)
    })
    .nodeContent((d) => {
      // Custom node content for each node
      // Some of the api calls for the photos did not work so I made my own
      return `
        <div class="flex justify-center -mb-6">
          <img src="https://ui-avatars.com/api/?name=${d.data['Name'].replace(' ', '+')}&background=random&rounded=true" />
        </div>
        ${d.data._directSubordinates == 0
        ? 
        `<div class="justify-center rounded-lg  ${d.data._highlighted 
          ? 
          `border-[8px] border-sky-600 -m-[5px] pt-[19px]`
          :
          `border-[3px] border-dashed border-emerald-400 pt-[24px]`
          } h-[410px]">` 
        : 
        `<div class="justify-center rounded-lg ${d.data._highlighted 
          ? 
          `border-[8px] border-sky-600 -m-[5px] pt-[19px]`
          :
          `border-[3px] border-dashed border-slate-400 pt-[24px]`
          } h-[450px]">`
        }
          <p class="font-inter text-2xl font-medium text-center">${d.data['Name']}</p>
          <p class="font-inter text-xl text-center text-slate-700">${d.data["Job Title"].length < 40 ? d.data["Job Title"] : d.data["Job Title"].substring(0,37) + "..."}</p>
          <div class="flex justify-center pt-1">
            <svg height="25" width="150" class="fill-rose-300  bg-green-300 rounded-xl">
            <rect height="25" width="${150 * d.data["Management Cost Ratio"]}"/>
            </svg>
          </div>
          <p class="font-inter text-base text-center -mt-6 text-slate-800">Cost Ratio: ${Math.round(d.data["Management Cost Ratio"]*100) / 100}</p>
          <div class="font-inter text-lg text-left text-slate-500 px-4 pt-1">
            <p>Management Cost: <span class="text-rose-500">${d.data["Management Cost"]}</span></p>
            <p>IC Cost: <span class="text-green-600">${d.data["IC Cost"]}</span></p>
            <p>Total cost: <span class="text-sky-600">${d.data["Total cost"]}</span></p>
            <p class="text-xl text-slate-900 text-center py-1">${d.data["Department"]}</p>
            <p>Location: <span class="text-slate-700">${d.data["Location"]}</span></p>
            <p>level: <span class="text-slate-700">${d.data["level"]}</span></p>
          </div>
        </div>
      `;})
    .nodeButtonHeight((d) => 100)
    .nodeButtonWidth((d) => 200)
    .nodeButtonY((d) => -50)
    .nodeButtonX((d) => -93)
    .buttonContent(({ node, state }) => {
      // Custom button content, contains Non Leaf Descendants and Total Descendants
      return `
        <div>
          <div class="bg-sky-50 p-2 rounded-full border-gray-900/25 border-2">
            <p class="font-inter text-xl text-center">${node.data['Non Leaf Descendants']} / ${node.data['Descendants']}</p>
            <div class="flex justify-center">
              ${node.children
                ? 
                `<svg height="25" width="40" class="fill-none stroke-[4] stroke-green-500">
                <polyline points="0,25 20,5 40,25"/>
                </svg>`
                : 
                `<svg height="25" width="40" class="fill-none stroke-[4] stroke-rose-500">
                <polyline points="0,0 20,20 40,0"/>
                </svg>`
              }
              
            </div>
          </div>
          <pre class="font-inter text-xl text-center">                                 </pre>
        </div>
      `;})
    .render();
    };

  // updates Chart with new data once props.data is changed
  watch(
  () => props.data,
  () => {
    createChart();
  }
  );
  
  // initiates chart with example data
  onMounted(() => {
    createChart();
  });

  // functions to be called from App.vue
  defineExpose({
    collapseAll,
    expandAll,
    fitScreen
  })

  function expandAll(){
    orgChartInstance.expandAll().fit();
  }

  function collapseAll(){
    orgChartInstance.collapseAll().fit();
  }

  function fitScreen(){
    orgChartInstance.fit();
  }

  function setHighlight(node){
    
    highlightedNode.value = node;
    orgChartInstance.clearHighlighting();
    orgChartInstance.setHighlighted(node['id']).render();
  }

  function clearHighlight(){
    highlightedNode.value = null;
    orgChartInstance.clearHighlighting();
  }
</script>

  