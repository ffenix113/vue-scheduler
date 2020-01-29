<template>
  <table
    class="scheduler"
    :class="{
      'scheduler-disabled': disabled
    }"
  >
    <thead>
      <tr>
        <th rowspan="2" class="slash">
          <div class="scheduler-time-title">
            Day
          </div>
          <div class="scheduler-week-title">
            Hour
          </div>
        </th>
      </tr>
      <tr>
        <td
          v-for="(day, dayIdx) in eventDates"
          :key="dayIdx"
          class="scheduler-day-toggle"
          @click="handleClickDay(dayIdx+1)"
        >
          {{ validDayNames[dayIdx] }}
        </td>
      </tr>
    </thead>
    <tbody>
      <tr
        v-for="hourPart in cellColAmount"
        :key="hourPart"
      >
        <td
          v-if="hourPart % accuracy === 1"
          class="scheduler-hour"
          :rowspan="accuracy"
          @click="handleClickHour((hourPart-1) / accuracy)"
        >
          {{ (hourPart-1) / accuracy }}:00
        </td>

        <td
          v-for="dayIdx in eventDates.length"
          :key="dayIdx"
          class="scheduler-hour"
          :class="{
            'scheduler-active': allHourSelected(dayIdx, hourPart-1)
          }"
          :style="{
            'opacity': opacity(dayIdx, hourPart-1) + '%'
          }"
          @mousedown="handleMouseDown(dayIdx, hourPart-1)"
          @mousemove="handleMouseMove(dayIdx, hourPart-1)"
          @mouseup="handleMouseUp(dayIdx, hourPart-1)"
        />
      </tr>
    </tbody>
    <tfoot v-if="footer">
      <tr>
        <td :colspan="eventDates.length + 1">
          <span class="scheduler-tips">Drag to select hours</span>
          <a
            class="scheduler-reset"
            @click="reset"
          >Reset Selected</a>
        </td>
      </tr>
    </tfoot>
  </table>
</template>

<script>
import SelectMode from '@/constants/SelectMode'
import { makeMatrix, mergeArray, rejectArray, sortCoord } from '@/utils/helper'
import format from 'date-fns/format'

export default {
  name: 'Scheduler',

  props: {
    value: [Array, Object, Number, String],
    /**
     * 将用户自定义的值解码为 VueScheduler 的标准值。
     * @param {Object} selected 已选值
     * @param {Number} accuracy 精度
     */
    decoder: Function,
    /**
     * 将 VueScheduler 的标准值编码为用户自定义的值。
     * @param {Object} selected 已选值
     * @param {Number} accuracy 精度
     */
    encoder: Function,
    // how many cells of an hour
    accuracy: {
      type: Number,
      default: 2
    },
    // footer exist?
    footer: {
      type: Boolean,
      default: true
    },
    multiple: Boolean,
    disabled: Boolean,
    eventDates: Array,
    userId: String
  },

  data () {
    return {
      selectMode: SelectMode.NONE,
      startCoord: null,
      endCoord: null,
      allSelected: {},
      // userSelected holds time object for current user
      userSelected: {},
      // userSelectedIdx hold index in event for current user
      userSelectedIdx: {},
      moving: false,
      selectedCache: {},
      isSelectedCached: false,
      // haveUserSelected tells if user have actually selected
      // at least one cell
      haveUserSelected: false
    }
  },

  computed: {
    cellColAmount () {
      return this.accuracy * 24
    },
    validDayNames() {
      return this.eventDates.map(day => format(day, 'iii, dd/LL/yyyy'))
    },
    validDayNamesISO() {
      return this.eventDates.map(day => format(day, 'yyyy-LL-dd'))
    },
    validDayNum() {
      return this.eventDates.length
    },
    opacityShift() {
      return this.haveUserSelected ? 0 : 1
    }
  },

  watch: {
    value: {
      handler (val) {
        if (this.decoder) {
          val = this.decoder(val, this.accuracy)
        } else if (this.$SCHEDULER.decoder) {
          val = this.$SCHEDULER.decoder(val, this.accuracy)
        }
        const self = this
        let currUserValueIdx = val.findIndex(config => config.user_id === self.userId)
        if (currUserValueIdx === -1) {
          const currUserValue = { user_id: self.userId, times: {}}
          val.push(currUserValue)
          currUserValueIdx = val.length - 1
        }
        this.userSelected = val[currUserValueIdx].times
        this.haveUserSelected = Object.keys(this.userSelected).length !== 0
        this.userSelectedIdx = currUserValueIdx
        this.allSelected = val
      },
      immediate: true
    }
  },

  methods: {
    format,
    isoDay(day) {
      return this.validDayNamesISO[day - 1]
    },
    opacity(day, hourIndex) {
      let currSelected = 0
      const _selected = this.selectedCache[day + '_' + hourIndex]
      if (_selected !== undefined) {
        currSelected = _selected
      }
      const userSelectedTimes = this.userSelected[this.isoDay(day)]
      currSelected += (userSelectedTimes && this.hourSelected(userSelectedTimes, hourIndex)) ? 1 : 0
      return (currSelected / (this.allSelected.length - this.opacityShift)) * 100
    },
    allHourSelected(day, hourIndex) {
      if (!this.isSelectedCached) {
        for (const val of this.allSelected) {
          this.validDayNamesISO.forEach((isoDay, idx) => {
            if (val.user_id === this.userId || !val.times.hasOwnProperty(isoDay)) {
              return
            }
            const selectedHours = val.times[isoDay]
            const self = this
            selectedHours.forEach(hourPart => {
              if (!self.selectedCache.hasOwnProperty((idx + 1) + '_' + (hourPart - 1))) {
                self.selectedCache[(idx + 1) + '_' + (hourPart - 1)] = 0
              }
              self.selectedCache[(idx + 1) + '_' + (hourPart - 1)] += 1
            })
          })
        }
        this.isSelectedCached = true
      }
      if (this.selectedCache[day + '_' + hourIndex] !== undefined) {
        return true
      }

      const isoDay = this.isoDay(day)
      const userSelectedTimes = this.userSelected[isoDay]
      return userSelectedTimes && this.hourSelected(userSelectedTimes, hourIndex)
    },
    hourSelected(value, hourIndex) {
      return ~value.indexOf(hourIndex)
    },
    isCellSelected (day, hourIndex) {
      const isoDay = this.isoDay(day - 1)
      for (const val of this.allSelected) {
        if (val.user_id === this.userId) {
          continue
        }
        const selectedHours = val.times[isoDay]
        if (selectedHours && this.hourSelected(selectedHours, hourIndex)) {
          return true
        }
      }
      const userSelectedTimes = this.userSelected[isoDay]
      const userSelected = userSelectedTimes && this.hourSelected(userSelectedTimes)
      console.log(day, hourIndex, userSelected)
      return userSelected
    },
    /**
     * toggle hour
     * @param {Number} hour 0 - 23
     */
    handleClickHour (hour) {
      if (this.disabled) {
        return
      }
      const fromIndex = hour * this.accuracy
      const toIndex = fromIndex + this.accuracy - 1
      const startCoord = [1, fromIndex] // [row, col] row start form 1
      const endCoord = [this.validDayNum, toIndex]
      const selectMode = this.getRangeSelectMode(startCoord, endCoord)
      this.updateToggle(startCoord, endCoord, selectMode)
    },
    handleClickDay (day) {
      if (this.disabled) { return }
      const startCoord = [day, 0] // [row, col] row start form 1
      const endCoord = [day, this.cellColAmount - 1]
      const selectMode = this.getRangeSelectMode(startCoord, endCoord)
      this.updateToggle(startCoord, endCoord, selectMode)
    },
    handleMouseDown (row, col) {
      if (this.disabled) {
        return
      }
      this.moving = true
      this.startCoord = [row, col]
      this.endCoord = this.startCoord.slice(0)
      this.selectMode = this.getCellSelectMode(this.startCoord)
      this.updateRange(this.startCoord, this.endCoord, this.selectMode)
    },
    handleMouseMove (row, col) {
      if (!this.moving) {
        return false
      }
      if (!this.selectMode || !this.startCoord || (this.endCoord &&
                                                  this.endCoord[0] === row &&
                                                  this.endCoord[1] === col)
      ) {
        return false
      }
      this.endCoord = [row, col]
      this.updateRange(this.startCoord, this.endCoord, this.selectMode)
    },
    handleMouseUp (row, col) {
      if (!this.moving) {
        return false
      }
      // 起始点都在同一个位置
      if (this.startCoord[0] === this.endCoord[0] &&
        this.startCoord[1] === this.endCoord[1]) {
        this.updateRange(this.startCoord, this.endCoord, this.selectMode)
      }
      this.end()
    },
    /**
   * 根据选择模式合并合个集合
   * @param {Object} origin 上一次选中的数据
   * @param {Object} current 当前选中的数据
   * @param {Number} selectMode 选择模式 {0: none, 1: 选择（合并）模式, 2: 排除模式（从选区中减去）}
   */
    merge (origin, current, selectMode) {
      var res = {}
      // 替换模式下，弃用之前的选区，直接使用当前选区
      let i
      if (selectMode === SelectMode.REPLACE) {
        for (i = 1; i <= this.validDayNum; i++) {
          const isoDay = this.isoDay(i)
          if (current[isoDay] && current[isoDay].length) {
            res[isoDay] = current[isoDay].slice(0)
          }
        }
        return res
      }
      for (i = 1; i <= this.validDayNum; i++) {
        const isoDay = this.isoDay(i)
        if (!current[isoDay]) {
          if (origin[isoDay] && origin[isoDay].length) {
            res[isoDay] = origin[isoDay].slice(0)
          }
          continue
        }
        if (origin[isoDay] && origin[isoDay].length) {
          const m = selectMode === SelectMode.JOIN
            ? mergeArray(origin[isoDay], current[isoDay])
            : rejectArray(origin[isoDay], current[isoDay])
          m.length && (res[isoDay] = m)
        } else if (selectMode === SelectMode.JOIN) {
          res[isoDay] = current[isoDay].slice(0)
        }
      }
      return res
    },
    /**
     * 根据当前选中的范围内时间格式的空闲情况，决定是全选还是全不选
     * 全空闲：总时间格目 === 空闲时间格数目
     * 部分空闲：总时间格目 !== 空闲时间格数目
     * 无空闲：空闲时间格数目 === 0
     * 状态切换：
     * 当前范围全空闲 > 采用合并模式，全选当前范围
     * 部分空闲 > 采用合并模式，全选当前范围
     * 无空闲 > 采用合并模式，全不选当前范围
     *
     * @param {Array} startCoord 起始坐标 [row, col]
     * @param {Array} endCoord 终结坐标 [row, col]
     * @return {SelectMode}
     */
    getRangeSelectMode (startCoord, endCoord) {
      if (!this.multiple) {
        return SelectMode.REPLACE
      }
      var rowRange = sortCoord(startCoord[0], endCoord[0])
      var colRange = sortCoord(startCoord[1], endCoord[1])
      var startRow = rowRange[0]
      var endRow = rowRange[1]
      var startCol = colRange[0]
      var endCol = colRange[1]
      var rows = endRow - startRow + 1
      var cols = endCol - startCol + 1
      var total = rows * cols

      // 计算已使用的时间格子
      // TODO 未过滤 disabled 的格子
      var used = 0
      for (var i = 0; i < rows; i++) {
        var day = startRow + i
        var data = this.allSelected[day]
        if (!data) {
          continue
        }
        for (var j = 0; j < data.length; j++) {
          if (data[j] >= startCol && data[j] <= endCol) {
            used++
          }
        }
      }

      return total === used ? SelectMode.MINUS : SelectMode.JOIN
    },

    /**
     * 根据当前选中的时间格式的空闲情况，决定是全选还是全不选
     * 状态切换：
     * 当前为单选模式(multiple=false)，-> 替换模式
     * 当前选中时间段为空闲 -> 全选不
     * 当前选中时间段为无空闲 - > 全不选
     *
     * @param {Array} coord 起始坐标 [row, col]
     * @return {SelectMode}
     */
    getCellSelectMode (coord) {
      if (!this.multiple) {
        return SelectMode.REPLACE
      }
      // TODO 未过滤 disabled 的格子
      const day = this.userSelected[this.isoDay(coord[0])]
      return day && this.hourSelected(day, coord[1]) ? SelectMode.MINUS : SelectMode.JOIN
    },

    /**
     * 根据当前的选中坐标系更新视图，并更新选中数据
     * @param {Array} startCoord 起始坐标 [row, col]
     * @param {Array} endCoord 终结坐标 [row, col]
     * @param {SelectMode} selectMode 选择模式
     */
    updateToggle (startCoord, endCoord, selectMode) {
      this.updateRange(startCoord, endCoord, selectMode)
      this.end()
    },

    /**
   * 根据当前的选中坐标系更新视图
   * @param {Array} startCoord 起始坐标 [row, col]
   * @param {Array} endCoord 终结坐标 [row, col]
   * @param {SelectMode} selectMode 选择模式
   */
    updateRange (startCoord, endCoord, selectMode) {
      var currentSelectRange = makeMatrix(this.isoDay, startCoord, endCoord)
      this.userSelected = this.merge(this.userSelected, currentSelectRange, selectMode)
      this.haveUserSelected = Object.keys(this.userSelected).length !== 0
    },

    // 并更新选中数据
    end () {
      this.moving = false
      this.startCoord = null
      this.endCoord = null
      this.selectMode = SelectMode.NONE
      this.emitChange(this.allSelected)
    },

    reset () {
      if (this.disabled) {
        return
      }
      this.haveUserSelected = false
      this.userSelected = {}
      this.emitChange(this.allSelected)
    },

    emitChange (val) {
      val[this.userSelectedIdx].times = this.userSelected
      if (this.encoder) {
        val = this.encoder(val, this.accuracy)
      } else if (this.$SCHEDULER.encoder) {
        val = this.$SCHEDULER.encoder(val, this.accuracy)
      }
      this.$emit('input', val)
      this.$emit('change', val)
    }
  }
}
</script>

<style lang="scss">
 @import '../scss/index.scss';
</style>
