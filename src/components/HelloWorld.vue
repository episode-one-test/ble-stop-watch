<template>
  <div class="container">
    <div class="top_title">
      <div>BLUETOOTH STOP WATCH</div>
    </div>
    <div class="top_panel">
      <div class="top_left_panel">
        <div>{{currentTime}}</div>
      </div>
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
    </div>

    <div class="middle_buttons">
      <v-btn variant="outlined" color="black" @click="onTimerStart" :disabled="timerStatus !== 'initial'">START</v-btn>
      <v-btn variant="outlined" color="black" @click="onTimerLap1" :disabled="timerStatus !== 'start'">LAP1</v-btn>
      <v-btn variant="outlined" color="black" @click="onTimerLap2" :disabled="timerStatus !== 'lap1'">LAP2</v-btn>
      <v-btn variant="outlined" color="black" @click="onTimerStop" :disabled="timerStatus !== 'start' && timerStatus !== 'lap1' && timerStatus !== 'lap2'">STOP</v-btn>
      <v-btn variant="outlined" color="black" @click="onTimerReset" :disabled="timerStatus !== 'stop'">RESET</v-btn>
    </div>

    <div class="runner_panel">
      <div class="runner_panel_left">
        <div class="runner_name">{{currentRunner.name}}</div>
        <div class="runner_times_container">
          <div class="runner_times" v-for="(runData, index) in currentRunner.data" :key="index">
            <div class="runner_times_no">
              [{{index + 1}}]
            </div>
            <div class="runner_times_info">
              <div>{{runData.time}}, lap1: {{runData.lap1}}, lap2: {{runData.lap2}}</div>
              <div>start: {{runData.startTimestamp}}</div>
              <div>finish: {{runData.goalTimestamp}}</div>
            </div>
          </div>
        </div>
      </div>
      <div class="runner_panel_right">
        <div v-for="runner in runnerList" :key="runner.number">
          <v-btn :class="{'active-runner': currentRunner === runner}" @click="onSelectRunner(runner)">{{runner.name}}</v-btn>
        </div>
        <div>
          <v-btn color="red" @click="onAddRunner"><v-icon>mdi-plus</v-icon></v-btn>
        </div>
      </div>
    </div>

    <div class="footer_buttons">
      <v-btn class="flex-grow-1" @click="onRequestDevice" v-show="bleStatus === 'disconnected'"><v-icon>mdi-bluetooth</v-icon>ON</v-btn>
      <v-btn class="flex-grow-1" @click="onDisconnect" v-show="bleStatus === 'connected'"><v-icon>mdi-bluetooth-off</v-icon>OFF</v-btn>
      <v-btn class="flex-grow-1" @click="onClearData">CLEAR</v-btn>
      <v-btn class="flex-grow-1" @click="onDownloadData">DOWNLOAD</v-btn>
    </div>
  </div>
</template>

<script>

import {ref} from "vue"
// eslint-disable-next-line no-undef
const ble = new BlueJelly();
export default {
  name: 'HelloWorld',
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

        currentRunner.value.data.push(createRunData(timer.value, lap1.value, lap2.value, start.value, goal.value))
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
      },
      onClearData() {
        initializeRunner()
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
  height: calc(100vh - 30px - 120px - 46px - 56px - 20px);
  display: flex;
  padding: 10px;
  gap: 10px;
}
.runner_panel_left {
  flex-grow: 1;
  overflow: hidden;
  border: 1px solid #333;
  border-radius: 5px;
}
.runner_name {
  background-color: #333;
  color: #fff;
  border-radius: 4px;
  margin: 5px;
  padding: 5px;
  font-weight: 500;
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
  width: 30px;
  align-self: center;
}
.runner_times_info {
  flex-grow: 1;
}
.runner_panel_right {
  width: 100px;
  overflow-y: scroll;
  overflow-x: hidden;
  border: 1px solid #333;
  border-radius: 5px;
  padding: 5px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 5px;
}
.runner_panel_right button {
  padding: 0 5px;
  text-transform: none;
  background-color: #ccc;
  color: #333;
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
</style>
