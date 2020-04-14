/* eslint-disable no-param-reassign */
<template>
  <div id="app">
    <header>
      <img src="./assets/logo.svg" alt="Лого" />
      <h3>Следи за расписанием наших мероприятий, мы ждем тебя!</h3>
    </header>
    <div class="el-calendar custom-calendar">
      <div class="el-calendar__header">
        <div class="el-calendar__title">
          {{ i18nDate }}
        </div>
      </div>
      <div
        class="el-calendar__body"
        v-if="validatedRange.length === 0"
        key="no-range"
      >
        <div class="el-calendar__girl"></div>
        <div class="el-calendar__boy"></div>
        <date-table
          :date="date"
          :events="events"
          :selected-day="realSelectedDay"
          slot="myslot"
          @pick="pickDay"
        />
        <div
          class="el-calendar__nav-elements"
          v-if="validatedRange.length === 0"
        >
          <div class="el-calendar__prev" @click="selectDate('prev-month')" />
          <div class="el-calendar__next" @click="selectDate('next-month')" />
        </div>
        <div>
          <div class="el-calendar__dragon"></div>
        </div>
      </div>
      <div class="el-calendar-footer">
        <div class="calendar-legend">
          <div class="calendar-legend__event">
            <div class="calendar-legend-shape">Название</div>
            <span>- мероприятия</span>
          </div>
          <div class="calendar-legend__big-event">
            <div class="calendar-legend-shape">Название</div>
            <span>- масштабные мероприятия</span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import Locale from "element-ui/src/mixins/locale";
import locale from "element-ui/lib/locale/lang/ru-RU";

import fecha from "element-ui/src/utils/date";
import { validateRangeInOneMonth } from "element-ui/src/utils/date-util";
import DateTable from "./date-table";

const validTypes = ["prev-month", "today", "next-month"];
const oneDay = 86400000;

export default {
  name: "ElCalendar",

  mixins: [Locale],

  components: {
    DateTable,
  },

  props: {
    value: [Date, String, Number],
    range: {
      type: Array,
      validator(range) {
        if (Array.isArray(range)) {
          return (
            range.length === 2 &&
            range.every(
              (item) =>
                typeof item === "string" ||
                typeof item === "number" ||
                item instanceof Date
            )
          );
        }
        return true;
      },
    },
  },

  provide() {
    return {
      elCalendar: this,
    };
  },

  methods: {
    // 接收子组件选中的日期
    pickDay(day) {
      // 设置为选中日
      this.realSelectedDay = day;
    },
    /**
      点击上月今天下月
     */
    selectDate(type) {
      if (validTypes.indexOf(type) === -1) {
        throw new Error(`invalid type ${type}`);
      }
      let day = "";
      if (type === "prev-month") {
        // 上月第一天
        day = `${this.prevMonthDatePrefix}-01`;
      } else if (type === "next-month") {
        // 下月第一天
        day = `${this.nextMonthDatePrefix}-01`;
      } else {
        // 今天
        day = this.formatedToday;
      }

      if (day === this.formatedDate) return;
      this.pickDay(day);
    },

    toDate(val) {
      if (!val) {
        throw new Error("invalid val");
      }
      return val instanceof Date ? val : new Date(val);
    },
  },

  computed: {
    prevMonthDatePrefix() {
      const temp = new Date(this.date.getTime());
      // 获取上个月最后一天
      temp.setDate(0);
      // 返回上个月年月
      return fecha.format(temp, "yyyy-MM");
    },

    curMonthDatePrefix() {
      return fecha.format(this.date, "yyyy-MM");
    },

    nextMonthDatePrefix() {
      const temp = new Date(
        this.date.getFullYear(),
        this.date.getMonth() + 1,
        1
      );
      return fecha.format(temp, "yyyy-MM");
    },

    formatedDate() {
      return fecha.format(this.date, "yyyy-MM-dd");
    },

    i18nDate() {
      const year = this.formatedDate.slice(0, 4);
      const month = this.formatedDate.slice(5, 7).replace("0", "");
      return `${locale.el.datepicker[`month${month}`]}, ${year}`;
    },

    formatedToday() {
      return fecha.format(this.now, "yyyy-MM-dd");
    },
    // 动态计算选中日,重写get,set方法
    realSelectedDay: {
      get() {
        if (!this.value) return this.selectedDay;
        return this.formatedDate;
      },
      set(val) {
        this.selectedDay = val;
        const date = new Date(val);
        // 此处双向绑定,相当于v-model
        this.$emit("input", date);
      },
    },
    // 没有range时,把date传给子组件date-table
    date() {
      if (!this.value) {
        if (this.realSelectedDay) {
          return new Date(this.selectedDay);
        } else if (this.validatedRange.length) {
          return this.validatedRange[0][0];
        }
        return this.now;
      }
      return this.toDate(this.value);
    },

    // if range is valid, we get a two-digit array
    validatedRange() {
      let range = this.range;
      if (!range) return [];
      const expetedMap = {
        0: {
          value: 1,
          message: "start of range should be Monday.",
        },
        1: {
          value: 0,
          message: "end of range should be Sunday.",
        },
      };
      range = range.reduce((prev, val, index) => {
        const date = this.toDate(val);
        if (date.getDay() !== expetedMap[index].value) {
          console.warn(
            "[ElementCalendar]",
            expetedMap[index].message,
            " invalid range will be ignored"
          );
        } else {
          prev = prev.concat(date);
        }
        return prev;
      }, []);
      if (range.length === 2) {
        const [start, end] = range;
        if (start > end) {
          console.warn(
            "[ElementCalendar]end time should be greater than start time"
          );
          return [];
        }
        // start time and end time in one month
        if (validateRangeInOneMonth(start, end)) {
          return [[start, end]];
        }
        const data = [];
        let startDay = new Date(start.getFullYear(), start.getMonth() + 1, 1);
        const lastDay = this.toDate(startDay.getTime() - oneDay);
        if (!validateRangeInOneMonth(startDay, end)) {
          console.warn(
            "[ElementCalendar]start time and end time interval must not exceed two months"
          );
          return [];
        }
        data.push([start, lastDay]);
        let interval = startDay.getDay();
        interval = interval <= 1 ? Math.abs(interval - 1) : 8 - interval;
        startDay = this.toDate(startDay.getTime() + interval * oneDay);
        if (startDay.getDate() < end.getDate()) {
          data.push([startDay, end]);
        }
        return data;
      }
      return [];
    },
  },

  data() {
    return {
      selectedDay: "2020-04-10",
      events: [
        {
          name: "Ледяное шоу Ледяное шоу Ледяное шоу",
          park: "Радуга Радуга Радуга Радуга Радуга",
          date: "05-04-2020",
          big: true,
        },
        {
          name: "Огненное шоу",
          park: "Радуга",
          date: "05-04-2020",
          big: false,
        },
        {
          name: "Ледяное шоу Ледяное шоу Ледяное шоу",
          park: "Радуга Радуга Радуга Радуга Радуга",
          date: "05-04-2020",
          big: true,
        },
        {
          name: "Огненное шоу",
          park: "Радуга",
          date: "05-04-2020",
          big: false,
        },
        {
          name: "Огненное шоу",
          park: "Радуга",
          date: "08-04-2020",
          big: true,
        },
        {
          name: "Огненное шоу",
          park: "Радуга",
          date: "08-04-2020",
          big: true,
        },
        {
          name: "Огненное шоу",
          park: "Радуга",
          date: "18-04-2020",
          big: true,
        },
        {
          name: "Огненное шоу",
          park: "Радуга",
          date: "20-04-2020",
          big: false,
        },
      ],
      now: new Date(),
    };
  },
};
</script>

<style>
#app {
  text-align: center;
  background: #fff center url("./assets/wave.svg") no-repeat;
  background-size: cover;
}

@font-face {
  font-family: "museo_sans_cyrl500";
  src: url("./assets/fonts/museo_sans_cyrillic-webfont.woff2") format("woff2"),
    url("./assets/fonts/museo_sans_cyrillic-webfont.woff") format("woff");
  font-weight: normal;
  font-style: normal;
}

@font-face {
  font-family: "museo_sans_cyrl900";
  src: url("./assets/fonts/museosanscyrl_3-webfont.woff2") format("woff2"),
    url("./assets/fonts/museosanscyrl_3-webfont.woff") format("woff");
  font-weight: normal;
  font-style: normal;
}

@font-face {
  font-family: "Shadow-Regular";
  src: url("./assets/fonts/Shadow-Regular.eot?#iefix")
      format("embedded-opentype"),
    url("./assets/fonts/Shadow-Regular.woff") format("woff"),
    url("./assets/fonts/Shadow-Regular.ttf") format("truetype"),
    url("./assets/fonts/Shadow-Regular.svg#Shadow-Regular") format("svg");
  font-weight: normal;
  font-style: normal;
}

header {
  margin-bottom: 42px;
}

header h3 {
  font-family: "museo_sans_cyrl500", sans-serif;
  font-weight: 500;
  margin: 0;
}

.custom-calendar .el-calendar__title {
  width: 100%;
  font-family: "museo_sans_cyrl500", sans-serif;
  font-style: normal;
  font-weight: 600;
  font-size: 32px;
}

.custom-calendar .el-calendar__button-group {
  position: absolute;
  top: 50%;
}

.custom-calendar .el-calendar-table thead th {
  font-size: 24px;
  font-family: "museo_sans_cyrl500", sans-serif;
  font-style: normal;
  font-weight: 600;
  letter-spacing: -0.03em;
  color: #303133;
}

.custom-calendar .el-calendar-table thead th:nth-of-type(6),
.custom-calendar .el-calendar-table thead th:nth-of-type(7) {
  text-align: right;
  padding-right: 20px;
}

.custom-calendar.el-calendar {
  background: transparent;
}

.custom-calendar .el-calendar__header {
  border: 0;
}

.custom-calendar .el-calendar-table .el-calendar-day {
  font-family: "museo_sans_cyrl500", sans-serif;
  font-style: normal;
  font-weight: normal;
  letter-spacing: -0.03em;
  color: #303133;
  display: flex;
  padding: 11px;
  min-height: 117px;
}

.custom-calendar .el-calendar-table .el-calendar-day span {
  position: relative;
}

.custom-calendar .el-calendar-table .prev .el-calendar-day,
.custom-calendar .el-calendar-table .next .el-calendar-day {
  color: #cccccc;
}

.el-calendar-table td {
  position: relative;
}

.custom-calendar .el-calendar-table .is-today .el-calendar-day {
  color: #ee3a85;
  font-family: "museo_sans_cyrl900", sans-serif;
  font-weight: bold;
}

.custom-calendar .el-calendar-table td,
.custom-calendar .el-calendar-table tr:first-child td {
  border-color: #ebebeb;
}
.custom-calendar .el-calendar__body {
  position: relative;
  width: 1062px;
  margin: 0 auto;
  padding: 0;
}

.custom-calendar .el-calendar-table tbody {
  background: #ffffff;
  box-shadow: 0px 4px 24px rgba(219, 219, 219, 0.25);
}

.custom-calendar .el-calendar__nav-elements {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  width: 100%;
}

.custom-calendar .el-calendar__prev,
.custom-calendar .el-calendar__next {
  position: absolute;
  background: transparent 0 0 url("./assets/arrow.svg") no-repeat;
  width: 43px;
  height: 43px;
  background-size: contain;
  cursor: pointer;
  transition: 0.2s all;
}

.custom-calendar .el-calendar__prev {
  left: -59px;
}

.custom-calendar .el-calendar__next {
  right: -59px;
  transform: rotate(180deg);
}

.custom-calendar .el-calendar__next:hover {
  transform: rotate(180deg) translateX(-3px);
}

.custom-calendar .el-calendar__prev:hover {
  transform: translateX(-3px);
}

.custom-calendar .el-calendar-table {
  position: relative;
}

.el-calendar__dragon {
  background: transparent 0 0 url("./assets/dragon.png") no-repeat;
  background-size: contain;
  width: 142px;
  height: 110px;
  position: absolute;
  top: -51px;
  right: 30px;
}

.el-calendar__girl {
  background: transparent 0 0 url("./assets/girl.svg") no-repeat;
  background-size: contain;
  width: 224px;
  height: 409px;
  position: absolute;
  bottom: 0;
  left: -188px;
}
.el-calendar__boy {
  background: transparent 0 0 url("./assets/boy.svg") no-repeat;
  background-size: contain;
  width: 195px;
  height: 336px;
  position: absolute;
  bottom: 0;
  right: -143px;
}
.el-calendar__boy:after {
  content: "";
  background: transparent 0 0 url("./assets/hand.png") no-repeat;
  background-size: contain;
  width: 17px;
  height: 30px;
  position: absolute;
  z-index: 1;
  top: 20px;
  left: 40px;
}
.custom-calendar .event-day {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: space-evenly;
  position: absolute;
  width: 100%;
  height: 100%;
  left: 0;
  top: 0;
}

.custom-calendar .event-day.event-scroll {
  justify-content: flex-start;
  overflow: auto;
}

.custom-calendar .event-row {
  background: transparent 0 0 url("./assets/blue-clouds.svg") no-repeat;
  background-size: cover;
  height: 48px;
  width: 100%;
  max-width: 144px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  padding: 5px;
  box-sizing: border-box;
  word-break: break-word;
  margin: 2px 0;
  flex-shrink: 0;
}

.custom-calendar .event-row.big-event {
  background-image: url("./assets/pink-cloud.svg");
}

.custom-calendar .event-name {
  font-family: "Shadow-Regular", sans-serif;
  font-size: 13px;
  line-height: 13px;
  letter-spacing: -0.03em;
  color: #003045;
  font-weight: normal;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}
.custom-calendar .event-park {
  font-family: "museo_sans_cyrl500", sans-serif;
  font-size: 12px;
  line-height: 12px;
  letter-spacing: -0.03em;
  color: #003045;
}

.custom-calendar .event-day .event-row:only-child {
  background-image: url("./assets/big-blue-cloud.png");
  width: 100%;
  height: 100%;
  max-width: 100%;
}

.custom-calendar .event-day .event-row.big-event:only-child {
  background-image: url("./assets/big-pink-cloud.png");
}

.calendar-legend {
  font-family: "museo_sans_cyrl500", sans-serif;
  font-size: 18px;
  line-height: 28px;
  letter-spacing: -0.03em;
  display: flex;
  justify-content: space-between;
  width: 790px;
  align-self: center;
  margin: 16px auto;
}

.calendar-legend-shape {
  border-radius: 12px;
  padding: 2px 49px 2px 8px;
  display: inline-block;
}

.calendar-legend__big-event .calendar-legend-shape {
  background: #ffcaf3;
}
.calendar-legend__event .calendar-legend-shape {
  background: #a7e4ff;
}

.calendar-legend__big-event {
}
</style>
