<template>
  <div class="q-pa-md">
    <q-slider
      v-model="value"
      :min="-20"
      :max="20"
      :step="4"
      label
      color="light-green"
    />
  </div>
</template>

<script>
export default {
  data () {
    return {
      value: 0
    }
  }
}
</script>
