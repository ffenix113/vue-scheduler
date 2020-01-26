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
            Hour
          </div>
          <div class="scheduler-week-title">
            Day
          </div>
        </th>
      </tr>
      <tr>
        <td
          v-for="n in 24"
          :key="n"
          class="scheduler-hour"
          :colspan="accuracy"
          @click="handleClickHour(n-1)"
        >
          {{ n - 1 }}
        </td>
      </tr>
    </thead>
    <tbody>
      <tr
        v-for="(day, dayIdx) in eventDates"
        :key="dayIdx"
      >
        <td
          class="scheduler-day-toggle"
          @click="handleClickDay(dayIdx+1)"
        >
          {{ validDayNames[dayIdx] }}
        </td>

        <td
          v-for="hourIndex in cellColAmout"
          :key="hourIndex"
          class="scheduler-hour"
          :class="{
            'scheduler-active': isCellSelected(dayIdx+1, hourIndex - 1)
          }"
          @mousedown="handleMouseDown(dayIdx+1, hourIndex - 1)"
          @mousemove="handleMouseMove(dayIdx+1, hourIndex - 1)"
          @mouseup="handleMouseUp(dayIdx+1, hourIndex - 1)"
        />
      </tr>
    </tbody>
    <tfoot v-if="footer">
      <tr>
        <td :colspan="accuracy * 24 + 1">
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
    value: [Object, Number, String],
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
      default: 1
    },
    // footer exist?
    footer: {
      type: Boolean,
      default: true
    },
    multiple: Boolean,
    disabled: Boolean,
    eventDates: Array
  },

  data () {
    return {
      selectMode: SelectMode.NONE,
      startCoord: null,
      endCoord: null,
      selected: {},
      tempSelected: null,
      moving: false
    }
  },

  computed: {
    cellColAmout () {
      return this.accuracy * 24
    },
    validDayNames() {
      return this.eventDates.map(day => format(day, 'iii, dd/LL/yyyy'))
    },
    validDayNum() {
      return this.eventDates.length
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
        this.selected = val
        this.tempSelected = val
      },
      immediate: true
    }
  },

  methods: {
    format,
    hourSelected(value, hourIndex) {
      return ~value.indexOf(hourIndex)
    },
    isCellSelected (day, hourIndex) {
      const { tempSelected = {}} = this
      const selectedHours = tempSelected[day]
      if (selectedHours && this.hourSelected(selectedHours, hourIndex)) {
        return 'scheduler-active'
      } else {
        return ''
      }
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
      const endCoord = [day, 24 * this.accuracy - 1]
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
          if (current[i] && current[i].length) {
            res[i] = current[i].slice(0)
          }
        }
        return res
      }
      for (i = 1; i <= this.validDayNum; i++) {
        if (!current[i]) {
          if (origin[i] && origin[i].length) {
            res[i] = origin[i].slice(0)
          }
          continue
        }
        if (origin[i] && origin[i].length) {
          var m = selectMode === SelectMode.JOIN
            ? mergeArray(origin[i], current[i])
            : rejectArray(origin[i], current[i])
          m.length && (res[i] = m)
        } else if (selectMode === SelectMode.JOIN) {
          res[i] = current[i].slice(0)
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
        var data = this.selected[day]
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
      var day = this.selected[coord[0]]
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
      var currentSelectRange = makeMatrix(startCoord, endCoord)
      this.tempSelected = this.merge(this.selected, currentSelectRange, selectMode)
    },

    // 并更新选中数据
    end () {
      this.moving = false
      this.startCoord = null
      this.endCoord = null
      this.selectMode = SelectMode.NONE
      this.emitChange(this.tempSelected)
    },

    reset () {
      if (this.disabled) {
        return
      }
      this.emitChange({})
    },

    emitChange (val) {
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
