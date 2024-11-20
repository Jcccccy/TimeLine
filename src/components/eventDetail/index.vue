<template>
  <div class="EventDetail">
    <div class="head-box">
      <div class="closeBtn" @click="closeDetail">
        <i class="el-icon-close" style="font-size: 24px; color: #fff"></i>
      </div>
      <div class="closeWords">{{ totalSelectedEventsNum }} selected events</div>
    </div>
    <div class="card-box" v-for="e in event" :key="e.eventId">
      <div class="event-number">事件ID: {{ e.eventId }}</div>
      <div class="line"></div>
      <div class="bottom-box">
        <div class="title-box">
          <div class="sub-title">事件标题</div>
          <div class="word">{{ e.title }}</div>
        </div>
        <div class="info-box">
          <div class="date-box">
            <div class="sub-title">发生时间</div>
            <div class="word">{{ formattedDate(e) }}</div>
          </div>
          <div class="location-box">
            <div class="sub-title">发生位置</div>
            <div class="word">{{ e.location }}</div>
          </div>
        </div>
        <div class="content-box">
          <div class="sub-title">事件内容</div>
          <div class="word-content">{{ e.summary }}</div>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
import * as d3 from "d3";
export default {
  name: "EventDetail",
  props: {
    event: {
      type: Array,
      required: false,
      default: () => null,
    },
  },
  data() {
    return {};
  },
  mounted() {},
  computed: {
    totalSelectedEventsNum() {
      return this.event === null ? 0 : this.event.length;
    },
  },
  methods: {
    formattedDate(event) {
      if (!event || !event.date) return "";
      const format = d3.timeFormat("%d %b %Y");
      return format(new Date(event.date));
    },
    //关闭详情card
    closeDetail() {
      this.$emit("close");
    },
  },
};
</script>
<style lang="scss" scoped>
.EventDetail {
  height: calc(100% - 10px);
  overflow: auto;
  .head-box {
    display: flex;
    background-color: black;
    padding: 10px;
    align-items: center;
    justify-content: space-between;
    .closeBtn {
      cursor: pointer;
    }
    .closeWords {
      color: #fff;
    }
  }
  .card-box {
    margin: 3px 3px 3px 3px;
    background-color: #e2e2e2;
    padding: 10px 20px 30px 20px;
    opacity: 0.8;
    .event-number {
      display: flex;
      flex-direction: row-reverse;
      padding-bottom: 10px;
      font-weight: bold;
    }
    .line {
      border-bottom: 1px solid black;
      margin-bottom: 20px;
    }
    .bottom-box {
      display: flex;
      flex-direction: column;
      gap: 30px;
      .info-box {
        display: flex;
        align-items: center;
        gap: 80px;
        .date-box {
          width: 120px;
        }
      }
    }
    .sub-title {
      color: #7b7b7b;
      font-weight: bold;
      margin-bottom: 5px;
    }
    .word {
      font-size: 15px;
      color: #333;
      line-height: 1.6; /* 调整行高，增加行间距 */
      letter-spacing: 0.5px; /* 增加字间距 */
      text-align: justify; /* 两端对齐，文本更加整齐 */
    }
    .word-content {
      font-size: 15px; /* 设置适中的字体大小 */
      line-height: 1.6; /* 调整行高，增加行间距 */
      letter-spacing: 0.5px; /* 增加字间距 */
      text-align: justify; /* 两端对齐，文本更加整齐 */
      color: #333; /* 设置更柔和的字体颜色 */
      margin-top: 10px; /* 增加顶部间距，与标题分隔 */
    }
  }
}
</style>
