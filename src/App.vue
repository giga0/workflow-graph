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
        <div v-if="item.position !== 'LAST'" class="arrow"></div>
        <div v-if="item.backwardFlow && item.backwardFlow.length" class="workflow-graph workflow-graph--backward">
          <div v-for="(backItem, index) in item.backwardFlow"
            :key="`${backItem.type}_${backItem.id}-${index}`"
            :style="backwardStyle(backItem.type)"
            :class="[ backItem.type === 'step' ? 'step' : 'move',
              backItem.type === 'move' && backItem.isBackward ? 'move--back' : false ]"
            class="flow-item flow-item--backward">
            <span>{{ backItem.name }}</span>
            <div class="arrow arrow--first"></div>
            <div class="arrow arrow--second"></div>
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
              if (lastStep.name === 'finished' && get(move, 'from.name') === 'rework asked' && get(move, 'to.name') === 'finished') {
                lastStep.backwardFlow.push(move)
                let reworkStep = steps.find(stp => stp.name === 'rework asked')
                reworkStep.type = 'step'
                lastStep.backwardFlow.push(reworkStep)
                moves = moves.filter(mv => mv.id !== move.id)
                steps = steps.filter(step => step.name !== 'rework asked')
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
      el.style.cssText = `width: calc(${calcWidth}rem + 3px); right: calc(-${calcWidth}rem - 3px);`
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
    right: -8rem;
    transform: translateY(-50%);
    width: 8rem;
    height: 2px;
    background-color: $darkgrey;
  }
  &--backward {
    position: absolute;
    &.move {
      &:after {
        top: auto;
        left: 50%;
        transform: translateX(-50%);
        bottom: calc(-8rem - 2px);
        width: 2px;
        height: calc(8rem - 2px);
      }
      .arrow {
        left: 50%;
        transform: translateX(-50%);
        border-width: 0rem 1rem 2rem 1rem;
        border-color: transparent transparent $darkgrey transparent;
        &--first {
          top: calc(-8rem - 3px);
        }
        &--second {
          top: calc(6rem - 2px);
        }
      }
      &--back {
        color: $white;
        background-color: $red;
        &:after {
          bottom: calc(-13rem + 2px);
          height: calc(13rem - 2px);
        }
        .arrow {
          top: auto;
          left: 50%;
          transform: translateX(-50%);
          border-width: 2rem 1rem 1rem 1rem;
          border-color: $darkgrey transparent transparent transparent;
          &--first {
            bottom: calc(5rem - 2px);
          }
          &--second {
            bottom: calc(-14rem - 1px);
          }
        }
      }
    }
    &.step {
      color: $white;
      background-color: $blue;
      &:after {
        content: none;
      }
      .arrow {
        right: calc(-2rem - 2px);
        border-width: 1rem 2rem 1rem 0;
        border-color: transparent $darkgrey transparent transparent;
      }
      #after-clone {
        position: absolute;
        top: 50%;
        transform: translateY(-50%);
        height: 2px;
        background-color: $darkgrey;
      }
    }
  }
  span {
    position: absolute;
  }
  .arrow {
    position: absolute;
    top: 50%;
    right: calc(-8rem - 2px);
    transform: translateY(-50%);
    width: 0;
    height: 0;
    border-style: solid;
    border-width: 1rem 0 1rem 2rem;
    border-color: transparent transparent transparent $darkgrey;
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
      bottom: calc(-8rem - 2px);
      width: 2px;
      height: 8rem;
      background-color: $darkgrey;
    }
  }
}
.move {
  height: 6rem;
}
</style>
