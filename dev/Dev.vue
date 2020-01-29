<template>
  <div>
    <scheduler
      v-model="selected"
      :multiple="multiple"
      :footer="footer"
      :disabled="disabled"
      :accuracy="accuracy"
      :encoder="encoder"
      :decoder="decoder"
      :starts-at="startsAt"
      :ends-at="endsAt"
      :user-id="this.userId"
    />
    <div style="margin: 10px;">
      <div>
        <label for="">Accuracy: </label>
        <input v-model="acc" type="number">
      </div>
      <div>
        <label>Disabled: <input v-model="disabled" type="checkbox"></label>
      </div>
      <div>
        <label>Multiple: <input v-model="multiple" type="checkbox"></label>
      </div>
      <div>
        <label>Footer: <input v-model="footer" type="checkbox"></label>
      </div>
    </div>
    <div style="padding: 10px; background-color: #ececec;">
      <pre>{{ value }}</pre>
    </div>
  </div>
</template>

<script>
import parseISO from 'date-fns/parseISO'
import format from 'date-fns/format'
import eachDayOfInterval from 'date-fns/eachDayOfInterval'
export default {
  data () {
    return {
      disabled: false,
      footer: true,
      accuracy: 2,
      multiple: false,
      decoder: this.parse,
      encoder: this.serialize,
      selected: [
        {
          user_id: '211',
          times: { '2019-01-01': [1, 2, 3] }
        },
        {
          user_id: '212',
          times: { '2019-01-01': [2, 3, 4, 5] }
        },
        {
          user_id: '222',
          times: { '2019-01-02': [2, 3, 4, 5] }
        }
      ],
      userId: '222',
      startsAt: parseISO('2019-01-01'),
      endsAt: parseISO('2019-01-04')
    }
  },
  computed: {
    acc: {
      get () {
        return this.accuracy
      },
      set (val) {
        if (val < 1) {
          return
        }
        this.accuracy = parseInt(val, 10)
      }
    },
    value () {
      return JSON.stringify(this.selected, '', 2)
    },
    validDaysStrings() {
      const formatDate = function(date) {
        return format(date, 'yyyy-LL-dd')
      }
      return this.validDays().map(formatDate)
    }
  },
  methods: {
    validDays() {
      return eachDayOfInterval({
        start: parseISO('2019-01-01'),
        end: parseISO('2019-01-04')
      })
    },
    serialize(data, accuracy) {
      this.selected = data
      return data
    },
    parse(strSequence, accuracy) {
      return strSequence
    }
    // serialize (data, accuracy) {
    //   if (data === null || data === undefined) return data
    //   const newData = {}
    //   for (const [day, selectedSlots] of Object.entries(data)) {
    //     newData[this.validDaysStrings[day - 1]] = selectedSlots
    //   }
    //   return JSON.stringify(newData)
    // },
    // parse  (strSequence, accuracy) {
    //   const data = JSON.parse(strSequence)
    //   const newData = {}
    //   for (const [day, selectedSlots] of Object.entries(data)) {
    //     newData[this.validDaysStrings.indexOf(day) + 1] = selectedSlots
    //   }
    //   return newData
    //   // return strSequence
    // }
  }
}
</script>

<style>

</style>
