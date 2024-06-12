<template>
  <div class="container">
    <div class="top_title">
      <div>BLUETOOTH STOPWATCH</div>
    </div>
    <div class="top_panel">
      <div class="top_left_panel">
        <div>{{currentTime}}</div>
      </div>
      <!--
      <div class="top_right_panel">
        <div class="top_right_sub_panel">
          <div class="top_right_sub_label">
            <div>LAP 1</div>
          </div>
          <div class="top_right_sub_time">
            <div>{{lap1Time}}</div>
          </div>
        </div>
        <div class="top_right_sub_panel">
          <div class="top_right_sub_label">
            <div>LAP 2</div>
          </div>
          <div class="top_right_sub_time">
            <div>{{lap2Time}}</div>
          </div>
        </div>
      </div>
      -->
    </div>

    <div class="middle_buttons">
      <v-btn variant="outlined" color="black" @click="onTimerStart" :disabled="timerStatus !== 'initial'">START</v-btn>
<!--      <v-btn variant="outlined" color="black" @click="onTimerLap1" :disabled="timerStatus !== 'start'">LAP1</v-btn>-->
<!--      <v-btn variant="outlined" color="black" @click="onTimerLap2" :disabled="timerStatus !== 'lap1'">LAP2</v-btn>-->
      <v-btn variant="outlined" color="black" @click="onTimerStop" :disabled="timerStatus !== 'start' && timerStatus !== 'lap1' && timerStatus !== 'lap2'">STOP</v-btn>
      <v-btn variant="outlined" color="black" @click="onTimerReset" :disabled="timerStatus !== 'stop'">RESET</v-btn>
    </div>

    <div class="runner_panel">
      <div class="runner_panel_left">
        <div class="runner_headline">
          <div class="runner_name"><input type="text" v-model="currentRunner.name" v-on:change="onChangeRunnerName"></div>
          <div class="runner_delete"><v-btn color="red" @click="onDeleteRunner">DELETE</v-btn></div>
        </div>
        <div class="runner_times_container">
          <div class="runner_times" v-for="(runData, index) in currentRunner.data" :key="index">
            <div class="runner_times_no">
              [{{currentRunner.data.length - index}}]
            </div>
            <div class="runner_times_info">
              <div>{{runData.time}}<!--<span class="runner_times_label">LAP1</span>{{runData.lap1}}<span class="runner_times_label">LAP2</span>{{runData.lap2}}--></div>
<!--              <div><span class="runner_times_label">START</span>{{runData.startTimestamp}}</div>-->
<!--              <div><span class="runner_times_label">FINISH</span>{{runData.goalTimestamp}}</div>-->
            </div>
            <div class="runner_times_delete">
              <v-btn color="red" size="x-small" @click="() => onDeleteRunnerTime(index)"><span style="font-size: 10px">DELETE</span></v-btn>
            </div>
          </div>
        </div>
      </div>
      <div class="runner_panel_right">
        <div v-for="runner in runnerList" :key="runner.number">
          <v-btn :class="{'active-runner': currentRunner === runner}" @click="onSelectRunner(runner)"><span>{{runner.name}}</span></v-btn>
        </div>
        <div>
          <v-btn color="red" @click="onAddRunner">ADD</v-btn>
        </div>
      </div>
    </div>

    <div class="footer_buttons">
      <v-btn class="flex-grow-1" @click="onRequestDevice" v-show="bleStatus === 'disconnected'">BLUETOOTH ON</v-btn>
      <v-btn class="flex-grow-1" @click="onDisconnect" v-show="bleStatus === 'connected'">BLUETOOTH OFF</v-btn>
      <v-btn class="flex-grow-1" @click="onClearData">CLEAR</v-btn>
      <v-btn class="flex-grow-1" @click="onDownloadData">DOWNLOAD</v-btn>
    </div>
  </div>
</template>

<script>

import {ref} from "vue"
import AppAlert from "@/components/AppAlert"
// eslint-disable-next-line no-undef
const ble = new BlueJelly();
export default {
  name: 'MainVue',
  setup() {
    const readData = ref('')
    const bleStatus = ref('disconnected')
    const timerStatus = ref('initial')
    const timer = ref(0)
    const lap1 = ref(0)
    const lap2 = ref(0)
    const start = ref('')
    const goal = ref('')
    const runnerList = ref([])
    const currentRunner = ref(null)
    ble.setUUID(
        "UUID1",
        "1b24e5c4-a39c-4d46-92fb-3bbcb2f34a41",
        "9d18d524-2a6e-44ce-8724-445575b23e9a");
    ble.onRead = function(data, uuid){
      if (uuid === "UUID1") {
        const decoder = new TextDecoder('utf-8')
        const decoded = decoder.decode(data)
        readData.value = decoded

        const jsonData = JSON.parse(decoded)
        if ((jsonData.sw & (1 << 0)) > 0) {
          console.log('RESET_SW1')
          resetTimer()
        }
        if ((jsonData.sw & (1 << 1)) > 0) {
          console.log('RESET_SW2')
          resetTimer()
        }
        if ((jsonData.tr & (1 << 0)) > 0) {
          console.log('START_TRG')
          startTimer()
        }
        if ((jsonData.tr & (1 << 1)) > 0) {
          console.log('MIDDLE1_TRG')
          setLap1()
        }
        if ((jsonData.tr & (1 << 2)) > 0) {
          console.log('MIDDLE2_TRG')
          setLap2()
        }
        if ((jsonData.tr & (1 << 3)) > 0) {
          console.log('GOAL_TRG')
          stopTimer()
        }
      }
    }
    ble.onScan = function(/*deviceName*/) {
      ble.connectGATT('UUID1')
    }
    ble.onConnectGATT = function(uuid) {
      if (uuid === "UUID1") {
        bleStatus.value = 'connected'
        ble.startNotify('UUID1');
      }
    }
    ble.onDisconnect = function() {
      bleStatus.value = 'disconnected'
      readData.value = ''
    }

    let beforeTime = 0;
    let intervalTimer = undefined
    function startTimer() {
      if (!intervalTimer) {
        const startTime = Date.now();
        timerStatus.value = 'start'
        const _beforeTime = beforeTime
        intervalTimer = setInterval(() => {
          const calculatedTime = _beforeTime + Date.now() - startTime;
          timer.value = calculatedTime
          beforeTime = calculatedTime
        }, 10)

        start.value = new Date().toLocaleString()
      }
    }
    function setLap1() {
      lap1.value = timer.value
      timerStatus.value = 'lap1'
    }
    function setLap2() {
      lap2.value = timer.value
      timerStatus.value = 'lap2'
    }
    function stopTimer() {
      if (intervalTimer) {
        clearInterval(intervalTimer)
        intervalTimer = undefined

        goal.value = new Date().toLocaleString()
        timerStatus.value = 'stop'

        currentRunner.value.data.unshift(createRunData(timer.value, lap1.value, lap2.value, start.value, goal.value))
        saveToLocalStorage()
      }
    }
    function resetTimer() {
      stopTimer()
      timer.value = 0
      lap1.value = 0
      lap2.value = 0
      beforeTime = 0

      start.value = ''
      goal.value = ''
      timerStatus.value = 'initial'
    }

    function createRunner(number) {
      return {
        number,
        name: `Runner${number}`,
        data: []
      }
    }
    function createRunData(time, lap1, lap2, start, goal) {
      return {
        time: (time / 1000).toFixed(2),
        lap1: lap1 ? (lap1 / 1000).toFixed(2) : '',
        lap2: lap2 ? (lap2 / 1000).toFixed(2) : '',
        startTimestamp: start,
        goalTimestamp: goal,
      }
    }
    function initializeRunner() {
      const newRunner = createRunner(1)
      runnerList.value = [newRunner]
      currentRunner.value = newRunner
      saveToLocalStorage()
    }
    function clearRunnerData() {
      runnerList.value.map(runner => {
        runner.data = []
      })
      saveToLocalStorage()
    }

    function downloadRunnerData() {
      //ダウンロードするCSVファイル名を指定する
      const filename = "download.csv";
      //CSVデータ
      const data = runnerList.value.map(runner => runner.data.reduce((value, data) => {
        if (data.lap1) {
          value.push(data.lap1)
          if (data.lap2) {
            value.push(data.lap2)
          }
        }
        value.push(data.time)
        return value
      }, [runner.name]).join(',')).join('\n')
      //BOMを付与する（Excelでの文字化け対策）
      const bom = new Uint8Array([0xef, 0xbb, 0xbf]);
      //Blobでデータを作成する
      const blob = new Blob([bom, data], { type: "text/csv" });

      if (navigator.share) {
        const file = new File([blob], filename, {
          type: "text/csv",
        })

        navigator.share({
          files: [file],
        }).then(() => {
          console.log("共有成功.");
        }).catch((error) => {
          console.log(error);
        });
      } else {
        //BlobからオブジェクトURLを作成する
        const url = (window.URL || window.webkitURL).createObjectURL(blob);
        //ダウンロード用にリンクを作成する
        const download = document.createElement("a");
        //リンク先に上記で生成したURLを指定する
        download.href = url;
        //download属性にファイル名を指定する
        download.download = filename;
        // 作成したリンクをクリックしてダウンロードを実行する
        download.click();
        // createObjectURLで作成したオブジェクトURLを開放する
        (window.URL || window.webkitURL).revokeObjectURL(url);
      }
    }

    function loadFromLocalStorage() {
      const runner = JSON.parse(localStorage.getItem('runner'))
      if (runner && runner.length > 0) {
        runnerList.value = runner
        currentRunner.value = runner[0]
        return true
      }
      return false
    }
    function saveToLocalStorage() {
      localStorage.setItem('runner', JSON.stringify(runnerList.value))
    }

    if (!loadFromLocalStorage()) {
      initializeRunner()
    }

    return {
      readData,
      bleStatus,
      timerStatus,
      timer,
      lap1,
      lap2,
      start,
      goal,
      runnerList,
      currentRunner,
      onRequestDevice() {
        ble.requestDevice('UUID1')
      },
      onConnect() {
        ble.connectGATT('UUID1')
      },
      onRead() {
        ble.read("UUID1")
      },
      onDisconnect() {
        ble.disconnect()
      },
      onTimerStart() {
        startTimer()
      },
      onTimerLap1() {
        setLap1()
      },
      onTimerLap2() {
        setLap2()
      },
      onTimerStop() {
        stopTimer()
      },
      onTimerReset() {
        resetTimer()
      },
      onAddRunner() {
        runnerList.value.push(createRunner(runnerList.value.length + 1))
        saveToLocalStorage()
      },
      onSelectRunner(runner) {
        currentRunner.value = runner
        if (timerStatus.value === 'stop') {
          resetTimer()
        }
      },
      async onDeleteRunner() {
        try {
          await AppAlert.show('削除します。よろしいですか？', 'yes_no')
        } catch(e) {
          return
        }
        const index = runnerList.value.findIndex(item => item === currentRunner.value)
        if (index < 0) return

        runnerList.value = runnerList.value.filter(item => item !== currentRunner.value)
        if (runnerList.value.length === 0) {
          initializeRunner()
        } else if (runnerList.value.length === index) {
          currentRunner.value = runnerList.value[index - 1]
          saveToLocalStorage()
        } else {
          currentRunner.value = runnerList.value[index]
          saveToLocalStorage()
        }
      },
      onChangeRunnerName() {
        saveToLocalStorage()
      },
      onDeleteRunnerTime(index) {
        currentRunner.value.data.splice(index, 1)
        saveToLocalStorage()
      },
      async onClearData() {
        try {
          await AppAlert.show('クリアします。よろしいですか？', 'yes_no')
          clearRunnerData()
          resetTimer()
        } catch(e) {
          // Do nothing.
        }
      },
      onDownloadData() {
        downloadRunnerData()
      }
    }
  },
  computed: {
    currentTime: function () {
      return (this.timer / 1000).toFixed(2)
    },
    lap1Time: function () {
      return (this.lap1 / 1000).toFixed(2)
    },
    lap2Time: function () {
      return (this.lap2 / 1000).toFixed(2)
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}

.container {
  width: 100vw;
  height: 100vh;
  padding-bottom: 30px;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}
.top_title {
  background-color: #333;
  border-radius: 10px;
  color: #fff;
  margin: 10px;
  height: 30px;
  font-weight: 500;
  font-size: 20px;
  display: flex;
  justify-content: center;
  align-items: center;
}
.top_panel {
  display: flex;
  width: 100%;
  height: 120px;
  justify-content: space-between;
  gap: 10px;
  padding: 10px;
}
.top_left_panel {
  background-color: #333;
  border-radius: 5px;
  color: #fff;
  font-weight: 500;
  font-size: 50px;
  display: flex;
  justify-content: center;
  align-items: center;
  flex-grow: 2;
  height: 100px;
}
.top_right_panel {
  display: flex;
  flex-direction: column;
  flex-grow: 1;
  gap: 10px;
}
.top_right_sub_panel {
  display: flex;
  flex-direction: row;
  flex-grow: 1;
}
.top_right_sub_label {
  font-weight: 500;
  font-size: 20px;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 0 5px;
}
.top_right_sub_time {
  background-color: #333;
  border-radius: 5px;
  color: #fff;
  font-weight: 500;
  font-size: 30px;
  display: flex;
  justify-content: center;
  align-items: center;
  flex-grow: 1;
}
.middle_buttons {
  display: flex;
  width: 100%;
  height: 46px;
  justify-content: space-between;
  gap: 2px;
  padding: 5px 10px;
}
.middle_buttons button {
  padding: 5px 0;
  flex-grow: 1;
}
.runner_panel {
  height: calc(100vh - 30px - 120px - 46px - 56px - 20px - 30px);
  max-height: calc(100vh - 30px - 120px - 46px - 56px - 20px - 30px);
  min-height: 100px;
  display: flex;
  padding: 10px;
  gap: 10px;
}
.runner_panel_left {
  flex-grow: 1;
  overflow: hidden;
  border: 1px solid #333;
  border-radius: 5px;
  min-width: 60%;
}
.runner_headline {
  display: flex;
  flex-direction: row;
}
.runner_name {
  background-color: #333;
  color: #fff;
  border-radius: 4px;
  margin: 5px;
  padding: 5px;
  font-weight: 500;
  flex-grow: 1;
}
.runner_name input {
  color: #fff;
  width: 100%;
}
.runner_name input:focus {
  outline: none;
}
.runner_delete {
  padding: 5px;
}
.runner_times_container {
  overflow-y: scroll;
  overflow-x: hidden;
  height: calc(100% - 34px);
}
.runner_times {
  display: flex;
  flex-direction: row;
  padding: 5px;
}
.runner_times_no {
  width: 25px;
  align-self: center;
  font-weight: 500;
}
.runner_times_info {
  flex-grow: 1;
  align-self: center;
  font-size: 14px;
}
.runner_times_delete {
  flex-grow: 0;
  align-self: center;
}
.runner_times_label {
  width: 50px;
  background-color: #ccc;
  color: #333;
  border-radius: 10px;
  display: inline-block;
  text-align: center;
  border: 1px solid #fff;
  margin: 0 3px;
  font-weight: 300;
  font-size: 12px;
  line-height: 16px;
}
.runner_panel_right {
  /*min-width: 100px;*/
  /*max-width: 50%;*/
  max-width: 600px;
  flex-grow: 1;
  overflow-y: scroll;
  overflow-x: hidden;
  max-height: 100%;
  border: 1px solid #333;
  border-radius: 5px;
  padding: 5px;
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(100px, 1fr));
  justify-items: center;
  align-content: start;
  gap: 5px;
}
.runner_panel_right button {
  padding: 0 5px;
  text-transform: none;
  background-color: #ccc;
  color: #333;
  width: 100px;
}
.runner_panel_right button span {
  width: 100px;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
  display: block;
}
.runner_panel_right .active-runner {
  background-color: #00c;
  color: #fff;
}
.footer_buttons {
  display: flex;
  width: 100%;
  height: 56px;
  justify-content: space-between;
  gap: 10px;
  padding: 10px;
}
.footer_buttons button {
  background-color: #ccc;
  color: #333;
}
::-webkit-scrollbar {
  width: 5px;
  background-color: #fff;
}
::-webkit-scrollbar-track {
  background-color: #fff;
}
::-webkit-scrollbar-thumb{
  background-color: #aaa;
  border-radius: 5px;
}
</style>
