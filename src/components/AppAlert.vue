<template>
  <div class="app_alert__background" v-show="store.isShowAlert">
    <div class="app_alert__container">
      <div class="app_alert__message">{{store.alertMessage}}</div>
      <div class="app_alert__buttons">
        <v-btn @click="close" color="#ccc" v-show="store.alertType === 'close'">閉じる</v-btn>
        <v-btn @click="yes" color="#ccc" v-show="store.alertType === 'yes_no'">はい</v-btn>
        <v-btn @click="no" color="#ccc" v-show="store.alertType === 'yes_no'">いいえ</v-btn>
      </div>
    </div>
  </div>
</template>

<script>
import { store } from "@/store"
const result = {
  resolve: undefined,
  reject: undefined
}

export default {
  setup() {
    return {
      store,
      close() {
        store.value.isShowAlert = false
        if (result.resolve) {
          result.resolve()
        }
      },
      yes() {
        store.value.isShowAlert = false
        if (result.resolve) {
          result.resolve()
        }
      },
      no() {
        store.value.isShowAlert = false
        if (result.reject) {
          result.reject()
        }
      }
    }
  },
  show(message, alertType) {
    store.value.isShowAlert = true
    store.value.alertMessage = message
    store.value.alertType = alertType
    return new Promise((resolve, reject) => {
      result.resolve = resolve
      result.reject = reject
    })
  }
}
</script>

<style>
.app_alert__background {
  position: fixed;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  background: rgba(0, 0, 0, 0.6);

  display: flex;
  justify-content: center;
  align-items: center;
}
.app_alert__container {
  display: flex;
  flex-direction: column;
  justify-content: center;
  margin: 60px 20px;
  padding: 16px 20px;
  gap: 16px;

  width: 400px;

  background: #fff;
  border-radius: 8px;
}
.app_alert__message {
  width: 100%;

  padding: 10px 12px;
  background: #fff;
  border-radius: 4px;

  font-weight: 400;
  font-size: 14px;
  line-height: 20px;

  color: #000;
}
.app_alert__buttons {
  display: flex;
  flex-direction: row;
  justify-content: center;
  gap: 10px;
}
</style>
