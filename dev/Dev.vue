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
      :valid-days="validDays"
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
export default {
  data () {
    return {
      disabled: false,
      footer: true,
      accuracy: 2,
      multiple: false,
      decoder: this.parse,
      encoder: this.serialize,
      selected: '{}',
      validDays: [
        '2020-01-01',
        '2020-01-02'
      ]
    }
  },
  computed: {
    acc: {
      get () {
        return this.accuracy
      },
      set (val) {
        this.accuracy = parseInt(val, 10)
      }
    },
    value () {
      return JSON.stringify(this.selected, '', 2)
    }
  },
  methods: {
    serialize (data, accuracy) {
      if (data === null || data === undefined) return data
      const newData = {}
      for (const [day, selectedSlots] of Object.entries(data)) {
        newData[this.validDays[day - 1]] = selectedSlots
      }
      return JSON.stringify(newData)
    },
    parse  (strSequence, accuracy) {
      const data = JSON.parse(strSequence)
      const newData = {}
      for (const [day, selectedSlots] of Object.entries(data)) {
        newData[this.validDays.indexOf(day) + 1] = selectedSlots
      }
      return newData
      // return strSequence
    }
  }
}
</script>

<style>

</style>
