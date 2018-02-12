<template>
    <div class="inline-editor">
        <div @click="handleDivClick()" v-show="!showInput">
            {{value || '&nbsp'}}
        </div>
        <input type="text" ref="input" v-show="showInput" :value="value" @input="$emit('input', $event.target.value)"
               @focusout.nativ="showInput = !showInput">
    </div>
</template>

<script>
  export default {
    name: 'app-inline-editor',
    props: {
      value: {
        required: true
      }
    },
    data () {
      return {
        showInput: false
      }
    },
    methods: {
      handleDivClick () {
        this.showInput = !this.showInput
        this.$nextTick(() => {
          this.$refs.input.focus()
        })
      }
      // focus () {
      //   this.$nextTick(() => {
      //     this.$refs.input.focus()
      //   })
      // }
    }
  }
</script>

<style scoped>
    .inline-editor::after {
        /*content: '\2006'*/
    }
    .inline-editor input {
        width: 150px
    }
</style>