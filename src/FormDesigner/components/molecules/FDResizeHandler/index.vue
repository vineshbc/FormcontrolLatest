<template>
  <div>
    <!-- <div :class="getMainSelected ? 'resize-area' : null"> -->
    <div v-for="handlerName in handlers" :key="handlerName">
      <div
        v-if="controlType === 'control'"
        :class="[getMainSelected ? `handle handle-${handlerName}`: null]"
        :style="handlerStyle"
        @mousedown.stop="handleMouseDown($event, handlerName, controlType, controlId)"
      ></div>

      <div
        v-else-if="controlType === 'userform'"
        :class="[`userFormHandle userFormHandle-${handlerName}`]"
        @mousedown.stop="handleMouseDown($event, handlerName, controlType)"
      ></div>
    </div>
    <div v-show="isMove" class="m-top-b move-border" :style="getLTStyle"/>
    <div v-show="isMove" class="m-right-b move-border" :style="getRStyle"/>
    <div v-show="isMove" class="m-bottom-b move-border" :style="getBStyle"/>
    <div v-show="isMove" class="m-left-b move-border" :style="getLTStyle"/>
  </div>
</template>

<script lang="ts">
import { Component, Vue, Prop, Emit, Watch } from 'vue-property-decorator'
import { State } from 'vuex-class'
import { EventBus } from '@/FormDesigner/event-bus'
import FDCommonMethod from '@/api/abstract/FormDesigner/FDCommonMethod'
@Component({
  name: 'Resizehandler',
  components: {}
})
export default class Resizehandler extends FDCommonMethod {
  $el: HTMLDivElement
  @State((state) => state.fd.selectedControls) selectedControls!: fdState['selectedControls'];
  @State((state) => state.fd.userformData) userformData!: userformData;

  @Prop({ required: true, type: String }) public controlId!: string;
  @Prop({ required: true, type: String }) public userFormId: string;
  @Prop({ required: true, type: String }) public controlType: string;
  @Prop() public size: {width: number, height: number}

  positions: IMousePosition = {
    clientX: 0,
    clientY: 0,
    movementX: 0,
    movementY: 0,
    offsetX: 0,
    offsetY: 0
  };
  resizeDiv: string = '';

  handlers: Array<String> = ['tl', 'tm', 'tr', 'mr', 'br', 'bm', 'bl', 'ml'];

  isGroupActivated: number = -1
  currentMouseDownEvent: CustomMouseEvent | null = null
  @Emit('updateResize')
  private updateResize (value: IResizeValue) {
    return value
  }
  @Emit('muldragControl')
  private muldragControl (event: MouseEvent, handler: string) {
    return { event: event, handler: handler }
  }

  @Emit('updateUserFormResize')
  private updateUserFormResize (value: IResizeValue) {
    return value
  }
  @Emit('createGroup')
  createGroup (groupId: string) {
    return groupId
  }

  @Emit('updateIsMove')
  updateIsMove (val: boolean) {
    return val
  }

  isMove = false
  isMainSelect = false
  isSelctedControl: string = ''
  currentELPosition: any = null
  created () {
    EventBus.$on('getMoveValue', this.getMoveValue)
    EventBus.$on('startMoveControl', this.startMoveControl)
    EventBus.$on('moveControl', this.moveControl)
    EventBus.$on('endMoveControl', this.endMoveControl)
  }
  destroyed () {
    EventBus.$off('getMoveValue', this.getMoveValue)
    EventBus.$off('startMoveControl', this.startMoveControl)
    EventBus.$off('moveControl', this.moveControl)
    EventBus.$off('endMoveControl', this.endMoveControl)
  }
  getMoveValue (callBack: Function) {
    if (this.isMainSelect) {
      if (this.userformData[this.userFormId][this.controlId].type === 'Frame' || this.userformData[this.userFormId][this.controlId].type === 'MultiPage') {
        callBack(this.positions.offsetX, this.positions.offsetY, this.isSelctedControl, this.getContainerList(this.isSelctedControl)[0])
      } else {
        callBack(this.positions.offsetX, this.positions.offsetY, this.controlId)
      }
    }
  }
  startMoveControl (event: MouseEvent) {
    EventBus.$emit('handleName', 'notDrag')
    if (this.getIsMoveTarget) {
      this.positions.clientX = event.clientX
      this.positions.clientY = event.clientY
    }
  }

  moveControl (event: MouseEvent, controltype: string) {
    if (this.getIsMoveTarget) {
      this.moveBorder(event)
      if (event.movementX !== 0 && event.movementY !== 0) {
        const containerType = this.userformData[this.userFormId][this.controlId].type
        if (containerType === 'Frame' || containerType === 'MultiPage') {
          EventBus.$emit('handleName', 'frameDrag')
          if (controltype === containerType) {
            this.isMove = true
          }
        } else {
          EventBus.$emit('handleName', 'drag')
          this.isMove = true
          this.updateIsMove(true)
        }
      }
    }
  }
  endMoveControl () {
    if (this.getIsMoveTarget) {
      this.positions.movementX = 0
      this.positions.movementY = 0
      this.isMove = false
    }
  }
  /**
   * @description Implementation  of Logic for resize and drag the control and userform, it preserve initial position of control/userform position
   * @function handleMouseDown
   * @param event it is type of  MouseEvent
   * @param handler specifies handlerName(for example topLeft, BottomRight etc)
   * @param controlType To differentiate between userform and control resize logic
   */
  handleMouseDown (event: CustomMouseEvent, handler: string, controlType: string, controlID: string) {
    this.isSelctedControl = controlID
    this.updateIsMove(false)
    EventBus.$emit('handleName', 'notDrag')
    const controlGroup = this.userformData[this.userFormId][this.controlId].properties.GroupID
    this.isGroupActivated = this.selectedControls[this.userFormId].selected.findIndex((val: string) => val.startsWith('group'))
    if (this.isGroupActivated === -1 || (controlGroup === '' && !this.selectedControl.includes(this.controlId))) {
      this.resizeDiv = handler
      this.positions.clientX = event.clientX
      this.positions.clientY = event.clientY
      this.currentMouseDownEvent = event
      if (controlType === 'control') {
        const containerType = this.userformData[this.userFormId][this.controlId].type
        if (handler !== 'drag') {
          document.onmousemove = this.elementDrag
        } else {
          this.positions.offsetX = event.offsetX
          this.positions.offsetY = event.offsetY
          this.isMainSelect = true
          EventBus.$emit('startMoveControl', event)
          if (containerType === 'Frame' || containerType === 'MultiPage') {
            document.onmousemove = (event: MouseEvent) => { this.moveControl(event, containerType) }
          } else {
            document.onmousemove = (event: MouseEvent) => { EventBus.$emit('moveControl', event) }
          }
        }
      } else {
        document.onmousemove = this.userFormResize
      }
      document.onmouseup = this.closeDragElement
    } else {
      this.muldragControl(event, handler)
    }
  }

  moveBorder (event: MouseEvent) {
    event.preventDefault()
    this.positions.movementX = this.positions.clientX - event.clientX
    this.positions.movementY = this.positions.clientY - event.clientY
  }
  get getLTStyle () {
    return {
      left: `${-this.positions.movementX}px`,
      top: `${-this.positions.movementY}px`
    }
  }
  get getRStyle () {
    return this.size ? {
      left: `${this.size.width - this.positions.movementX}px`,
      top: `${-this.positions.movementY}px`
    } : null
  }
  get getBStyle () {
    return this.size ? {
      left: `${-this.positions.movementX}px`,
      top: `${this.size.height - this.positions.movementY}px`
    } : null
  }
  /**
   * @description Implementation  of Logic for  resizing the userform
   * @function userFormResize
   * @param event it is type of  MouseEvent
   */
  userFormResize (event: MouseEvent): void {
    event.preventDefault()
    this.positions.movementX = this.positions.clientX - event.clientX
    this.positions.movementY = this.positions.clientY - event.clientY
    this.positions.clientX = event.clientX
    this.positions.clientY = event.clientY

    const decWidth: number =
      this.propControlData.properties.Width! - this.positions.movementX
    const decHeight: number =
      this.propControlData.properties.Height! - this.positions.movementY

    if (decWidth > 102 && decHeight > 29) {
      this.updateUserFormResize({
        x: decWidth,
        y: decHeight,
        handler: this.resizeDiv
      })
    }
  }

  /**
   * @description Implementation  of Logic for resize and drag the control and userform
   * @function elementDrag
   * @param event it is type of  MouseEvent
   */
  elementDrag (event: MouseEvent): void {
    event.preventDefault()
    this.positions.movementX = this.positions.clientX - event.clientX
    this.positions.movementY = this.positions.clientY - event.clientY
    const scale: number = (this.propControlData.properties.Zoom! * 1) / 100
    const scale1: number = (this.propControlData.properties.Zoom! * 10) / 100
    const grid: Array<number> = [scale1, scale1]
    const x: number =
      Math.round(this.positions.movementX / scale / grid[0]) * grid[0]
    const y: number =
      Math.round(this.positions.movementY / scale / grid[1]) * grid[1]

    const diffGridX: number = x - this.positions.movementX
    const diffGridY: number = y - this.positions.movementY

    this.positions.clientX = event.clientX - diffGridX
    this.positions.clientY = event.clientY - diffGridY
    if (this.currentMouseDownEvent && (x !== 0 || y !== 0)) {
      this.currentMouseDownEvent.customCallBack && this.currentMouseDownEvent.customCallBack()
    }
    this.updateResize({ x: x, y: y, handler: this.resizeDiv })
  }

  /**
   * @description To destory the document event(mouseup,mousemove) attached while drag and resize implementation
   * @function closeDragElement
   */
  closeDragElement (): void {
    EventBus.$emit('endMoveControl')
    this.isMainSelect = false
    this.positions.offsetX = 0
    this.positions.offsetY = 0
    document.onmouseup = null
    document.onmousemove = null
    const groupId = this.userformData[this.userFormId][this.controlId]
      .properties.GroupID
    if (groupId !== '') {
      this.createGroup(groupId!)
    }
  }

  /**
   * @description  To get main and sub selected control details by passing
   * userFormId  which is taken as props from parent Component
   * @function selectedControl
   *
   */
  get selectedControl () {
    return this.selectedControls[this.userFormId].selected
  }

  /**
   * @description style object to dynamically changing the styles of  main and sub selected control in case of multiple selection  based on propControlData
   * @function handlerStyle
   *
   */
  get handlerStyle () {
    return {
      backgroundColor:
        this.selectedControl[0] === this.controlId ? 'white' : 'black',
      borderColor:
        this.selectedControl[0] === this.controlId ? 'black' : 'white'
    }
  }

  /**
   * @description propControlData is abstract class providing implementation to resize UserForm by passing
   * userFormId  which is taken as props from parent Component
   * @function propControlData
   *
   */
  get propControlData (): controlData {
    return this.userformData[this.userFormId][this.userFormId]
  }

  get getMainSelected () {
    return (
      this.selectedControls[this.userFormId].selected.includes(this.controlId) ||
      (this.selectedControls[this.userFormId].container.includes(this.controlId) &&
      this.selectedControls[this.userFormId].selected.includes(this.controlId) !== (this.selectedControls[this.userFormId].container.includes(this.controlId))
      ))
  }

  get getContainerSelect () {
    return this.controlId === this.selectedControls[this.userFormId].container[0]
  }

  get getIsMoveTarget () {
    return this.getMainSelected
  }
}
</script>

<style  scoped>
.move-border {
  z-index: 9999;
  position: absolute;
  border: 1px rgb(59, 58, 58) dashed;
}
.m-top-b, .m-bottom-b{
  width: 100%;
}
.m-left-b, .m-right-b{
  height: 100%;
}
.resize-area {
  position: absolute;
  width: 100%;
  height: 100%;
}
.handle {
  box-sizing: border-box;
  position: absolute;
  width: 6px;
  height: 6px;

  background: white;
  border: 1px solid #333;
}
.handleActivate {
  box-sizing: border-box;
  position: absolute;
  width: 6px;
  height: 6px;

  background: black;
  border: 1px solid white;
}
.handle-tl {
  top: -5px;
  left: -5px;
  cursor: nw-resize;
}
.handle-tm {
  top: -5px;
  left: 50%;
  margin-left: -5px;
  cursor: n-resize;
}
.handle-tr {
  top: -5px;
  right: -5px;
  cursor: ne-resize;
}
.handle-ml {
  top: 50%;
  margin-top: -5px;
  left: -5px;
  cursor: w-resize;
}
.handle-mr {
  top: 50%;
  margin-top: -5px;
  right: -5px;
  cursor: e-resize;
}
.handle-bl {
  bottom: -5px;
  left: -5px;
  cursor: sw-resize;
}
.handle-bm {
  bottom: -5px;
  left: 50%;
  margin-left: -5px;
  cursor: s-resize;
}
.handle-br {
  bottom: -5px;
  right: -5px;
  cursor: se-resize;
}
.userFormHandle {
  box-sizing: border-box;
  position: absolute;
  width: 8px;
  height: 8px;
  background: white;
  border: 1px solid #333;
}
.userFormHandle-tl {
  top: -8px;
  left: -8px;
  cursor: nw-resize;
  background: black;
  border: 1px solid white;
  cursor: default;
}
.userFormHandle-tm {
  top: -9px;
  left: 50%;
  margin-left: -8px;
  cursor: n-resize;
  background: black;
  border: 1px solid white;
  cursor: default;
}
.userFormHandle-tr {
  top: -8px;
  right: -8px;
  cursor: ne-resize;
  background: black;
  border: 1px solid white;
  cursor: default;
}
.userFormHandle-ml {
  top: 50%;
  margin-top: -8px;
  left: -9px;
  background: black;
  border: 1px solid white;
  cursor: default;
}
.userFormHandle-mr {
  top: 50%;
  margin-top: -8px;
  right: -9px;
  cursor: e-resize;
}
.userFormHandle-bl {
  bottom: -8px;
  left: -8px;
  cursor: sw-resize;
  background: black;
  border: 1px solid white;
  cursor: default;
}
.userFormHandle-bm {
  bottom: -9px;
  left: 50%;
  margin-left: -8px;
  cursor: s-resize;
}
.userFormHandle-br {
  bottom: -8px;
  right: -8px;
  cursor: se-resize;
}
</style>
