<template>
  <div
    class="listStyle"
    :style="listStyleObj"
    :title="properties.ControlTipText"
    @click="listBoxClick"
    @mousedown="controlEditMode"
    :tabindex="properties.TabIndex"
    @keydown="forMatchEntry"
    v-on="eventStoppers()"
    @keydown.esc="setContentEditable($event, false)"
  >
    <div class="table-style" :style="tableStyleObj" ref="listBoxTableRef" v-if="properties.RowSource !== ''">
      <div v-if="properties.ColumnHeads === true" class="theadClass">
        <div class="thead">
          <div
            :style="tdStyleObj"
            v-if="properties.ListStyle === 1"
            class="tdClass"
          ></div>
          <template v-for="(a, columnIndex) in extraDatas.ColumnHeadsValues">
            <div
              v-if="
                columnIndex < properties.ColumnCount ||
                properties.ColumnCount === -1
              "
              :key="columnIndex"
              :style="updateColHeads(columnIndex)"
            >
              {{ a }}
            </div>
          </template>
        </div>
      </div>
      <!-- <div v-else></div> -->
      <div class="table-body">
        <div
          :tabindex="index"
          class="tr"
          ref="listStyleRef"
          v-for="(item, index) of extraDatas.RowSourceData"
          :key="index"
          @mouseenter.stop="handleDrag"
          @keydown.esc="releaseEditMode"
          @blur.stop="clearMatchEntry"
          @keydown.stop="handleExtendArrowKeySelect"
          @mousedown="
            isRunMode || isEditMode ? handleMousedown($event) : stopMousedown($event)
          "
        >
          <div
            :style="tdStyleObj"
            class="tdClassIn"
            v-if="
              (properties.ListStyle === 1 && properties.ColumnCount > 0) ||
              (properties.ListStyle === 1 && properties.ColumnCount === -1)
            "
          >
            <input
              :name="properties.MultiSelect === 2 ? 'checkbox' : 'radio'"
              :type="
                properties.MultiSelect === 1 || properties.MultiSelect === 2
                  ? 'checkbox'
                  : 'radio'
              "
              @click.prevent.stop
              class="inputClass"
            />
          </div>
          <div
            class="column-item"
            v-for="(i, index) in item"
            :key="index"
            :style="[updateColumnWidths(index),styleColumnObj]"
          >
            <template
              v-if="
                index < properties.ColumnCount || properties.ColumnCount === -1
              "
              >{{ i }}</template
            >
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import {
  Component,
  Prop,
  Model,
  Emit,
  Mixins,
  Watch,
  Ref
} from 'vue-property-decorator'
import FdControlVue from '@/api/abstract/FormDesigner/FdControlVue'
import { EventBus } from '@/FormDesigner/event-bus'

@Component({
  name: 'FDListBox'
})
export default class FDListBox extends Mixins(FdControlVue) {
  @Ref('listStyleRef') listStyleRef: HTMLDivElement[];
  @Ref('listBoxTableRef') listBoxTableRef!: HTMLDivElement;
  @Prop() isActivated: boolean;
  @Prop() toolBoxSelect: string
  checkedvalue: boolean;
  $el: HTMLDivElement;
  handleMousedown (e: MouseEvent) {
    if (this.toolBoxSelect === 'Select') {
      e.stopPropagation()
      if (this.properties.RowSource !== '') {
        if (
          e.target instanceof HTMLDivElement ||
      e.target instanceof HTMLTableRowElement ||
      e.target instanceof HTMLInputElement
        ) {
          this.tempListBoxComboBoxEvent = e
          const targetElement = e.target
          const tempData = targetElement!.parentElement!
            .children[0] as HTMLDivElement
          const tempDataOption = targetElement.parentElement!
            .children[1] as HTMLDivElement
          const tempPath = e.composedPath()
          targetElement.focus()
          let data = targetElement.innerText
          let splitData = data.replace(/\t/g, ' ').split(' ')

          targetElement.focus()
          if (this.properties.Enabled && this.properties.Locked === false) {
            if (this.properties.MultiSelect === 0) {
              this.clearOptionBGColorAndChecked(e)
              this.setOptionBGColorAndChecked(e)
            } else if (this.properties.MultiSelect === 1) {
              this.setOptionBGColorAndChecked(e)
            } else if (this.properties.MultiSelect === 2) {
              if (e.ctrlKey === true) {
                if (targetElement.tagName === 'INPUT') {
                  this.setOptionBGColorAndChecked(e)
                } else {
                  this.setOptionBGColorAndChecked(e)
                }
              } else if (e.shiftKey === true && this.properties.Value !== '') {
                let startPoint = 0
                let endPoint = 0
                for (let i = 0; i < tempPath.length; i++) {
                  const ele = tempPath[i] as HTMLDivElement
                  if (ele.className === 'table-body') {
                    // extend points start and end
                    for (let j = 0; j < ele.childNodes.length; j++) {
                      const cd = ele.childNodes[j] as HTMLDivElement
                      if (cd.innerText === this.properties.Value) {
                        startPoint = j + 1
                      }
                      if (cd.innerText === targetElement.innerText) {
                        endPoint = j
                      }
                    }
                    // upward selection start and end swap
                    if (startPoint > endPoint) {
                      let temp = startPoint
                      startPoint = endPoint
                      endPoint = temp
                    }
                    // setting selection
                    for (let k = startPoint; k <= endPoint; k++) {
                      const node = ele.childNodes[k] as HTMLDivElement
                      const tempNode = node.childNodes[0]
                        .childNodes[0] as HTMLInputElement
                      node.style.backgroundColor = 'rgb(59, 122, 231)'
                      if (this.properties.ListStyle === 1 && !tempNode.checked) {
                        // tempNode.checked = !tempNode.checked
                        tempNode.checked = true
                      }
                    }
                    break
                  }
                }
              }

              if (this.properties.ControlSource !== '') {
                this.updateDataModel({
                  propertyName: 'Text',
                  value: this.selectionData[0]
                })
                this.updateDataModel({
                  propertyName: 'Value',
                  value: this.selectionData[0]
                })
              }
              this.clearOptionBGColorAndChecked(e)
              this.setOptionBGColorAndChecked(e)
            }
          }
          if (this.properties.MultiSelect === 0) {
            for (let i = 0; i < this.extraDatas.RowSourceData!.length; i++) {
              if (this.listStyleRef[i].style.backgroundColor !== '') {
                const text = this.extraDatas.RowSourceData![i][0]
                this.updateDataModel({ propertyName: 'Text', value: text })
                const x = this.extraDatas.RowSourceData![i][this.properties.BoundColumn! - 1]
                this.updateDataModel({ propertyName: 'Value', value: x })
              }
            }
          } else {
            this.updateDataModel({ propertyName: 'Text', value: '' })
            this.updateDataModel({ propertyName: 'Value', value: '' })
          }
          this.getSelectedStyle()
        }
      }
    }
  }
  clearMatchEntry () {
    this.updateDataModelExtraData({ propertyName: 'MatchData', value: '' })
  }

  getSelectedStyle () {
    if (this.listStyleRef) {
      for (let i = 0; i < this.listStyleRef.length; i++) {
        if (this.listStyleRef[i].style.backgroundColor === 'rgb(59, 122, 231)') {
          for (let j = 0; j < this.listStyleRef[i].children.length; j++) {
            const a = this.listStyleRef[i].children[j] as HTMLDivElement
            a.style.backgroundColor = 'rgb(59, 122, 231)'
          }
        } else if (this.listStyleRef[i].style.backgroundColor === '') {
          for (let j = 0; j < this.listStyleRef[i].children.length; j++) {
            const a = this.listStyleRef[i].children[j] as HTMLDivElement
            a.style.backgroundColor = ''
          }
        }
      }
    }
  }
  /**
   * @description style object is passed to :style attribute in div tag
   * dynamically changing the styles of the component based on properties
   * @function listStyleObj
   */
  protected get listStyleObj (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    let display = ''
    if (this.isRunMode) {
      display = controlProp.Visible ? 'inline-block' : 'none'
    } else {
      display = 'inline-block'
    }
    return {
      backgroundColor: controlProp.BackColor,
      borderColor: controlProp.BorderStyle === 1 ? controlProp.BorderColor : '',
      cursor:
        controlProp.MousePointer !== 0 || controlProp.MouseIcon !== ''
          ? this.getMouseCursorData
          : 'default',
      borderLeft:
        controlProp.BorderStyle === 1
          ? '1px solid ' + controlProp.BorderColor
          : controlProp.SpecialEffect === 2
            ? '2px solid gray'
            : controlProp.SpecialEffect === 3
              ? '1.5px solid gray'
              : controlProp.SpecialEffect === 4
                ? '0.5px solid gray'
                : '',
      borderRight:
        controlProp.BorderStyle === 1
          ? '1px solid ' + controlProp.BorderColor
          : controlProp.SpecialEffect === 1
            ? '2px solid gray'
            : controlProp.SpecialEffect === 4
              ? '1.5px solid gray'
              : controlProp.SpecialEffect === 3
                ? '0.5px solid gray'
                : '',
      borderTop:
        controlProp.BorderStyle === 1
          ? '1px solid ' + controlProp.BorderColor
          : controlProp.SpecialEffect === 2
            ? '2px solid gray'
            : controlProp.SpecialEffect === 3
              ? '1.5px solid gray'
              : controlProp.SpecialEffect === 4
                ? '0.5px solid gray'
                : '',
      borderBottom:
        controlProp.BorderStyle === 1
          ? '1px solid ' + controlProp.BorderColor
          : controlProp.SpecialEffect === 1
            ? '2px solid gray'
            : controlProp.SpecialEffect === 4
              ? '1.5px solid gray'
              : controlProp.SpecialEffect === 3
                ? '0.5px solid gray'
                : '',
      left: `${controlProp.Left}px`,
      width: `${controlProp.Width}px`,
      height: `${controlProp.Height}px`,
      top: `${controlProp.Top}px`,
      display: display
    }
  }

  get styleColumnObj () {
    const controlProp = this.properties
    const font: font = controlProp.Font
      ? controlProp.Font
      : {
        FontName: 'Arial',
        FontSize: 10,
        FontItalic: true,
        FontBold: true,
        FontUnderline: true,
        FontStrikethrough: true
      }
    return {
      fontFamily: font.FontStyle! !== '' ? this.setFontStyle : font.FontName!,
      fontSize: `${font.FontSize}px`,
      fontStyle: font.FontItalic || this.isItalic ? 'italic' : '',
      textDecoration:
        font.FontStrikethrough === true && font.FontUnderline === true
          ? 'underline line-through'
          : font.FontUnderline
            ? 'underline'
            : font.FontStrikethrough
              ? 'line-through'
              : '',
      textUnderlinePosition: 'under',
      fontWeight: font.FontBold
        ? 'bold'
        : font.FontStyle !== ''
          ? this.tempWeight
          : '',
      fontStretch: font.FontStyle !== '' ? this.tempStretch : ''

    }
  }
  /**
   * @description style object is passed to :style attribute in table tag
   * dynamically changing the styles of the component based on properties
   * @function tableStyleObj
   */
  protected get tableStyleObj (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    const font: font = controlProp.Font
      ? controlProp.Font
      : {
        FontName: 'Arial',
        FontSize: 10,
        FontItalic: true,
        FontBold: true,
        FontUnderline: true,
        FontStrikethrough: true
      }
    return {
      borderCollapse: 'collapse',
      tableLayout: 'fixed',
      color: controlProp.ForeColor,
      fontFamily: font.FontStyle! !== '' ? this.setFontStyle : font.FontName!,
      fontSize: `${font.FontSize}px`,
      fontStyle: font.FontItalic || this.isItalic ? 'italic' : '',
      textDecoration:
        font.FontStrikethrough === true && font.FontUnderline === true
          ? 'underline line-through'
          : font.FontUnderline
            ? 'underline'
            : font.FontStrikethrough
              ? 'line-through'
              : '',
      textUnderlinePosition: 'under',
      fontWeight: font.FontBold
        ? 'bold'
        : font.FontStyle !== ''
          ? this.tempWeight
          : '',
      fontStretch: font.FontStyle !== '' ? this.tempStretch : '',
      width:
        controlProp.ColumnWidths === ''
          ? '100%'
          : `${controlProp.Width}px` +
            parseInt(controlProp.ColumnWidths!) +
            'px',
      display: this.properties.ColumnCount === 0 ? 'none' : ''
    }
  }

  /**
   * @description style object is passed to :style attribute in td tag
   * dynamically changing the styles of the component based on properties
   * @function tdStyleObj
   */
  protected get tdStyleObj (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    return {
      textAlign:
        controlProp.TextAlign === 0
          ? 'left'
          : controlProp.TextAlign === 2
            ? 'right'
            : 'center'
    }
  }

  /**
   * @description watches changes in properties for Value
   * @function ValueData
   * @param oldVal previous properties data
   * @param newVal  new/changed properties data
   */
  @Watch('properties.Value', { deep: true })
  ValueData (newVal: string, oldVal: string) {
    if (this.properties.RowSource !== '') {
      if (newVal !== '' && this.properties.MultiSelect === 0) {
        if (
        this.properties.BoundColumn! > 0 &&
        this.properties.BoundColumn! < this.extraDatas.RowSourceData!.length
        ) {
          let tempData = [...this.extraDatas.RowSourceData!]
          let tempBoundColumn = this.properties.BoundColumn! - 1
          for (let i = 0; i < this.extraDatas.RowSourceData!.length; i++) {
            if (tempData[i][tempBoundColumn] === newVal) {
              this.clearOptionBGColorAndChecked(this.tempListBoxComboBoxEvent)
              this.listStyleRef[i].style.backgroundColor = 'rgb(59, 122, 231)'
              this.setOptionBGColorAndChecked(
              this.tempListBoxComboBoxEvent as MouseEvent
              )
            }
          }
          if (tempData![0][this.properties.BoundColumn! - 1] === newVal) {
            this.updateDataModel({ propertyName: 'Value', value: newVal })
          }
        } else {
          return undefined
        }
      } else {
        this.updateDataModel({ propertyName: 'Value', value: '' })
      }
    } else {
      this.updateDataModel({ propertyName: 'Value', value: '' })
    }
  }

  /**
   * @description mounted initializes the values which are required for the component
   */
  mounted () {
    this.$el.focus()
    var event = new MouseEvent('mousedown.stop')
    if (this.properties.RowSource !== '') {
      const initialRowSourceData = this.extraDatas.RowSourceData!
      this.updateDataModel({ propertyName: 'ControlSource', value: '' })
      if (initialRowSourceData && initialRowSourceData.length === 0) {
        this.updateDataModel({ propertyName: 'TopIndex', value: -1 })
      } else {
        this.updateDataModel({ propertyName: 'TopIndex', value: 0 })
      }
    }
  }

  /**
   * @description watches changes in properties for MultiSelect
   * @function multiSelectCheck
   * @param oldVal previous properties data
   * @param newVal  new/changed properties data
   */
  @Watch('properties.MultiSelect', { deep: true })
  multiSelectCheck (newVal: number, oldVal: number) {
    if (newVal === 1 || newVal === 2) {
      this.updateDataModel({ propertyName: 'Value', value: '' })
      this.updateDataModel({ propertyName: 'Text', value: '' })
    }
    if (this.tempListBoxComboBoxEvent) {
      this.clearOptionBGColorAndChecked(this.tempListBoxComboBoxEvent)
    }
  }

  /**
   * @description watches changes in properties for ListStyle
   * @function listCheck
   * @param oldVal previous properties data
   * @param newVal  new/changed properties data
   */
  @Watch('properties.ListStyle', { deep: true })
  listCheck (newVal: number, oldVal: number) {
    this.clearOptionBGColorAndChecked(this.tempListBoxComboBoxEvent)
  }

  @Watch('isEditMode', { deep: true })
  isEditCheck (newVal: boolean, oldVal: boolean) {
    if (this.isEditMode) {
      this.$el.click()
    }
  }

  forMatchEntry (event: KeyboardEvent) {
    if (event.key === 'Enter' && event.keyCode === 13) {
      this.setContentEditable(event, true)
    }
    if (this.isEditMode) {
      this.listStyleRef[0].click()
    }
    if (event.key === 'Escape' && event.keyCode === 27) {
      this.releaseEditMode(event)
    }
    if (event.key === 'Delete') {
      this.deleteItem(event)
    }
  }

  @Watch('properties.RowSource')
  rowSourceValidate () {
    const initialRowSourceData = this.extraDatas.RowSourceData!
    this.updateDataModel({ propertyName: 'ControlSource', value: '' })
    if (initialRowSourceData && initialRowSourceData.length === 0) {
      this.updateDataModel({ propertyName: 'TopIndex', value: -1 })
    } else {
      this.updateDataModel({ propertyName: 'TopIndex', value: 0 })
    }
  }

  @Watch('properties.ControlSource')
  controlSourceUpdate () {
    if (this.properties.ControlSource !== '') {
      for (let i = 0; i < this.extraDatas.RowSourceData!.length; i++) {
        this.listStyleRef[i].style.backgroundColor = ''
        if (
          this.extraDatas.RowSourceData![i][this.properties.BoundColumn! - 1] === this.extraDatas.ControlSourceValue!
        ) {
          const listRow = this.listStyleRef[i]
          listRow.style.backgroundColor = 'rgb(59, 122, 231)'
        }
      }
    }
  }
  releaseEditMode (event: KeyboardEvent) {
    this.$el.focus()
    this.setContentEditable(event, false)
  }
  eventStoppers () {
    const eventStop = (event: Event) => event.stopPropagation()
    return this.isEditMode === false
      ? null
      : {
        keydown: eventStop
      }
  }
  listBoxClick (e: MouseEvent) {
    if (this.toolBoxSelect === 'Select') {
      e.stopPropagation()
      this.selectedItem(e)
      if (!this.isActivated) {
        EventBus.$emit('focusUserForm')
      }
    }
  }
  stopMousedown (event: MouseEvent) {
    if (this.toolBoxSelect === 'Select') {
      event.stopPropagation()
    }
  }
}
</script>

<style scoped>
.listStyle {
  width: 200px;
  height: 200px;
  background-color: lightgray;
  border: 1px solid gray;
  overflow: auto;
  box-sizing: border-box;
  /* box-shadow: -1px -1px lightgray; */
}
.list-outer {
  border: 0.1px solid lightgray;
  width: 164px;
  height: 130px;
  overflow-y: auto;
  box-shadow: -1px -1px grey;
}
.tr {
  outline: none;
  display: flex;
}
.ul {
  display: grid;
  grid-template-columns: 100%;
  list-style-type: none;
  margin-block-start: 0em;
  margin-block-end: 0em;
  margin-inline-start: 0px;
  margin-inline-end: 0px;
  padding-inline-start: 0px;
}
.li {
  /* padding: 3px; */
  margin: 1px;
  text-align: left;
  font-size: 14px;
  height: 16px;
  border: 0.5px solid white;
  cursor: pointer;
}
.li:hover {
  margin: 1px;
  border-color: rgb(52, 52, 255);
  border-radius: 2px;
  background-color: rgb(59, 122, 231);
}
.fa {
  margin-left: 4px;
  margin-top: 3px;
}
.span {
  /* margin: 4px;  */
  margin-left: 7px;
  margin-top: 0px;
  font-family: Arial, Helvetica, sans-serif;
}
.div {
  display: grid;
  grid-template-columns: 1fr;
}
.listbox {
  display: grid;
  grid-template-columns: 100%;
  height: 100px;
  width: 300px;
  background-color: lightgray;
  border: 1px solid gray;
}
.columnHeads {
  height: 19.2px;
  border: 1px solid black;
  display: grid;
  grid-template-columns: 100%;
}
.optionsItems {
  display: grid;
  grid-template-columns: 100%;
}
.table-style {
  width: 100%;
}
.theadClass {
  border-bottom: 1px solid;
  white-space: nowrap;
}
.tdClass {
  width: 15px;
  border-right: 1px solid;
}
.tdClassIn {
  width: 40px;
}
.inputClass {
  margin: 0;
}
.thead {
  display: flex;
}
</style>
