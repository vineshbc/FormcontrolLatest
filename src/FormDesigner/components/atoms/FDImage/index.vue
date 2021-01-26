<template>
  <div
    class="picture"
    :style="cssStyleProperty"
    :title="properties.ControlTipText"
    @keydown.delete="deleteItem"
    @click="imageClick"
  ></div>
</template>

<script lang="ts">
import {
  Component,
  Prop,
  Model,
  Emit,
  Mixins,
  Ref,
  Watch
} from 'vue-property-decorator'
import FdControlVue from '@/api/abstract/FormDesigner/FdControlVue'
import { controlProperties } from '@/FormDesigner/controls-properties'

@Component({
  name: 'FDImage'
})
export default class FDImage extends Mixins(FdControlVue) {
  $el!: HTMLDivElement;
  /**
   * @description updating width and height of image control
   * @function updateAutoSize
   * @param oldVal previous AutoSize value
   * @param newVal  new/changed AutoSize value
   */
  @Watch('properties.AutoSize')
  updateAutoSize (newVal: boolean, oldVal: boolean) {
    this.setHeightWidth(newVal)
  }

  @Watch('properties.Picture')
  pictureChanges (newVal: string, oldVal: string) {
    this.setHeightWidth(this.properties.AutoSize!)
  }
  /**
   * @description When AutoSize is true update width and height property
   * @function setHeightWidth
   * @param isAutoSize AutoSize property value
   *
   */
  setHeightWidth (isAutoSize: boolean) {
    const that = this
    if (isAutoSize && this.properties.Picture) {
      const img = new Image()
      img.onload = function () {
        that.updateDataModel({
          propertyName: 'Height',
          value: img.height
        })
        that.updateDataModel({
          propertyName: 'Width',
          value: img.width
        })
      }
      img.src = this.properties.Picture
    } else {
      return undefined
    }
  }
  /**
   * @description style object is passed to :style attribute in label tag
   * dynamically changing the styles of the component based on propControlData
   * @function cssStyleProperty
   *
   */
  protected get cssStyleProperty (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    let display = ''
    if (this.isRunMode) {
      display = controlProp.Visible ? 'block' : 'none'
    } else {
      display = 'block'
    }
    return {
      ...this.baseStyle,
      left: `${controlProp.Left}px`,
      width: `${controlProp.Width}px`,
      height: `${controlProp.Height}px`,
      top: `${controlProp.Top}px`,
      backgroundColor: controlProp.BackStyle
        ? controlProp.BackColor
        : 'transparent',
      borderColor: controlProp.BorderStyle === 1 ? controlProp.BorderColor : '',
      borderLeft: controlProp.BorderStyle === 1 ? '1px solid ' + controlProp.BorderColor : controlProp.SpecialEffect === 2 ? '2px solid gray' : controlProp.SpecialEffect === 3 ? '1.5px solid gray' : controlProp.SpecialEffect === 4 ? '0.5px solid gray' : '',
      borderRight: controlProp.BorderStyle === 1 ? '1px solid ' + controlProp.BorderColor : controlProp.SpecialEffect === 1 ? '2px solid gray' : controlProp.SpecialEffect === 4 ? '1.5px solid gray' : controlProp.SpecialEffect === 3 ? '0.5px solid gray' : '',
      borderTop: controlProp.BorderStyle === 1 ? '1px solid ' + controlProp.BorderColor : controlProp.SpecialEffect === 2 ? '2px solid gray' : controlProp.SpecialEffect === 3 ? '1.5px solid gray' : controlProp.SpecialEffect === 4 ? '0.5px solid gray' : '',
      borderBottom: controlProp.BorderStyle === 1 ? '1px solid ' + controlProp.BorderColor : controlProp.SpecialEffect === 1 ? '2px solid gray' : controlProp.SpecialEffect === 4 ? '1.5px solid gray' : controlProp.SpecialEffect === 3 ? '0.5px solid gray' : '',
      cursor:
        controlProp.MousePointer !== 0 || controlProp.MouseIcon !== ''
          ? this.getMouseCursorData
          : 'default',
      backgroundImage: `url(${controlProp.Picture})`,
      backgroundSize: controlProp.Picture === '' ? '' : this.getSizeMode,
      backgroundRepeat: this.getRepeatData,
      backgroundPosition: controlProp.Picture === '' ? '' : this.getPosition,
      display: display
    }
  }
  imageClick (event: MouseEvent) {
    if (this.toolBoxSelectControl === 'Select') {
      event.stopPropagation()
      this.selectedItem(event)
    }
  }
}
</script>

<style scoped>
.picture {
  /* position: relative; */
  overflow: hidden;
  box-sizing: border-box;
  width: 0px;
  height: 0px;
  left: 0px;
  top: 0px;
}
</style>
