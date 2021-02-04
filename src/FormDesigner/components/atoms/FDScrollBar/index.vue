<template>
  <div
  v-on="eventStoppers()"
  @click="scrollBarClick"
  :style="outerScrollBarDivObj"
  :title="properties.ControlTipText"
  @keydown.enter="setContentEditable($event, true)"
  @keydown.esc="setContentEditable($event, false)"
  :tabindex="0"
  @mousedown="controlEditMode"
  >
    <div class="slidecontainer" :style="cssVars">
      <button :style="scrollBarButtonStyleObj" @click="!getDisableValue?properties.Min > properties.Max ? increaseTheValue() : decreaseTheValue():''">
        <FdSvgImage
          key="leftArrow"
          name="left-arrow.svg"
          @hook:mounted="changeForeColor"
          class="svgLeftRightStyle"
        />
      </button>
      <input
        :disabled="getDisableValue"
        type="range"
        :min="properties.Min > properties.Max ? properties.Max : properties.Min"
        :max="properties.Min > properties.Max ? properties.Min : properties.Max"
        :value="properties.Value"
        class="slider"
        :style="inputStyleObj"
        @input="updateValueProperty"
        orient="vertical"
      />
      <button :style="scrollBarButtonStyleObj" @click="!getDisableValue?properties.Min > properties.Max ? decreaseTheValue() : increaseTheValue():''">
        <FdSvgImage
          key="rightArrow"
          name="right-arrow.svg"
          @hook:mounted="changeForeColor"
          class="svgLeftRightStyle"
        />
      </button>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Emit, Mixins, Watch } from 'vue-property-decorator'
import FdControlVue from '@/api/abstract/FormDesigner/FdControlVue'
import FdSvgImage from '@/FormDesigner/components/atoms/FDSVGImage/index.vue'
import { controlProperties } from '@/FormDesigner/controls-properties'

@Component({
  name: 'FDScrollBar',
  components: {
    FdSvgImage
  }
})
export default class FDScrollBar extends Mixins(FdControlVue) {
  $el: HTMLDivElement
  isInvert: boolean = false
  intervalVariable: number = 0
  updateValueProperty (e: Event) {
    if (e.target instanceof HTMLInputElement) {
      const targetValue = parseInt(e.target!.value)
      this.updateDataModel({ propertyName: 'Value', value: targetValue })
    }
  }

  get cssVars () {
    const controlProp = this.properties
    return {
      position: 'relative',
      left: '0px',
      top: (this.properties.Min! > this.properties.Max!) ? this.checkOtherOrientations() ? `${controlProp.Height!}px` : '0px' : '0px',
      gridTemplateColumns: this.checkOtherOrientations() ? '20px ' + `${controlProp.Height! - 40}px` + ' 20px' : '',
      '--bg-color': this.properties.BackColor,
      '--height': this.checkOtherOrientations() ? `${this.properties.Width!}px` : `${this.properties.Height!}px`,
      transform: (this.properties.Min! > this.properties.Max!) ? this.scrollReAlign() : this.checkOtherOrientations() ? 'rotate(90deg)' : '',
      transformOrigin: (this.properties.Min! > this.properties.Max!) ? this.checkOtherOrientations() ? '0% 0%' : '' : '',
      '--invertValue': this.isEditMode ? this.isInvert ? '1' : '0' : '0'
    }
  }

  @Watch('isEditMode')
  editModeValidate () {
    if (this.isEditMode) {
      this.intervalVariable = setInterval(() => {
        this.isInvert = !this.isInvert
      }, 1000)
    } else {
      clearInterval(this.intervalVariable)
    }
  }
  scrollReAlign () {
    if (this.checkOtherOrientations()) {
      return 'rotate(-90deg)'
    }
    return 'ScaleX(-1)'
  }
  get getDisableValue () {
    if (this.isRunMode || this.isEditMode) {
      return this.properties.Enabled === false
    } else {
      return true
    }
  }
  /**
   * @description changes ForeColor property and then updates the getForeColor variable which is given to fill attribute of the svg element
   * @function getForeColorValue
   *
   */
  @Watch('properties.ForeColor', { deep: true })
  changeForeColor () {
    if (this.properties.Enabled) {
      this.$el.querySelectorAll('.foreColor').forEach((e) => {
        (e as SVGGElement).style.fill = this.getForeColorValue
      })
    } else {
      this.$el.querySelectorAll('.foreColor').forEach((e) => {
        (e as SVGGElement).style.fill = 'rgb(200,200,200)'
      })
    }
  }

  @Watch('properties.Enabled', { deep: true })
  enabledValidate () {
    this.changeForeColor()
  }
  get outerScrollBarDivObj () {
    const controlProp = this.properties
    let display = ''
    if (this.isRunMode) {
      display = controlProp.Visible ? 'inline-block' : 'none'
    } else {
      display = 'inline-block'
    }
    return {
      width: `${controlProp.Width}px`,
      height: `${controlProp.Height}px`,
      left: `${controlProp.Left}px`,
      top: `${controlProp.Top}px`,
      display: display,
      overflow: 'hidden',
      cursor:
        controlProp.MousePointer !== 0 || controlProp.MouseIcon !== ''
          ? this.getMouseCursorData
          : 'default',
      backgroundColor: 'white'
    }
  }

  get inputStyleObj () {
    const controlProp = this.properties
    let a = null
    let temprgba
    if (controlProp.BackColor!!.startsWith('rgb')) {
      a = controlProp.BackColor!.split('rgba(')[1].split(',')
    } else {
      temprgba = this.hexToRgbA(controlProp.BackColor!)
    }

    return {
      width: this.checkOtherOrientations() ? `${controlProp.Height! - 40}px` : `${controlProp.Width! - 40}px`,
      height: this.checkOtherOrientations() ? `${controlProp.Width!}px` : `${controlProp.Height!}px`,
      cursor:
        controlProp.MousePointer !== 0 || controlProp.MouseIcon !== ''
          ? this.getMouseCursorData
          : 'default',
      backgroundColor: controlProp.BackColor!.startsWith('rgb') ? `rgba(${a![0]},${a![1]},${a![2]},0.5)` : temprgba,
      margin: '0px'
    }
  }
  hexToRgbA (hex: string) {
    let c: any
    if (/^#([A-Fa-f0-9]{3}){1,2}$/.test(hex)) {
      c = hex.substring(1).split('')
      if (c.length === 3) {
        c = [c[0], c[0], c[1], c[1], c[2], c[2]]
      }
      c = '0x' + c.join('')
      return 'rgba(' + [(c >> 16) & 255, (c >> 8) & 255, c & 255].join(',') + ',0.5)'
    }
  }
  get scrollBarButtonStyleObj () {
    const controlProp = this.properties
    return {
      backgroundColor: this.cssVars['--bg-color'],
      overflow: 'hidden',
      outline: 'none',
      cursor:
        controlProp.MousePointer !== 0 || controlProp.MouseIcon !== ''
          ? this.getMouseCursorData
          : 'default',
      border: !controlProp.Enabled ? '1px solid gray' : ''
    }
  }
  mounted () {
    this.$el.focus()
  }

  /**
   * @description checkOtherOrientations returns string value from
   * controlProperties.controlsOrientationProp
   * @function checkOtherOrientations
   * @returns boolean value
   * @override
   */
  checkOtherOrientations (): boolean {
    return controlProperties.controlsOrientationProp(this.data)
  }
  eventStoppers () {
    const eventStop = (event: Event) => event.stopPropagation()
    return this.isEditMode === false ? null : {
      keydown: eventStop
    }
  }
  scrollBarClick (event: MouseEvent) {
    if (this.toolBoxSelectControl === 'Select') {
      event.stopPropagation()
      this.selectedItem(event)
    }
  }
}
</script>

<style scoped>
.slidecontainer {
  display: grid;
  grid-template-columns: 20px auto 20px;
}

.slider {
  -webkit-appearance: none;
  content: '';
  --rgb:250,250,250;
  --alpha: 0.5;
  background-color: rgba(var(--rgb), var(--alpha));
  outline: none;
  overflow: hidden;
  pointer-events: none;
}

.slider:hover {
  opacity: 1;
}

.slider::-webkit-slider-thumb {
  background-color: var(--bg-color);
  border: 2px solid gray;
  border-right: 2px solid black;
  height: var(--height);
  opacity: var(--opacityValue);
  -webkit-appearance: none;
  appearance: none;
  width: 25px;
  --alpha: 1;
  cursor: inherit;
  pointer-events:auto;
  filter: invert(var(--invertValue));
}

.slider::-moz-range-thumb {
  background: rgb(139, 138, 138);
  cursor: pointer;
}
</style>
