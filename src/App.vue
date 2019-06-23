<template>
  <div id="app">
    <div v-if="!isMobile && workflow.length" class="workflow-graph workflow-graph--forward-main">
      <div v-for="(item, index) in workflow"
        :key="`${item.type}_${item.id}-${index}`"
        :ref="`${item.type}_${item.id}`"
        :class="[ item.type === 'step' ? 'step' : 'move',
          index === 0 ? 'step--first' : false,
          item.position === 'LAST' ? 'step--last' : false,
          item.backwardFlow && item.backwardFlow.length ? 'has-backward' : false ]"
        class="flow-item">
        <span>{{ item.name }}</span>
        <div class="workflow-graph workflow-graph--backward">
          <div v-for="(backItem, index) in item.backwardFlow"
            :key="`${backItem.type}_${backItem.id}-${index}`"
            :style="backwardStyle(backItem.type)"
            :class="[ backItem.type === 'step' ? 'step' : 'move',
              backItem.type === 'move' && backItem.isBackward ? 'move--back' : false ]"
            class="flow-item flow-item--backward">
            <span>{{ backItem.name }}</span>
            <div v-if="backItem.type === 'step'" id="after-clone"></div>
          </div>
        </div>
      </div>
    </div>
    <div v-if="isMobile" class="workflow-graph--warning">Sorry, but you will need a bigger screen to see workflow graph.</div>
  </div>
</template>

<script>
import api from '@/api'

const get = require('lodash.get')
const cloneDeep = require('lodash.clonedeep')

export default {
  name: 'app',
  data() {
    return {
      isMobile: false,
      steps: [],
      moves: []
    }
  },
  computed: {
    workflow () {
      const workflow = []
      let steps = cloneDeep(this.steps)
      let moves = cloneDeep(this.moves)
      if (steps.length) workflow.push(this.firstStep(steps))
      else return false
      steps = steps.filter(stp => stp.id !== workflow[0].id)
      for (let i = steps.length; i >= steps.length; i--) {
        if (!steps.length) break
        // eslint-disable-next-line
        for (let step of steps) {
          let lastItem = workflow[workflow.length - 1]
          if (lastItem.type === 'move') {
            let moveGoesTo = get(lastItem, 'to.id')
            let foundStep = steps.find(stp => stp.id === moveGoesTo)
            steps = steps.filter(step => step.id !== foundStep.id)
            if (!foundStep.isBackward) {
              foundStep.type = 'step'
              foundStep.backwardFlow = []
              workflow.push(foundStep)
            }
          } else {
            let lastStep = workflow[workflow.length - 1]
            for (let move of moves) {
              move.type = 'move'
              // case for specific 'finished' step
              if (lastStep.id === 2 && get(move, 'from.id') === 42 && get(move, 'to.id') === 2) {
                lastStep.backwardFlow.push(move)
                let reworkStep = steps.find(stp => stp.id === 42)
                reworkStep.type = 'step'
                lastStep.backwardFlow.push(reworkStep)
                moves = moves.filter(mv => mv.id !== move.id)
                steps = steps.filter(step => step.id !== 42)
              } else if (get(move, 'from.id') === lastStep.id) {
                if (!move.isBackward) {
                  workflow.push(move)
                } else {
                  lastStep.backwardFlow.push(move)
                }
                moves = moves.filter(mv => mv.id !== move.id)
              }
            }
          }
        }
      }
      return workflow
    }
  },
  methods: {
    firstStep (steps) {
      let firstStep = steps.find(step => step.position === "FIRST")
      firstStep.type = 'step'
      return firstStep
    },
    backwardStyle (type) {
      if (type === 'step') {
        return { top: 'calc(32rem - 2px)' }
      } else {
        return { top: 'calc(18rem - 2px)' }
      }
    },
    renderAfterClone (el) {
      const html = document.getElementsByTagName('html')[0]
      const htmlFontSize = parseFloat(window.getComputedStyle(html).fontSize).toFixed(2)
      const backStep = document.querySelectorAll('.flow-item--backward.step')[0]
      const stepsWithBack = document.querySelectorAll('.step.has-backward')
      const lastStepWithBack = stepsWithBack[stepsWithBack.length - 1]
      const backStepPosRight = Math.round(backStep.getBoundingClientRect().right)
      const lastStepWithBackPosCenter = Math.round(lastStepWithBack.getBoundingClientRect().left + lastStepWithBack.getBoundingClientRect().width / 2)
      const calcWidth = (lastStepWithBackPosCenter - backStepPosRight) / htmlFontSize
      el.style.cssText = `width: ${calcWidth}rem; right: calc(-${calcWidth}rem + -2px);`
    }
  },
  created () {
    // fetch steps
    api.get('steps')
      .then(res => {
        this.steps = res.data
        return Promise.resolve(res)
      })
      .catch(err => {
        // eslint-disable-next-line
        console.error(err)
        return Promise.reject(err)
      })
    // fetch moves
    api.get('moves')
      .then(res => {
        this.moves = res.data
        return Promise.resolve(res)
      })
      .catch(err => {
        // eslint-disable-next-line
        console.error(err)
        return Promise.reject(err)
      })
  },
  mounted () {
    if (window.innerWidth < 768) this.isMobile = true
  },
  updated () {
    const afterClone = document.getElementById('after-clone')
    if (afterClone) this.renderAfterClone(afterClone)
  }
}
</script>

<style lang="scss">
@import "./assets/style/main";

#app {
  font-family: "Open Sans", sans-serif;
  @include fontSizeRem(10, 14);
}
.workflow-graph {
  display: flex;
  flex-direction: row;
  align-items: center;
  width: 100%;
  padding: 4rem;
  &--backward {
    flex-direction: column;
    padding: 0;
  }
  &--warning {
    position: absolute;
    top: 50%;
    transform: translateY(-50%);
    width: 100%;
    color: $red;
    text-align: center;
  }
}
.flow-item {
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
  width: 10rem;
  margin-right: 8rem;
  color: $lightblack;
  text-align: center;
  border: 2px solid $darkgrey;
  &:after {
    content: "";
    position: absolute;
    top: 50%;
    right: calc(-8rem + -2px);
    transform: translateY(-50%);
    width: 8rem;
    height: 4px;
    background-color: $darkgrey;
  }
  &--backward {
    position: absolute;
    &.move {
      &:after {
        top: auto;
        left: 50%;
        transform: translateX(-50%);
        bottom: calc(-13rem + -4px);
        width: 4px;
        height: calc(13rem + 4px);
      }
      &--back {
        color: $white;
        background-color: $red;
      }
    }
    &.step {
      color: $white;
      background-color: $blue;
      &:after {
        content: none;
      }
      #after-clone {
        position: absolute;
        top: 50%;
        transform: translateY(-50%);
        height: 4px;
        background-color: $darkgrey;
      }
    }
  }
  span {
    position: absolute;
  }
}
.step {
  height: 10rem;
  border-radius: 50%;
  &--first {
    background-color: $yellow;
  }
  &--last {
    color: $white;
    background-color: $green;
    &:after {
      content: none;
    }
  }
  &.has-backward {
    &:before {
      content: "";
      position: absolute;
      bottom: calc(-8rem + -2px);
      width: 4px;
      height: 8rem;
      background-color: $darkgrey;
    }
  }
}
.move {
  height: 6rem;
}
</style>
