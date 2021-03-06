<template>
  <div class="tutorial">
    <div class="animation" v-show="tpay_method == 'felica'">
      <div class="handbox">
        <img class="hand" :class="{pause: isPause}" src="~/assets/images/felica_hand.svg" alt>
      </div>
      <img class="reader" src="~/assets/images/felica.png" alt>
    </div>
    <qriously :value="testURL" :size="200" v-show="tpay_method == 'qr'"/>
    <div class="warning">
      <p>{{ announceText }}</p>
      <el-button @click="reSelect" type="text">別の支払方法を選択する</el-button>
      <el-button @click="changeMethod" type="text">{{ changeMethodText }}</el-button>
    </div>

    <sweet-modal
      ref="modal"
      width="80%"
      overlay-theme="dark"
      :blocking="true"
      :hide-close-button="true"
      icon="success"
    >
      <div class="main">
        <p>決済がタイムアウトしました</p>
        <el-button type="primary" @click="preparePayment();" round>もう一度</el-button>
      </div>
    </sweet-modal>
  </div>
</template>

<script>
import { mapActions, mapState } from "vuex";

export default {
  data() {
    return {
      isPause: false,
      tpay_method: "felica",
      testURL: "http://www.google.com",
      reader_timeout: 0
    };
  },
  methods: {
    changeMethod() {
      if (this.tpay_method == "felica") {
        this.tpay_method = "qr";
        this.forceQuitReader();
      } else {
        this.tpay_method = "felica";
        this.preparePayment();
      }
    },
    reSelect() {
      this.$emit("reSelect");
    },
    /*
     ** Start Card Reader
     */
    async preparePayment() {
      this.reader_timeout = 10;
      await this.startCardReader(); // フラグを立てる
      await this.showMessage(["Touch Felica", "Card"]);

      while (this.isReading && this.reader_timeout > -1) {
        const response = await this.execCardReader();

        this.reader_timeout--;
        if (response === true) {
          // IDを取得後、決済開始
          this.connectApi();
          break;
        } else if (response == false) {
          this.$ons.notification.alert("不明なエラーが発生しました。");
        } else if (this.reader_timeout <= -1) {
          // exit timeout
          this.emissionLED("error");
          this.isPause = true;
          await this.showMessage(["Timeout Reader", ""]);
          await this.stopCardReader();

          this.$ons.notification.confirm({
            title: "エラー",
            message: "リーダーがタイムアウトしました",
            cancelable: true,
            buttonLabel: ["キャンセル", "再試行"],
            callback: async index => {
              if (index == 1) {
                this.isPause = false;
                this.preparePayment();
              }
            }
          });
        }
      }
    },

    /**
     * API Tokenなどの発行
     */
    async connectApi() {
      this.isPause = true;
      this.loading = this.$loading({
        text: "Loading",
        lock: false
      });
      if (await this.getApiToken()) {
        const response = await this.checkoutWithFelica({
          amount: this.totalPrice,
          idm: this.idm
        });
        if (response == true) {
          this.recognization();
        } else {
          this.emissionLED("error");
          let message = "T-Payサーバーとの通信の際に不明なエラーが発生しました";
          switch (response) {
            case "Insufficient funds":
              await this.showMessage([
                "Balance is under",
                "the payment price."
              ]);
              message = "残高不足です。チャージしてください";
              break;
            case "User auth failed":
              await this.showMessage(["This card is not", "registerd."]);
              message = "このカードは登録されていません";
              break;
            case "Merchant not in auth user":
              message =
                "お使いのアカウントはこの店舗で決済を行うことができません";
              break;
          }
          this.quitReader();
          this.isReading = false;
          this.loading.close();
          this.$ons.notification.alert(message);
        }
      }
    },

    async recognization() {
      if (
        await this.purchaseCreate({
          id: this.method.id,
          uuid: this.uuid // 決済番号
        })
      ) {
        this.quitReader();
        this.loading.close();
        this.$emit("pushSuccess");
      } else {
        this.loading.close();
        this.$ons.notification.alert("決済エラーが発生しました");
        this.quitReader();
      }
    },
    ...mapActions("pos/purchase", ["purchaseCreate"]),
    ...mapActions("t-pay", [
      "getApiToken",
      "getMerchantID",
      "checkoutWithFelica"
    ]),
    ...mapActions("t-pay/card-reader", [
      "startCardReader",
      "stopCardReader",
      "execCardReader",
      "showMessage",
      "emissionLED",
      "quitReader",
      "forceQuitReader"
    ])
  },
  computed: {
    changeMethodText() {
      if (this.tpay_method == "felica") return "QRコード認証を使う";
      else return "Felica認証を使う";
    },
    announceText() {
      if (this.tpay_method == "felica")
        return "リーダーに、Felicaをかざしてください";
      else return "スマートフォンでQRを読み取って下さい";
    },

    method() {
      return this.payment_method.filter((_method, index) => {
        if (_method.uuid == "2ADEA824-0027-41B5-B243-10F2D24FDD4B")
          return _method;
      })[0];
    },

    totalPrice() {
      let total = 0;
      this.cart.forEach(product => {
        total += product.price * product.quantity;
      });
      return total;
    },
    ...mapState("pos/payment-method", ["payment_method"]),
    ...mapState("pos/purchase", ["cart"]),
    ...mapState("t-pay", ["uuid"]),
    ...mapState("t-pay/card-reader", ["displayText", "idm", "isReading"])
  },
  mounted() {
    this.preparePayment();
  }
};
</script>


<style lang="scss" scoped>
.tutorial {
  @include pc {
    width: 100%;
  }
  @include tab {
    height: 80%;
  }
  display: inline-block;
  text-align: center;
}

.animation {
  text-align: center;
  .handbox {
    .hand {
      max-width: 50vh;
      margin-left: 25vh;
      animation-name: felicaHandAnimation;
      animation-duration: 1.5s;
      animation-direction: alternate;
      animation-iteration-count: infinite;
      &.pause {
        animation-play-state: paused;
      }
    }
  }
  .reader {
    max-width: 45vh;
    margin-right: 25vh;
    margin-top: -200px;
  }
}
.warning {
  text-align: center;
  margin: 2vh 50px;
  font-size: 30px;
}

@keyframes felicaHandAnimation {
  0% {
    transform: rotate(0deg);
  }
  10% {
    transform: rotate(0deg);
  }
  90% {
    transform: rotate(-40deg);
  }
  100% {
    transform: rotate(-40deg);
  }
}
</style>

<style lang="scss">
.el-loading-mask.is-fullscreen {
  z-index: 30000 !important;
}
</style>