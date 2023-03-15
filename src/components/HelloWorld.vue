<template>
  <div class="hello">
    <h1>{{ msg }}</h1>
    <div>
      <div>
        <button @click="onRequestDevice">接続</button>
        <!--<button @click="onConnect">Connect</button>-->
        <!--<button @click="onRead">Read</button>-->
        <button @click="onDisconnect">切断</button>
      </div>
      <div>{{bleStatus}}</div>
      <div>{{readData}}</div>
      <div>
        <div class="timer">time: {{currentTime}}</div>
        <div class="timer">lap1: {{lap1Time}}</div>
        <div class="timer">lap2: {{lap2Time}}</div>
        <div>
          <button @click="onTimerStart" :disabled="timerStatus !== 'initial'">スタート</button>
          <button @click="onTimerLap1" :disabled="timerStatus !== 'start'">ラップ1</button>
          <button @click="onTimerLap2" :disabled="timerStatus !== 'lap1'">ラップ2</button>
          <button @click="onTimerStop" :disabled="timerStatus !== 'start' && timerStatus !== 'lap1' && timerStatus !== 'lap2'">ストップ</button>
          <button @click="onTimerReset" :disabled="timerStatus !== 'stop'">リセット</button>
        </div>
      </div>
      <div>
        <div>{{currentRunner.name}}</div>
        <div v-for="(runData, index) in currentRunner.data" :key="index">
          [{{index + 1}}]<br>
          time: {{runData.time}}, lap1: {{runData.lap1}}, lap2: {{runData.lap2}}<br>
          {{runData.startTimestamp}} - {{runData.goalTimestamp}}
        </div>
      </div>
      <div>
        <div v-for="runner in runnerList" :key="runner.number">
          <div @click="onSelectRunner(runner)">{{runner.name}}</div>
        </div>
        <div>
          <button @click="onAddRunner">追加</button>
        </div>
      </div>
    </div>
    <div>
      <button @click="onClearData">データクリア</button>
      <button @click="onDownloadData">出力</button>
      <a href="#" @click="onDownloadData2($event)">ダウンロード</a>
    </div>
  </div>
</template>

<script>

import {ref} from "vue"
// eslint-disable-next-line no-undef
const ble = new BlueJelly();
export default {
  name: 'HelloWorld',
  props: {
    msg: String
  },
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
        "4fafc201-1fb5-459e-8fcc-c5c9c331914b",
        "beb5483e-36e1-4688-b7f5-ea07361b26a8");
    ble.onRead = function(data, uuid){
      if (uuid === "UUID1") {
        const decoder = new TextDecoder('utf-8')
        readData.value = decoder.decode(data)
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
    }

    initializeRunner()

    function downloadRunnerData(/*event*/) {
      //ダウンロードするCSVファイル名を指定する
      const filename = "download.csv";
      //CSVデータ
      const data = "テスト, テスト, テスト\nテスト, テスト, テスト";
      //BOMを付与する（Excelでの文字化け対策）
      const bom = new Uint8Array([0xef, 0xbb, 0xbf]);
      //Blobでデータを作成する
      const blob = new Blob([bom, data], { type: "text/csv" });

      let reader = new FileReader();
      reader.readAsDataURL(blob); // blob を base64 へ変換し onload を呼び出します

      reader.onload = function() {
        const download = document.createElement("a");
        download.href = reader.result; // data url
        download.download = filename;
        download.click();
      };
      // //BlobからオブジェクトURLを作成する
      // const url = (window.URL || window.webkitURL).createObjectURL(blob);
      // //ダウンロード用にリンクを作成する
      // const download = event ? event.currentTarget : document.createElement("a");
      // //リンク先に上記で生成したURLを指定する
      // download.href = url;
      // //download属性にファイル名を指定する
      // download.download = filename;
      // download.target = '_blank'
      //作成したリンクをクリックしてダウンロードを実行する
      //download.click();
      //createObjectURLで作成したオブジェクトURLを開放する
      //(window.URL || window.webkitURL).revokeObjectURL(url);
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
      },
      onSelectRunner(runner) {
        currentRunner.value = runner
      },
      onClearData() {
        initializeRunner()
      },
      onDownloadData() {
        downloadRunnerData()
      },
      onDownloadData2($event) {
        downloadRunnerData($event)
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
</style>
