<template>
  <div class="sponsors-wrap">
    <h1 class="txt">
      截止至{{ onMountedTime }}，已收到：{{ receiveMoney / 100 }}元赞助~
    </h1>
    <div class="pay-list">
      <div
        v-for="(item, index) in payList"
        :key="index"
        class="item"
      >
        <div class="time">发起时间：{{ item.created_at }}，</div>
        <div class="user">
          <template v-if="item.user">
            <img
              :src="item.user.avatar"
              class="avatar"
              alt=""
            />
            <span class="username">{{ item.user.username }}</span>
          </template>
          <span v-else>游客</span>，
        </div>

        <div class="account">支付宝账号：{{ item.buyer_logon_id }}，</div>
        <div class="gift">
          赞助了：{{ item.subject }}（{{ item.total_amount }}元），
        </div>
        <div class="status">
          状态：{{
            item.trade_status === PayStatusEnum.WAIT_BUYER_PAY
              ? '支付中'
              : '已支付'
          }}，
        </div>
        <div class="time">支付时间：{{ item.send_pay_date || '-' }}</div>
      </div>
    </div>
    <h2>开始赞助（支付宝）</h2>
    <div class="gift-list">
      <div
        v-for="(item, index) in sponsorsGoodsList"
        :key="index"
        class="item"
        @click="startPay(item)"
      >
        {{ item.name }}（{{ item.price }}元）
      </div>
    </div>
    <div class="qrcode-wrap">
      <img
        v-if="aliPayBase64 !== ''"
        class="qrcode"
        :src="aliPayBase64"
        alt=""
      />
      <template v-if="currentPayStatus !== PayStatusEnum.error">
        <div class="mask">
          <div class="txt">
            {{
              currentPayStatus === PayStatusEnum.TRADE_SUCCESS
                ? '支付成功'
                : '等待支付'
            }}
          </div>
        </div>
      </template>
    </div>
    <div v-if="aliPayBase64 !== ''">
      <div class="bottom">
        <div class="sao">打开支付宝扫一扫</div>
        <div class="expr">有效期5分钟（{{ formatDownTime(downTime) }}）</div>
      </div>
    </div>

    <h3 v-if="currentPayStatus === PayStatusEnum.WAIT_BUYER_PAY">
      ps：支付宝标题显示：东圃牛杂档，是正常的~
    </h3>

    <div
      v-if="payOk"
      class="bottom"
    >
      <h2>感谢您的赞助~</h2>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { hrefToTarget, isMobile } from 'billd-utils';
import QRCode from 'qrcode';
import { onMounted, onUnmounted, ref } from 'vue';

import { fetchGoodsList } from '@/api/goods';
import { fetchAliPay, fetchAliPayStatus, fetchOrderList } from '@/api/order';
import { GoodsTypeEnum, IGoods, IOrder, PayStatusEnum } from '@/interface';

const payOk = ref(false);
const onMountedTime = ref('');
const aliPayBase64 = ref('');
const payStatusTimer = ref();
const downTimer = ref();
const receiveMoney = ref(0);
const downTime = ref();
const downTimeEnd = ref();

const payList = ref<IOrder[]>([]);

const currentPayStatus = ref(PayStatusEnum.error);

const sponsorsGoodsList = ref<IGoods[]>([]);

onUnmounted(() => {
  clearInterval(payStatusTimer.value);
  clearInterval(downTimer.value);
});

onMounted(() => {
  onMountedTime.value = new Date().toLocaleString();
  getPayList();
  getGoodsList();
});

function formatDownTime(startTime: number) {
  const time2 = downTimeEnd.value - startTime;
  const ms = 1;
  const second = ms * 1000;
  const minute = second * 60;
  const hour = minute * 60;
  const day = hour * 24;
  if (time2 > day) {
    const res = (time2 / day).toFixed(4).split('.');
    return `${res[0]}天${Math.ceil(Number(`0.${res[1]}`) * 24)}时`;
  } else if (time2 > hour) {
    const res = (time2 / hour).toFixed(4).split('.');
    return `${res[0]}时${Math.ceil(Number(`0.${res[1]}`) * 60)}分`;
  } else if (time2 > minute) {
    const res = (time2 / minute).toFixed(4).split('.');
    return `${res[0]}分${Math.ceil(Number(`0.${res[1]}`) * 60)}秒`;
  } else {
    const res = (time2 / second).toFixed(4).split('.');
    return `${res[0]}秒`;
  }
}

async function generateQR(text) {
  let base64 = '';
  try {
    base64 = await QRCode.toDataURL(text, {
      margin: 1,
    });
  } catch (err) {
    console.error('生成二维码失败！', err);
  }
  return base64;
}

function handleDownTime() {
  clearInterval(downTimer.value);
  downTimeEnd.value = +new Date() + 1000 * 60 * 5;
  downTime.value = +new Date();
  downTimer.value = setInterval(() => {
    downTime.value = +new Date();
  }, 1000);
}

async function getGoodsList() {
  const res = await fetchGoodsList({
    type: GoodsTypeEnum.sponsors,
    orderName: 'created_at',
    orderBy: 'desc',
  });
  if (res.code === 200) {
    sponsorsGoodsList.value = res.data.rows;
  }
}

async function getPayList() {
  try {
    const res = await fetchOrderList({
      trade_status: PayStatusEnum.TRADE_SUCCESS,
    });
    if (res.code === 200) {
      payList.value = res.data.rows;
      receiveMoney.value = payList.value.reduce(
        (pre, item) => pre + Number(item.total_amount) * 100,
        0
      );
    }
  } catch (error) {
    console.log(error);
  }
}
async function startPay(item: IGoods) {
  currentPayStatus.value = PayStatusEnum.error;
  payOk.value = false;
  clearInterval(payStatusTimer.value);
  clearInterval(downTimer.value);
  try {
    const res = await fetchAliPay({
      total_amount: item.price!,
      subject: item.name!,
      body: item.name!,
    });
    if (res.code === 200) {
      if (isMobile()) {
        hrefToTarget(res.data.qr_code);
        return;
      }
      const base64 = await generateQR(res.data.qr_code);
      aliPayBase64.value = base64;
      getPayStatus(res.data.out_trade_no);
      handleDownTime();
    }
  } catch (error) {
    console.log(error);
  }
}

function getPayStatus(outTradeNo: string) {
  clearInterval(payStatusTimer.value);
  payStatusTimer.value = setInterval(async () => {
    try {
      const res = await fetchAliPayStatus({
        out_trade_no: outTradeNo,
      });
      if (res.data.tradeStatus === PayStatusEnum.WAIT_BUYER_PAY) {
        currentPayStatus.value = PayStatusEnum.WAIT_BUYER_PAY;
        console.log('等待支付');
      }
      if (res.data.tradeStatus === PayStatusEnum.TRADE_SUCCESS) {
        currentPayStatus.value = PayStatusEnum.TRADE_SUCCESS;
        clearInterval(downTimer.value);
        clearInterval(payStatusTimer.value);
        console.log('支付成功！');
        payOk.value = true;
        getPayList();
      }
    } catch (error) {
      console.log(error);
    }
  }, 1000);
}
</script>

<style lang="scss" scoped>
.sponsors-wrap {
  text-align: center;
  .pay-list {
    display: flex;
    overflow: scroll;
    align-items: center;
    flex-direction: column;
    box-sizing: border-box;
    margin-bottom: 20px;
    padding: 10px;
    width: 100%;
    height: 200px;
    background-color: papayawhip;
    .item {
      display: inline-flex;
      flex-wrap: wrap;
      justify-content: center;
      margin-bottom: 4px;
      width: 100%;
      text-align: left;

      .user {
        width: 120px;
        padding: 0 10px;
        display: flex;
        align-items: center;
        justify-content: center;
        .avatar {
          width: 30px;
          height: 30px;
          border-radius: 50%;
        }
        .username {
          @extend %singleEllipsis;
        }
      }
      .account {
        width: 250px;
      }
      .gift {
        width: 260px;
      }
      .status {
        width: 120px;
        text-align: left;
      }
      .time {
        width: 280px;
      }
    }
  }
  .gift-list {
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    justify-content: center;
    .item {
      margin: 5px;
      padding: 5px 10px;
      border-radius: 4px;
      background-color: skyblue;
      cursor: pointer;
    }
  }
  .qrcode-wrap {
    position: relative;
    display: flex;
    align-items: center;
    justify-content: center;
    box-sizing: border-box;
    margin: 20px auto 0;
    width: 140px;
    height: 140px;

    .mask {
      position: absolute !important;
      display: flex;
      align-items: center;
      justify-content: center;

      @extend %maskBg;
      .txt {
        color: white;
        font-weight: bold;
      }
    }
  }
  .bottom {
    margin-top: 2px;
    width: 100%;
    text-align: center;
    font-size: 14px;
  }
}
</style>
