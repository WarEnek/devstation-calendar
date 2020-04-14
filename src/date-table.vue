<script>
import fecha from "element-ui/src/utils/date";
import {
  range as rangeArr,
  getFirstDayOfMonth,
  getPrevMonthLastDays,
  getMonthDays,
  validateRangeInOneMonth,
} from "element-ui/src/utils/date-util";
import locale from "element-ui/lib/locale/lang/ru-RU";

export default {
  props: {
    selectedDay: String, // formated date yyyy-MM-dd
    range: {
      type: Array,
      validator(val) {
        if (!(val && val.length)) return true;
        const [start, end] = val;
        return validateRangeInOneMonth(start, end);
      },
    },
    date: Date,
    hideHeader: Boolean,
    events: Array,
  },

  inject: ["elCalendar"],

  methods: {
    // 本月分割成7个数组,赋给rows,动态渲染
    toNestedArr(days) {
      return rangeArr(days.length / 7).map((_, index) => {
        const start = index * 7;
        return days.slice(start, start + 7);
      });
    },

    getFormateDate(day, type) {
      if (!day || ["prev", "current", "next"].indexOf(type) === -1) {
        throw new Error("invalid day or type");
      }
      let prefix = this.curMonthDatePrefix;
      if (type === "prev") {
        prefix = this.prevMonthDatePrefix;
      } else if (type === "next") {
        prefix = this.nextMonthDatePrefix;
      }
      day = `00${day}`.slice(-2);
      return `${prefix}-${day}`;
    },

    getCellClass({ text, type }) {
      const classes = [type];
      if (type === "current") {
        const date = this.getFormateDate(text, type);
        if (date === this.selectedDay) {
          classes.push("is-selected");
        }
        if (date === this.formatedToday) {
          classes.push("is-today");
        }
      }
      return classes;
    },

    getEvent({ text, type }) {
      const date = this.getFormateDate(text, type);
      const filterEvent = this.events.filter(
        (event) => event.date.split("-").reverse().join("-") === date
      );

      if (filterEvent.length !== 0) {
        return (
          <div
            class={
              filterEvent.length > 2 ? "event-scroll event-day" : "event-day"
            }
          >
            {filterEvent.map((event) => {
              return (
                <div class={event.big ? "big-event event-row" : "event-row"}>
                  <strong class="event-name">{event.name}</strong>
                  <div class="event-park">{event.park}</div>
                </div>
              );
            })}
          </div>
        );
      }
      return null;
    },
    // 点击哪天事件往上传递,日期传出去,父组件接收pick事件
    pickDay({ text, type }) {
      const date = this.getFormateDate(text, type);
      this.$emit("pick", date);
    },

    cellRenderProxy({ text, type }) {
      const render = this.elCalendar.$scopedSlots.dateCell;
      if (!render) return <span>{text}</span>;

      const day = this.getFormateDate(text, type);
      const date = new Date(day);
      const data = {
        isSelected: this.selectedDay === day,
        type: `${type}-month`,
        day,
        kp: true,
      };
      return render({ date, data });
    },
  },

  computed: {
    prevMonthDatePrefix() {
      const temp = new Date(this.date.getTime());
      temp.setDate(0);
      return fecha.format(temp, "yyyy-MM");
    },

    curMonthDatePrefix() {
      const temp = new Date(this.date.getTime());
      temp.setDate(0);

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

    formatedToday() {
      return this.elCalendar.formatedToday;
    },

    isInRange() {
      return this.range && this.range.length;
    },
    /*
      动态计算rows根据接收的date
     */
    rows() {
      let days = [];
      // if range exists, should render days in range.
      if (this.isInRange) {
        const [start, end] = this.range;
        const currentMonthRange = rangeArr(
          end.getDate() - start.getDate() + 1
        ).map((_, index) => ({
          text: start.getDate() + index,
          type: "current",
        }));
        let remaining = currentMonthRange.length % 7;
        remaining = remaining === 0 ? 0 : 7 - remaining;
        const nextMonthRange = rangeArr(remaining).map((_, index) => ({
          text: index + 1,
          type: "next",
        }));
        days = currentMonthRange.concat(nextMonthRange);
      } else {
        const date = this.date;
        // 获取当月第一天
        const firstDay = getFirstDayOfMonth(date);
        // 根据当月第一天计算当月中上个月显示几天
        const prevMonthDays = getPrevMonthLastDays(date, firstDay - 1).map(
          (day) => ({
            text: day,
            type: "prev",
          })
        );
        // 获取本月多少天
        const currentMonthDays = getMonthDays(date).map((day) => ({
          text: day,
          type: "current",
        }));
        days = [...prevMonthDays, ...currentMonthDays];
        // 日历一共6行每周7天共42天,42-上个月-本月=下月天数
        const nextMonthDays = rangeArr(35 - days.length).map((_, index) => ({
          text: index + 1,
          type: "next",
        }));
        // 连起来共多少天
        days = days.concat(nextMonthDays);
      }
      return this.toNestedArr(days);
    },
  },

  data() {
    const dayNames = Object.values(locale.el.datepicker.weeks);

    return {
      DAYS: dayNames.slice(1).concat(dayNames[0]),
    };
  },

  render() {
    const thead = this.hideHeader ? null : (
      <thead>
        {this.DAYS.map((day) => (
          <th key={day}>{day}</th>
        ))}
      </thead>
    );
    return (
      <table
        class={{
          "el-calendar-table": true,
          "is-range": this.isInRange,
        }}
        cellspacing="0"
        cellpadding="0"
      >
        {thead}
        <tbody>
          {this.rows.map((row, index) => (
            <tr
              class={{
                "el-calendar-table__row": true,
                "el-calendar-table__row--hide-border":
                  index === 0 && this.hideHeader,
              }}
              key={index}
            >
              {row.map((cell, key) => (
                <td
                  key={key}
                  class={this.getCellClass(cell)}
                  onClick={this.pickDay.bind(this, cell)}
                >
                  <div class="el-calendar-day">
                    {this.getEvent(cell)}
                    {this.cellRenderProxy(cell)}
                  </div>
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>
    );
  },
};
</script>
