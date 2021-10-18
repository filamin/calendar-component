<template>
  <div id="app">
    <div class="container">
      <!-- Типа хлебные крошки -->
      <div class="breadcrumbs">
        <span>Главная</span>
        <i class="arrow right"/>
        <span>Сервисы</span>
        <i class="arrow right"/>
        <span class="selected">Планирование</span>
      </div>
      <h1>Планирование</h1>
      <div class="grid">
        <div class="card main">
          <div class="title">
            <h3>{{ selectedMonth }}</h3>
            <div class="buttons">
              <button @click="selectPreviousMonth" class="previous">
                <i class="arrow left"/>
              </button>
              <button @click="selectNextMonth" class="next">
                <i class="arrow right"/>
              </button>
            </div>
          </div>
          <hr/>
          <div class="calendar">
            <div class="weekdays">
            <span v-for="(weekday, i) in weekdays" :key="i" class="weekday">
              {{ weekday }}
            </span>
            </div>
            <div class="days">
              <div v-for="day in daysWithWorkerDataAndWeekends" :key="day.date" class="day">
              <span class="date" :class="{ 'muted' : !day.isCurrentMonth }">
                {{ getDay(day) }}
              </span>
                <div v-if="day.option" class="option" :style="{ background: day.color }">
                  {{ day.option }}
                </div>
              </div>
            </div>
          </div>
        </div>
        <div class="card worker">
          <h4>Сотрудник</h4>
          <select class="custom-select" v-model="worker">
            <option v-for="(worker, idx) in workerOptions" :key="idx" :value="worker">
              {{ worker.name }}
            </option>
          </select>
        </div>
        <div class="card stat">
          <div class="row">
            <!-- Не понял логики-->
            <span class="counter">0</span>
            <span class="label">поездки</span>
          </div>
          <hr/>
          <div class="row">
            <span class="counter">{{ dayInBusinessTrip }}</span>
            <span class="label">дней командировки</span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import dayjs from 'dayjs';
import weekday from 'dayjs/plugin/weekday';
import weekOfYear from 'dayjs/plugin/weekOfYear';
import updateLocale from 'dayjs/plugin/updateLocale';

import { WEEKDAYS } from './constants';
import {
  DEFAULT_HOLIDAYS,
  DEFAULT_WORKER_DATA1,
  DEFAULT_WORKER_DATA2,
} from './hardcodeWorkerData';

dayjs.extend(weekday);
dayjs.extend(weekOfYear);
dayjs.extend(updateLocale);
dayjs.updateLocale('en', {
  months: [
    'Январь', 'Февраль', 'Март', 'Апрель', 'Май', 'Июнь', 'Июль',
    'Август', 'Сентябрь', 'Октябрь', 'Ноябрь', 'Декабрь',
  ],
});

export default {
  name: 'App',
  data() {
    return {
      weekdays: WEEKDAYS,
      selectedDate: dayjs(),
      worker: {
        name: 'Имя',
        dates: [],
      },
      workerOptions: [DEFAULT_WORKER_DATA1, DEFAULT_WORKER_DATA2],
    };
  },
  computed: {
    days() {
      return [
        ...this.previousMonthDays,
        ...this.currentMonthDays,
        ...this.nextMonthDays,
      ];
    },

    daysWithWorkerDataAndWeekends() {
      return [...this.days].map((day) => {
        const dayDefault = day;
        const cityIndex = this.worker.dates.findIndex((i) => i.date === day.date);
        const holidayIndex = DEFAULT_HOLIDAYS.findIndex((i) => i === day.date);

        if (day.isWeekend) {
          dayDefault.option = 'Выходной';
        }

        if (holidayIndex !== -1) {
          dayDefault.option = 'Праздник';
        }

        if (cityIndex !== -1) {
          dayDefault.option = this.worker.dates[cityIndex].city;
          dayDefault.color = this.worker.dates[cityIndex].color;
          dayDefault.isBusinessTrip = true;
        }

        if (!day.isWeekend && cityIndex === -1 && holidayIndex === -1) {
          dayDefault.option = '';
          dayDefault.color = '';
          dayDefault.isBusinessTrip = false;
        }

        return dayDefault;
      });
    },

    month() {
      return Number(this.selectedDate.format('M'));
    },

    year() {
      return Number(this.selectedDate.format('YYYY'));
    },

    selectedMonth() {
      return this.selectedDate.format('MMMM YYYY');
    },

    numberOfDaysInMonth() {
      return dayjs(this.selectedDate).daysInMonth();
    },

    dayInBusinessTrip() {
      return this.daysWithWorkerDataAndWeekends.reduce((sum, day) => sum + !!day.isBusinessTrip, 0);
    },

    currentMonthDays() {
      return [...Array(this.numberOfDaysInMonth)].map((day, index) => {
        const dateString = `${this.year}-${this.month}-${index + 1}`;
        return {
          date: dayjs(dateString).format('YYYY-MM-DD'),
          isCurrentMonth: true,
          isWeekend: dayjs(dateString).day() === 0 || dayjs(dateString).day() === 6,
        };
      });
    },

    previousMonthDays() {
      const firstDayOfTheMonthWeekday = this.getWeekday(this.currentMonthDays[0].date);
      const previousMonth = dayjs(`${this.year}-${this.month}-01`).subtract(1, 'month');
      const visibleNumberOfDaysFromPreviousMonth = firstDayOfTheMonthWeekday
        ? firstDayOfTheMonthWeekday - 1
        : 6;

      const previousMonthLastMondayDayOfMonth = dayjs(this.currentMonthDays[0].date)
        .subtract(visibleNumberOfDaysFromPreviousMonth, 'day')
        .date();

      return [...Array(visibleNumberOfDaysFromPreviousMonth)].map((day, index) => {
        const dateString = `${previousMonth.year()}-${previousMonth.month() + 1}-${previousMonthLastMondayDayOfMonth + index}`;
        return {
          date: dayjs(dateString).format('YYYY-MM-DD'),
          isCurrentMonth: false,
          isWeekend: dayjs(dateString).day() === 0 || dayjs(dateString).day() === 6,
        };
      });
    },

    nextMonthDays() {
      const lastDayOfTheMonthWeekday = this.getWeekday(`${this.year}-${this.month}-${this.currentMonthDays.length}`);

      const nextMonth = dayjs(`${this.year}-${this.month}-01`).add(1, 'month');

      const visibleNumberOfDaysFromNextMonth = lastDayOfTheMonthWeekday
        ? 7 - lastDayOfTheMonthWeekday
        : lastDayOfTheMonthWeekday;

      return [...Array(visibleNumberOfDaysFromNextMonth)].map((day, index) => {
        const dateString = `${nextMonth.year()}-${nextMonth.month() + 1}-${index + 1}`;
        return {
          date: dayjs(dateString).format('YYYY-MM-DD'),
          isCurrentMonth: false,
          isWeekend: dayjs(dateString).day() === 0 || dayjs(dateString).day() === 6,
        };
      });
    },
  },

  methods: {
    getWeekday(date) {
      return dayjs(date).weekday();
    },

    getDay(day) {
      return dayjs(day.date).format('D');
    },

    selectDate(newSelectedDate) {
      this.selectedDate = newSelectedDate;
    },

    selectPreviousMonth() {
      this.selectedDate = dayjs(this.selectedDate).subtract(1, 'month');
    },

    selectNextMonth() {
      this.selectedDate = dayjs(this.selectedDate).add(1, 'month');
    },
  },
};
</script>

<style lang="scss">
$primary: #002033;
$secondary: rgba(0, 32, 51, 0.6);
$black: #232A3E;
$background: #E5E5E5;
* {
  margin: 0;
  padding: 0;
}
#app {
  background: $background;
  font-family: Roboto condensed, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  /* Styles */
  .container {
    margin: 0 auto;
    padding: 32px 16px;

    .breadcrumbs {
      font-size: 18px;
      line-height: 150%;
      color: $secondary;
      margin-bottom: 15px;
      width: fit-content;
      span:not(:last-child) {
        margin-right: 20px;
      }
      span:not(:first-child) {
        margin-left: 20px;
      }
      .selected {
        color: $primary;
      }
      .arrow {
        border-color: $secondary;
      }
    }
    h1 {
      margin-bottom: 20px;
      font-weight: bold;
      font-size: 72px;
      line-height: 120%;
      color: $primary;
    }
    .grid {
      display: grid;
      grid-template:
            "main worker" min-content
            "main stat"   min-content
            "main space"  auto / 2fr 1fr;
      grid-gap: 20px;
    }
    .card {
      background: white;
      border-radius: 7px;
      padding: 24px 32px 66px;
      &.main {
        grid-area: main;
        padding: 24px 24px 66px;
        hr {
          margin-bottom: 32px;
        }
      }
      &.worker {
        grid-area: worker;
        padding: 24px;
        h4 {
          font-weight: bold;
          font-size: 22px;
          color: $primary;
          margin-bottom: 20px;
        }
      }
      &.stat {
        grid-area: stat;
        padding: 24px;
        hr {
          margin: 12px 0;
        }
        .row {
          display: flex;
          justify-content: space-between;
          align-items: center;
          .counter {
            font-size: 32px;
            line-height: 120%;
          }
          .label {
            font-size: 16px;
            line-height: 105%;
          }
        }
      }
      .title {
        width: 100%;
        display: inline-flex;
        justify-content: space-between;
        align-items: center;
        margin-bottom: 14px;
        h3 {
          font-weight: bold;
          font-size: 32px;
          line-height: 150%;
          color: $primary;
        }
        .buttons {
          display: inline-flex;
          button {
            color: white;
            width: 35px;
            height: 35px;
            border-radius: 50%;
            border: none;
            &.next {
              background: #0078D2;
              .arrow {
                border-color: white;
              }
            }
            &.previous {
              margin-right: 5px;
              background: rgba(0, 66, 105, 0.07);
              .arrow {
                border-color: rgba(0, 57, 92, 0.8);
              }
            }
          }
        }
      }
    }
    .calendar {
      .weekdays {
        display: grid;
        grid-template: auto / repeat(7, 1fr);
        margin-bottom: 8px;
        .weekday {
          font-weight: bold;
          font-size: 12px;
          line-height: 14px;
          color: $black;
        }
      }
      .days {
        display: grid;
        grid-template: auto / repeat(7, 1fr);
        border: 2px solid #E9E9E9;
        grid-gap: 2px;
        background-color: #E9E9E9;
        .day {
          background-color: white;
          height: 90px;
          padding: 9px 7px;
          display: flex;
          flex-direction: column;
          justify-content: space-between;
          align-items: flex-end;
          .date {
            font-size: 16px;
            line-height: 14px;
            color: #232A3E;
            &.muted {
              color: #C0C3CC;
            }
          }
          .option {
            font-size: 12px;
            line-height: 21px;
            color: #FAFAFA;
            background: #97B2C4;
            border-radius: 4px;
            padding: 1px 10px;
          }
        }
      }
    }
  }
  .arrow {
    border: solid black;
    border-width: 0 2px 2px 0;
    display: inline-block;
    padding: 3px;
    vertical-align: middle;
    &.right {
      transform: rotate(-45deg);
    }

    &.left {
      transform: rotate(135deg);
    }

    &.up {
      transform: rotate(-135deg);
    }

    &.down {
      transform: rotate(45deg);
    }
  }
  .custom-select {
    padding: 8px 16px;
    border-color:  rgba(0, 66, 105, 0.28);
    border-radius: 4px;
  }
  hr {
    background: rgba(0, 65, 102, 0.2);
    opacity: 0.3;
  }
}
</style>
