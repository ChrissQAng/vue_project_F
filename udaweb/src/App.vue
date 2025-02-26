<script setup>
import { ref, onMounted, computed, nextTick } from 'vue'
import { format } from 'date-fns'
import { Chart } from 'chart.js/auto'

const backendUrl = 'http://localhost:3000'

const data = ref([])
let charts = []

const setupmode = ref(false)
const setupchannel = ref(1) // Sensor #, not channel
const setupdata = ref([])
const isLoading = ref(false) // LA

let setupChart = null
let setupHistory = []

const destroySetupChart = () => {
  if(setupChart != null) {
    setupChart.destroy()
    setupChart = null
  }
  setupHistory = []
}

const updateM = async (channel, value) => {
  const response = await fetch(backendUrl + '/set?ch=' + channel + '&m=' + (value ? '1' : '0'))
}

const updateMI = async (channel, value) => {
  const response = await fetch(backendUrl + '/set?ch=' + channel + '&mi=' + value)
}

const updateSetupMode = async (channel, value) => {
  const response = await fetch(backendUrl + '/set?ch=' + channel + '&setup=' + value)
}

const updateFt = async () => {
  const response = await fetch(backendUrl + '/data?ch=' + setupchannel.value + ',' + (setupchannel.value + 1) + '&type=ft')
  const json = await response.json()
  if(setupmode.value) {
    setupdata.value = [ json[0]['ft'], json[1]['ft'] ]
    setupHistory.push( { time: new Date().valueOf(), front: json[0]['ft'], rear: json[1]['ft'] } )
    while(setupHistory[0].time < new Date().valueOf() - 30000) setupHistory.shift()
    setTimeout(updateFt, 500)
    updateSetupChart()
  }
  else {
    destroySetupChart()
  }
}

const updateSetupChart = () => {
  const timeoffset = setupHistory.slice(-1)[0].time - 30000
  let datasets = []
  for(let i = 0; i < 2; i++) {
    datasets.push({
      label: i == 0 ? 'Ft (F)' : 'Ft (B)',
      showLine: true,
      data: setupHistory.map(el => ({ x: el.time - timeoffset, y: i == 0 ? el.front : el.rear }))
    })
  }

  setupChart.data.datasets = datasets
  setupChart.update()
}

const startSetupMode = async (channel) => {
  setupmode.value = true
  setupchannel.value = channel
  //updateM(channel * 2 - 1, 1)
  //updateMI(channel * 2 - 1, 750)
  //updateMI(channel * 2, 750)
  updateSetupMode(channel, 1)
  updateFt()
  destroySetupChart()
  await nextTick()
  createSetupChart()
}

const endSetupMode = () => {
  setupmode.value = false
  updateSetupMode(setupchannel.value * 2 - 1, 0)
  destroySetupChart()
}

const updateData = async () => {
  const response2 = await fetch(backendUrl + '/data')
  // TODO error handling
  data.value = await response2.json()

  await nextTick()

  console.log(data.value)

  for(let i = 1; i <= data.value.length / 2; i++) {
    const ctxfront = document.getElementById('satchartfront' + i)
    const ctxrear = document.getElementById('satchartrear' + i)

    const dataindex = i * 2 - 2

    const datafront = data.value[dataindex].ychart.map((value, index) => ({ x: index * 50, y: value }))
    const datarear = data.value[dataindex + 1].ychart.map((value, index) => ({ x: index * 50, y: value }))
    
    const chartdatasetsfront = [
      {
	label: 'Ft (F)',
	data: datafront,
	fill: false
      }
    ]
    const chartdatasetsrear = [
      {
	label: 'Ft (R)',
	data: datarear,
	fill: false
      }
    ]

    console.log("Charts available: " + charts.length + ", charts needed: " + (i * 2))
    if(charts.length < i * 2) {
      charts.push(new Chart(ctxfront, {
	type: 'scatter',
	data: {
	  datasets: chartdatasetsfront
	},
	options: {
	  scales: {
	    y: {
	      ticks: {
		display: false
	      }
	    },
	    x: {
	      ticks: {
		display: false
	      }
	    }
	  },
	  elements: {
	    point: {
	      radius: 2
	    },
	    line: {
	      borderWidth: 2
	    }
	  },
	  plugins: {
	    legend: {
	      display: false
	    }
	  },
	  animation: false,
	  responsive: false
	}
      }))
      charts.push(new Chart(ctxrear, {
	type: 'scatter',
	data: {
	  datasets: chartdatasetsrear
	},
	options: {
	  scales: {
	    y: {
	      ticks: {
		display: false
	      }
	    },
	    x: {
	      ticks: {
		display: false
	      }
	    }
	  },
	  elements: {
	    point: {
	      radius: 2
	    },
	    line: {
	      borderWidth: 2
	    }
	  },
	  plugins: {
	    legend: {
	      display: false
	    }
	  },
	  animation: false,
	  responsive: false
	}
      }))
    }
    else {
      charts[i * 2 - 2].data.datasets = chartdatasetsfront
      charts[i * 2 - 2].update()
      charts[i * 2 - 1].data.datasets = chartdatasetsrear
      charts[i * 2 - 1].update()
    }
  }
}

const runMeasurement = async (channel, withFlash) => {
  console.log("Run measurement")
  isLoading.value = true // LA
  const meastype = withFlash ? "sat" : "f"
  try {
    const response1 = await fetch(backendUrl + '/set?ch=' + (channel * 2 - 1) + '&meas=' + meastype)
    console.log("Measurement finished")
    await updateData()
  } catch (error) {
    console.error("Error during measurement:", error)
  } finally {
    isLoading.value = false
  }
 
 
  // const response1 = await fetch(backendUrl + '/set?ch=' + (channel * 2 - 1) + '&meas=' + meastype)
  // console.log("Measurement finished")
  // updateData()
}

onMounted(async () => {
  var x = setTimeout('');
  for (var i = 0; i <= x; i++)
    clearTimeout(i);

  updateData()
})

const formatDate = (date) => {
  if(date)
    return format(date, 'yyyy-MM-dd HH:mm')
  return ''
}

const calcYII = (f, fm) => {
  let y = 1 - (f / fm)
  return y.toFixed(3)
}

const pairedSensorData = computed(() => {
  let ret = []
  for(let i = 0; i < data.value.length; i += 2) {
    ret.push( { sensorno: (data.value[i].channel + 1) / 2, front: data.value[i], back: data.value[i + 1] } )
  }
  return ret
})

const createSetupChart = () => {
  const ctx = document.getElementById('setupChart')
  setupChart = new Chart(ctx, {
    type: 'scatter',
    data: {
      datasets: [
	{
	  label: 'Ft (F)',
	  data: [],
	  fill: false,
	},
	{
	  label: 'Ft (R)',
	  data: [],
	  fill: false,
	}
      ]
    },
    options: {
      scales: {
        y: {
          min: 0,
	  max: 1000,
	  ticks: {
	    display: false
	  }
        },
	x: {
	  min: 0,
	  max: 30000,
	  ticks: {
	    display: false
	  }
	},
      },
      elements: {
	point: {
	  radius: 0
	},
	line: {
	  borderWidth: 2,
	}
      },
      plugins: {
	legend: {
	  display: false
	}
      },
      animation: false,
      responsive: false
    }
  })
}

</script>

<template>
<div v-if="setupmode">
  <div>
    <div @click="endSetupMode">
      <img class="icon-close" src="./assets/icons/close.png" alt="close icon">
    </div>
    <h2 class="headline-setup">Setup</h2>
<div class="data-setup">
<h2>Sensor: {{ setupchannel }}</h2>
<p>Front: {{ setupdata[0] }}</p>
<p>Rear: {{ setupdata[1] }}</p>
</div>


    <canvas id="setupChart" style="width: 20em; height: 10em; border: solid 1px black;"></canvas>
  </div>
</div>
<div v-else>

  <div v-for="sensor in pairedSensorData" :key="sensor.sensorno">
 <h2>Sensor {{ sensor.sensorno }}</h2>
 <div class="buttons">
 <button :class="isLoading? 'button-setup button-setup-disabled':'button-setup'" @click="startSetupMode(sensor.sensorno)">Setup</button>
 <button :class="isLoading? 'button-run button-run-disabled':'button-run'" @click="runMeasurement(sensor.sensorno, 0)">Measure F</button>
 <button :class="isLoading? 'button-run button-run-disabled':'button-run'" @click="runMeasurement(sensor.sensorno, 1)">Measure Yield</button>
 <video v-if="isLoading" src="./assets/icons/Animation.webm" autoplay loop muted class="loading-animation"></video>
</div>

<div class="measurement-headline">
  <h3>F Measurement <span>&#10003</span></h3>
<!-- check implementation here -->
</div>
<h4>({{ formatDate(sensor.front.ftime) }})</h4>
 <div class="grid-wrapper grid-border">

<div></div>
<p>F</p>
<p>B</p>

<div></div>
<p class="text-align">F</p>
<p class="number">{{ sensor.front.f }}</p>
<p>{{ sensor.back.f }}</p>
<span>&#10003</span>
<p class="text-align">Deviation %</p>
<p class="number">3</p>
<p>4</p>
<span>&#10003</span>
<p class="text-align">Temp °C</p>
<p class="number">3.4</p>
<p>4.2</p>
<span>&#10003</span>

</div>

<div class="measurement-headline">
<h3>Yield Measurement <span>&#10003</span></h3>
<!-- check implementation here -->
</div>
<h4>({{ formatDate(sensor.front.ytime) }})</h4>

<div class="grid-border">
<div class="grid-wrapper">

<div></div>
<p>F</p>
<p>B</p>

<div></div>
<p class="text-align">F</p>
<p>{{ sensor.front.yf }}</p>
<p>{{ sensor.back.yf }}</p>
<span>&#10003</span>

<p class="text-align">Fm</p>
<p>{{ sensor.front.yfm }}</p>
<p>{{ sensor.back.yfm }}</p>
<span>&#10003</span>
<p class="text-align">Y(II)</p>
<p>{{ calcYII(sensor.front.yf, sensor.front.yfm) }}</p>
<p>{{ calcYII(sensor.back.yf, sensor.back.yfm) }}</p>
<span>&#10003</span>

<p class="text-align">Temp °C</p>
<p>3.4</p>
<p>4.2</p>
<span>&#10003</span>
</div>

<div class="charts">
<canvas class="chart-yield" :id="'satchartfront' + sensor.sensorno" style="width:45vw;height:22.5vw;border:1px solid black;"></canvas>
<canvas class="chart-yield" :id="'satchartrear' + sensor.sensorno" style="width:45vw;height:22.5vw;border:1px solid black;"></canvas>
</div>

</div>

</div>
</div>

</template>

<style scoped>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: Arial, Helvetica, sans-serif;
}
/* sensor */
h2 {
  background-color: #ddd;
  font-size: 3rem;
}
.buttons {
  margin: 1.5625rem 0;
  display: flex;
  gap: .625rem

}
.button-setup, .button-run {
display: inline-block;
outline: 0;
cursor: pointer;
text-align: center;
border: 0;
padding: 7px 16px;
/* min-height: 54px; */height: 3.375rem;
/* min-width: 54px; */width: 30vw;
color: #ffffff;
background: blue;
border-radius: 4px;
font-weight: 500;
font-size: 18px;
box-shadow: rgba(0, 0, 0, 0.05) 0px 1px 0px 0px, rgba(0, 0, 0, 0.2) 0px -1px 0px 0px inset;
}
.button-run {
  background: #008060;
}
.button-setup-disabled {
  background: lightgray;
}
.button-run-disabled {
  background: lightgray;
}
video {
  height: 3.375rem;
}
h3 {
  padding-top: .625rem;
  font-size: 2rem;
  background-color:azure;
}
h4 {
  font-size: 1.7rem;
}
.measurement-headline {
  display: flex;
  align-items: center;
  gap: 1.25rem;
}
.grid-wrapper {
  display: grid;
  grid-template-columns: 30vw auto auto 3rem;
  gap: .625rem;

}
.grid-border {
  padding: 2%;
  margin: 1%;
  border: 1px solid black;
}
p {
  font-size: 1.5rem;
  text-align: end;
}
span {
  font-size: 1.5rem;
  text-align: start;
  color: green;
}
.charts {

  margin: 2vh 0;
  display: flex;
  /* flex-direction: row; */
  /* flex-wrap: wrap; */
  justify-content: space-around;
  /* align-items: center; */
  gap: .625rem;
  
}
.chart-yield {
  width: 45%;
}
.text-align {
  text-align: start;
}
.text-align-center{
  text-align: center;
}
/* Setup */

img {
  width: 5vw;
}

.headline-setup {
  margin: 1.5625rem 0 ;
  background-color: blue;
  color: white;
}
.icon-close {
  cursor: pointer;
}
.data-setup {
  margin:  1.25rem 0;
}
.data-setup p{
  margin: .9375rem 0;
  text-align: start;
  font-weight: 700;
}
</style>
