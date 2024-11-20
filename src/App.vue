<template>
  <div class="timeMap">
    <!-- 选择事件类型 -->
    <div class="eventType-selector">
      <label>选择事件类型:</label>
      <el-checkbox-group
        v-model="selectedEventTypes"
        @change="changeEventType"
        class="event-group"
      >
        <el-checkbox v-for="type in eventTypes" :key="type" :label="type">{{
          type
        }}</el-checkbox>
      </el-checkbox-group>
    </div>

    <!-- 折叠按钮 -->
    <div :class="['fold-box', { hidden: !isTimelineVisible }]">
      <el-button @click="toggleTimeline" class="fold-btn">
        <i
          :class="
            isTimelineVisible ? 'el-icon-caret-bottom' : 'el-icon-arrow-up'
          "
          style="font-size: 24px; color: #fff"
        ></i>
      </el-button>
    </div>

    <!-- 时间范围显示 -->
    <div :class="['time-range-display', { hidden: !isTimelineVisible }]">
      <span>Seeing events that occurred between</span>
      <div class="date-text">{{ formattedTimerange }}</div>
    </div>

    <!-- 事件详情 -->
    <div class="event-detail-box" v-if="isEventDetailVisible">
      <EventDetail :event="selectedEvent" @close="closeEventDetail" />
    </div>

    <!--向后移动时间轴-->
    <div
      class="moveToLeft"
      @click="onMoveTime('backwards')"
      v-if="isTimelineVisible"
    >
      <i class="el-icon-caret-left" style="font-size: 42px; color: black"></i>
    </div>

    <!--向前移动时间轴-->
    <div
      class="moveToRight"
      @click="onMoveTime('forward')"
      v-if="isTimelineVisible"
    >
      <i class="el-icon-caret-right" style="font-size: 42px; color: black"></i>
    </div>

    <!-- 时间线 -->
    <div class="timeline" :class="{ hidden: !isTimelineVisible }">
      <svg ref="timelineSvg" :width="svgWidth" height="300"></svg>
      <div class="time-range">
        <div
          v-for="option in timeRangeOptions"
          :key="option.value"
          :class="[
            'time-range-option',
            { active: selectedRange === option.value },
          ]"
          @click="changeRange(option.value)"
        >
          {{ option.label }}
        </div>
      </div>
    </div>
  </div>
</template>
<script>
import * as d3 from "d3";
import EventDetail from "./components/eventDetail";
export default {
  name: "App",
  components: { EventDetail },
  data() {
    return {
      isTimelineVisible: true,
      selectedRange: 1, // 默认选择1个月
      timeRangeOptions: [
        { value: 1, label: "1 month" },
        { value: 3, label: "3 months" },
        { value: 6, label: "6 months" },
      ],
      svgWidth: window.innerWidth - 100, // SVG宽度减去预留空间
      eventTypes: ["Journalist", "Civilian", "Medic", "Legal Observer"], // 示例事件类型
      selectedEventTypes: ["Journalist", "Civilian", "Medic", "Legal Observer"],
      startDate: new Date(), // 时间轴起始日期
      timerange: [new Date(2024, 8, 11), new Date(2024, 9, 11)], // 初始时间范围
      rangeLimits: [new Date(2023, 0, 1), new Date(2025, 11, 31)], // 时间范围限制
      height: 300,
      margin: {
        top: 20,
        right: 20,
        bottom: 20,
        left: 100,
      },
      transitionDuration: 300,
      dragPos0: null,
      scaleX: null,
      scaleY: null,
      events: [],
      dragScaleFactor: 0.00004,
      lastTimeShift: 0,
      timeIntervals: {
        1: d3.timeDay.every(2), // 1个月，2天为间隔
        3: d3.timeWeek.every(1), // 3个月，7天为间隔
        6: d3.timeMonth.every(1), // 6个月，1个月为间隔
      },
      selectedEvent: [], //保存当前选中的事件
      isEventDetailVisible: false, // 控制显示与否
    };
  },
  mounted() {
    this.handleEvents();
    this.makeScaleX();
    this.makeScaleY();
    this.drawTimeline();
    window.addEventListener("resize", this.updateSvgWidth);
  },
  beforeDestroy() {
    window.removeEventListener("resize", this.updateSvgWidth);
  },
  computed: {
    // 动态选择当前的 ticks 间隔
    selectedInterval() {
      return this.timeIntervals[this.selectedRange];
    },
    filteredEvents() {
      return this.events.filter((event) =>
        this.selectedEventTypes.includes(event.type)
      );
    },
    //显示时间范围
    formattedTimerange() {
      const format = d3.timeFormat("%d %b %Y");
      return `${format(this.timerange[0])} - ${format(this.timerange[1])}`;
    },
  },
  methods: {
    //隐藏详情框
    closeEventDetail() {
      this.isEventDetailVisible = false;
    },
    //控制时间轴的折叠
    toggleTimeline() {
      this.isTimelineVisible = !this.isTimelineVisible;
    },

    //更新选中的时间跨度
    changeRange(value) {
      this.selectedRange = value;
      this.updateTimeline();
      this.drawHighlightCircle();
    },

    /**
     * 当界面宽度变化时，更新svg宽度
     */
    updateSvgWidth() {
      this.svgWidth = window.innerWidth - 100;
      this.updateTimeline();
    },

    //改变显示事件类型
    changeEventType() {
      d3.select(this.$refs.timelineSvg).selectAll("*").remove(); // 清除旧内容
      this.selectedEvent = [];
      this.drawTimeline();
    },

    /**
     * 删除原来的svg，构建绘制
     */
    updateTimeline() {
      d3.select(this.$refs.timelineSvg).selectAll("*").remove(); // 清除旧内容
      this.drawTimeline();
    },

    //获得事件，处理事件格式
    handleEvents() {
      const parseDate = d3.timeParse("%Y-%m-%d");
      const eventData = [
        {
          eventId: 1,
          date: "2024-09-14",
          location: "New York City, USA",
          title: "Battle of New York: Avengers Assemble",
          summary:
            "The Avengers unite for the first time to stop Loki and his Chitauri army from invading Earth. With the Tesseract opening a portal to space, Iron Man, Captain America, Thor, Hulk, Black Widow, and Hawkeye fight to save New York from total destruction.",
          type: "Journalist",
          source: "www.shield.gov",
        },
        {
          eventId: 2,
          date: "2024-09-23",
          location: "Sokovia, Eastern Europe",
          title: "Age of Ultron: Sokovia Crisis",
          summary:
            "Ultron, an AI created by Tony Stark, attempts to exterminate humanity by lifting Sokovia into the sky and using it as a massive meteor. The Avengers, with the help of Wanda and Pietro Maximoff, work together to save civilians and stop Ultron's devastating plan.",
          type: "Civilian",
          source: "www.avengers.org",
        },
        {
          eventId: 3,
          date: "2024-10-02",
          location: "Wakanda, Africa",
          title: "Infinity War: Wakandan Defense",
          summary:
            "As Thanos closes in on collecting all six Infinity Stones, the Avengers and their allies fight a desperate battle in Wakanda to protect Vision and the Mind Stone. Despite their efforts, Thanos succeeds in completing the Gauntlet, leading to the Snap.",
          type: "Journalist",
          source: "www.wakandaforever.com",
        },
        {
          eventId: 4,
          date: "2024-9-23",
          location: "San Francisco, USA",
          title: "Ant-Man and the Quantum Realm Rescue",
          summary:
            "Scott Lang, with the help of Hank Pym and Hope Van Dyne, ventures into the Quantum Realm to rescue Janet Van Dyne. This marks a breakthrough in quantum mechanics and sets the stage for time travel in the fight against Thanos.",
          type: "Legal Observer",
          source: "www.pymtech.net",
        },
        {
          eventId: 5,
          date: "2024-9-27",
          location: "Avengers Headquarters, USA",
          title: "Endgame: The Final Battle",
          summary:
            "In a last-ditch effort to undo Thanos' Snap, the Avengers assemble all their remaining allies for an epic showdown. Using the Quantum Realm to gather Infinity Stones from the past, they restore the universe, but the battle comes at a great cost with Tony Stark sacrificing himself.",
          type: "Civilian",
          source: "www.avengersendgame.com",
        },
      ];

      // 将日期字符串转换为 Date 对象
      eventData.forEach((event) => {
        event.date = parseDate(event.date);
      });

      this.events = eventData;
    },

    /**
     * 生成x轴比例尺
     */
    makeScaleX() {
      this.scaleX = d3
        .scaleTime()
        .domain(this.timerange)
        .range([0, this.svgWidth - this.margin.left - this.margin.right]);
    },

    //生成y轴比例尺
    makeScaleY() {
      this.scaleY = d3
        .scaleBand()
        .domain(this.selectedEventTypes)
        .range([0, this.height - this.margin.top - this.margin.bottom])
        .padding(0.1);
    },

    //为选中的事件点外围添加虚线环
    drawHighlightCircle() {
      const svg = d3.select(this.$refs.timelineSvg);

      // 移除任何现有的虚线环
      svg.selectAll(".highlight-circle").remove();

      if (this.selectedEvent && this.selectedEvent.length > 0) {
        this.selectedEvent.forEach((event) => {
          svg
            .append("circle")
            .attr("class", "highlight-circle")
            .attr("cx", this.scaleX(event.date) + this.margin.left)
            .attr(
              "cy",
              this.scaleY(event.type) +
                this.margin.top +
                this.scaleY.bandwidth() / 2
            )
            .attr("r", 14)
            .style("fill", "none")
            .style("stroke", "#FF5722")
            .style("stroke-width", 1.5)
            .style("stroke-dasharray", "4,4");
        });
      }
    },

    /**
     * 构建新的svg
     */
    drawTimeline() {
      const svg = d3.select(this.$refs.timelineSvg);
      const width = this.svgWidth - this.margin.left - this.margin.right;
      const xScale = this.scaleX;
      const xAxis = d3
        .axisTop(xScale)
        .ticks(this.selectedInterval)
        .tickFormat(d3.timeFormat("%d %b"));

      // y轴比例尺
      this.makeScaleY();
      const yScale = this.scaleY;

      // 添加x轴
      svg
        .append("g")
        .attr("class", "x-axis")
        .attr("transform", `translate(${this.margin.left}, ${this.margin.top})`)
        .call(xAxis)
        .selectAll("text")
        .style("font-size", "12px")
        .style("fill", "black");

      // 添加事件类型标签（y轴）
      svg
        .append("g")
        .attr(
          "transform",
          `translate(${this.margin.left - 10}, ${this.margin.top})`
        )
        .call(d3.axisLeft(yScale).tickSize(0))
        .selectAll("text")
        .style("font-size", "12px")
        .style("fill", "black");

      // 修改x轴主线和刻度线的透明度
      svg
        .selectAll(".domain") // 选择x轴的主轴线
        .style("stroke-opacity", 0); // 设置透明度为0，使主轴线透明

      //为每个x轴刻度添加垂直虚线
      const ticks = xScale.ticks(this.selectedInterval);
      ticks.forEach((tick) => {
        svg
          .append("line")
          .attr("class", "grid-line")
          .attr("x1", xScale(tick) + this.margin.left)
          .attr("x2", xScale(tick) + this.margin.left)
          .attr("y1", this.margin.top)
          .attr("y2", this.height - this.margin.bottom)
          .style("stroke", "#aaa") // 线的颜色
          .style("stroke-dasharray", "4,4"); // 虚线样式
      });
      // 绘制事件横线
      this.selectedEventTypes.forEach((eventType) => {
        svg
          .append("line")
          .attr("x1", this.margin.left)
          .attr("x2", width + this.margin.left)
          .attr(
            "y1",
            yScale(eventType) + this.margin.top + yScale.bandwidth() / 2
          )
          .attr(
            "y2",
            yScale(eventType) + this.margin.top + yScale.bandwidth() / 2
          )
          .style("stroke", "#555")
          .style("stroke-dasharray", "2,2");
      });

      // 绘制事件点
      svg
        .selectAll("circle")
        .data(this.filteredEvents)
        .enter()
        .append("circle")
        .attr("cx", (d) => xScale(d.date) + this.margin.left)
        .attr(
          "cy",
          (d) => yScale(d.type) + this.margin.top + yScale.bandwidth() / 2
        )
        .attr("r", 8)
        .style("fill", "#FF5722")
        .on("click", (event, d) => {
          console.log("点击事件", event, d);
          this.selectedEvent = this.filteredEvents.filter(
            (event) => event.date.getTime() === d.date.getTime()
          );
          this.isEventDetailVisible = true;
          console.log("选中的事件合集", this.selectedEvent);
          // this.selectedEvent = d;

          // 在选中的事件点外围添加虚线环
          this.drawHighlightCircle();
        });

      // 启用拖拽
      this.enableDrag(svg);
    },

    // 启用拖拽行为
    enableDrag(svg) {
      const dragBehavior = d3
        .drag()
        .on("start", this.onDragStart)
        .on("drag", this.onDrag)
        .on("end", this.onDragEnd);

      svg.call(dragBehavior);
    },

    // 拖拽开始
    onDragStart(event) {
      console.log("onDragStart");
      event.sourceEvent.stopPropagation();
      this.dragPos0 = event.x; // 记录开始拖拽的位置
      this.toggleTransition(false);
    },
    // 拖拽进行中
    onDrag(event) {
      console.log("onDrag");
      const drag0 = this.scaleX.invert(this.dragPos0).getTime();
      const dragNow = this.scaleX.invert(event.x).getTime();
      const timeShift = (drag0 - dragNow) * this.dragScaleFactor;
      // console.log("timeshift", timeShift)

      let newDomain0 = d3.timeSecond.offset(this.timerange[0], timeShift);
      let newDomainF = d3.timeSecond.offset(this.timerange[1], timeShift);

      if (this.rangeLimits) {
        const [minDate, maxDate] = this.rangeLimits;
        newDomain0 = newDomain0 < minDate ? minDate : newDomain0;
        newDomainF = newDomainF > maxDate ? maxDate : newDomainF;
      }

      this.timerange = [newDomain0, newDomainF];
      this.makeScaleX();
      this.redrawAxisAndEvents();
    },

    // 拖拽结束
    onDragEnd() {
      console.log("dragEnd");
      this.toggleTransition(true);
    },

    // 切换过渡效果
    toggleTransition(isTransition) {
      this.transitionDuration = isTransition ? 300 : 0;
    },

    /**
     * shift time range by moving forward or backwards
     * @param {String} direction:'forward' / 'backwards'
     */
    onMoveTime(direction) {
      //计算时间范围总跨度(以分钟为单位)
      const extent = (this.timerange[1] - this.timerange[0]) / 60000;

      //计算当前时间范围的中心点
      const newCentralTime = d3.timeMinute.offset(this.timerange[0], extent);

      let domain0, domain1;
      domain0 = newCentralTime;
      domain1 = d3.timeMinute.offset(newCentralTime, extent);

      if (direction === "backwards") {
        console.log("发生后移");
        domain0 = d3.timeMinute.offset(newCentralTime, -(2 * extent));
        domain1 = d3.timeMinute.offset(newCentralTime, -extent);
      }

      this.timerange = [domain0, domain1];

      //重新绘制时间轴和事件点
      this.makeScaleX(); // 重新生成比例尺
      this.redrawAxisAndEvents(); // 更新坐标轴和事件点
    },

    // 重绘 x 轴和事件点
    redrawAxisAndEvents() {
      // 应用过渡到 x 轴
      d3.select(this.$refs.timelineSvg)
        .select(".x-axis")
        .transition()
        .duration(this.transitionDuration)
        .call(
          d3
            .axisTop(this.scaleX)
            .ticks(this.selectedInterval) // 根据 selectedRange 调整刻度
            .tickFormat(d3.timeFormat("%d %b"))
        );

      // 更新事件点位置，确保仅选择在 SVG 内的 circle 元素
      d3.select(this.$refs.timelineSvg)
        .selectAll("circle")
        .data(this.filteredEvents)
        .join("circle") // 确保重用现有的 circle 元素
        .transition()
        .duration(this.transitionDuration)
        .attr("cx", (d) => this.scaleX(d.date) + this.margin.left);

      //重新绘制虚线环
      this.drawHighlightCircle();
      // 重绘 x 轴刻度的垂直虚线
      const svg = d3.select(this.$refs.timelineSvg);

      // 更新已有虚线的位置
      let gridLines = svg
        .selectAll(".grid-line")
        .data(this.scaleX.ticks(this.selectedInterval));

      gridLines
        .transition()
        .duration(this.transitionDuration)
        .attr("x1", (d) => this.scaleX(d) + this.margin.left)
        .attr("x2", (d) => this.scaleX(d) + this.margin.left)
        .attr("y1", this.margin.top)
        .attr("y2", this.height - this.margin.bottom);

      // 添加新的虚线（如果有新的刻度出现）
      gridLines
        .enter()
        .append("line")
        .attr("class", "grid-line") // 为新元素添加 .grid-line 类
        .attr("x1", (d) => this.scaleX(d) + this.margin.left)
        .attr("x2", (d) => this.scaleX(d) + this.margin.left)
        .attr("y1", this.margin.top)
        .attr("y2", this.height - this.margin.bottom)
        .style("stroke", "#aaa")
        .style("stroke-dasharray", "4,4");

      // 删除多余的虚线
      gridLines.exit().remove();
    },
  },
};
</script>
<style lang="scss" scoped>
.timeMap {
  width: 100%;
  height: 100%;
  // background-color: #293241;
  .eventType-selector {
    ::v-deep .el-checkbox__label {
      font-size: 20px;
      color: black;
    }
    margin-bottom: 20px;
    .event-group {
      display: flex;
      flex-direction: column;
      gap: 5px;
    }
  }
  .fold-box {
    position: fixed;
    left: 50%;
    bottom: 300px;
    z-index: 1100; /* 确保在最上层 */
    cursor: pointer;
    display: flex;
    justify-content: center;
    align-items: center;
    transition: bottom 0.3s ease, background-color 0.3s ease;
    .fold-btn {
      background-color: black;
      border: none;
    }
  }
  .fold-box.hidden {
    bottom: 0px; /* 隐藏时在页面底部 */
  }

  .time-range-display {
    display: flex;
    flex-direction: column;
    gap: 5px;
    position: fixed;
    z-index: 1100;
    padding: 10px;
    border-radius: 5px;
    color: #fff;
    background-color: black;
    left: 5px;
    bottom: 300px;
    transition: bottom 0.3s ease;
    .date-text {
      font-weight: 600;
    }
  }
  .time-range-display.hidden {
    bottom: -60px;
  }

  .event-detail-box {
    position: fixed;
    z-index: 1000;
    right: 3px;
    top: 0px;
    height: calc(100% - 300px);
    width: 400px;
  }

  .moveToLeft {
    position: fixed;
    z-index: 1200;
    bottom: 10px;
    left: 50px;
    cursor: pointer;
  }
  .moveToRight {
    position: fixed;
    z-index: 1200;
    bottom: 10px;
    right: 70px;
    cursor: pointer;
  }

  .timeline {
    display: flex;
    position: fixed;
    bottom: 0; /* 固定在页面底部 */
    left: 0;
    width: 100%; /* 占满宽度 */
    z-index: 1000; /* 确保在上层 */
    transition: transform 0.3s ease-in-out;
    transform: translateY(0); /* 默认显示 */
    .time-range {
      display: flex;
      flex-direction: column;
      gap: 20px;
      justify-content: center;
      align-items: center;
      .time-range-option:hover {
        color: green;
      }

      .time-range-option.active {
        color: purple;
      }
    }
  }
  .timeline.hidden {
    transform: translateY(100%); /* 隐藏时向下移动 */
  }
}
</style>
