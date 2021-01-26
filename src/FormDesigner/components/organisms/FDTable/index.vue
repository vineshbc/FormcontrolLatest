<template>
  <div>
    <div class="btn-group">
      <button class="button" default>Alphabetic</button>
      <button class="button">Categorized</button>
    </div>
    <div
      style="
        position: absolute;
        height: calc(100% - 59px);
        overflow-y: auto;
        overflow-x: hidden;
        width: 100%;
      "
    >
      <FDTableItems
        v-for="(value, propertyName, index) in tableData"
        :key="index"
        :controlPropertyData="value"
        :propertyName="propertyName"
        @updateAppearance="updateAppearance"
        @emitFont="emitFont"
        @colorPaletteProp="colorPaletteProp"
      />
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue, Emit } from 'vue-property-decorator'
import { State, Action } from 'vuex-class'
import FDTableItems from '../../molecules/FDTableItems/index.vue'
import { IselectControl, IsetChildControls, IupdateControl, IupdateControlExtraData, IupdatedSelectedControl } from '@/storeModules/fd/actions'
import { EventBus } from '@/FormDesigner/event-bus'
import { checkPropertyValue } from '@/storeModules/fd/checkValidation'

interface IPropControl {
  propertyName: keyof controlProperties
  value : any
}

@Component({
  name: 'FDTable',
  components: {
    FDTableItems
  }
})
export default class FDTable extends Vue {
  @Prop({ type: Object, required: true }) tableData!: tableDatas;
  @State((state) => state.fd.userformData) userformData!: userformData;
  @Prop({ required: true, type: String }) public readonly userFormId! : string
  @Prop({ required: true }) public readonly getSelectedControlsDatas: any
  @Action('fd/updateControl') updateControl!: (payload: IupdateControl) => void;
  @Action('fd/updateControlExtraData') updateControlExtraData!: (
    payload: IupdateControlExtraData
  ) => void;
  @Action('fd/setChildControls') setChildControls!: (
    payload: IsetChildControls
  ) => void;
  @State((state) => state.fd.selectedControls) selectedControls!: fdState['selectedControls'];

  emitUpdateProperty (
    propertyName: keyof controlProperties,
    value: any
  ) {
    this.updateProperty({ propertyName: propertyName, value: value })
  }

  emitUpdateExtraProperty (
    propertyName: keyof controlProperties,
    value: any
  ) {
    this.updateProperty({ propertyName: propertyName, value: value })
  }
  updatePageIndex (propValue: number) {
    const userData = this.userformData[this.userFormId]
    const selected = this.selectedControls[this.userFormId].selected[0]
    const swapTabIndex = userData[selected].properties.Index!
    const container = this.selectedControls[this.userFormId].container[0]

    if (propValue < userData[container].controls.length) {
      const index = userData[container].controls.findIndex(val => userData[val].properties.Index === propValue)
      this.updatePropValue(userData[container].controls[index], 'Index', swapTabIndex)
      this.updatePropValue(selected, 'Index', propValue)
      const controls = userData[container].controls
      controls.sort((a, b) => {
        return userData[a].properties.Index! - userData[b].properties.Index!
      })
      this.setChildControls({
        userFormId: this.userFormId,
        containerId: container,
        targetControls: controls
      })
      return true
    } else {
      return false
    }
  }
  getLowestIndex (tempControls: string[], controlLength: number, propertyType: boolean) {
    let lastControlId = controlLength
    const userData = this.userformData[this.userFormId]
    for (let i = 0; i < tempControls.length; i++) {
      const propetyName = propertyType ? userData[tempControls[i]].extraDatas!.zIndex! : userData[tempControls[i]].extraDatas!.TabIndex!
      if (propetyName !== -1) {
        const IdNum = propetyName
        if (!isNaN(IdNum) && lastControlId > IdNum) {
          lastControlId = IdNum
        }
      }
    }
    return lastControlId
  }
  swapTabIndex (tempZIndex: number) {
    const userData = this.userformData[this.userFormId]
    const container = this.selectedControls[this.userFormId].container[0]
    const selected = this.selectedControls[this.userFormId].selected[0]
    const swapTabIndex = userData[selected].properties!.TabIndex!
    if (tempZIndex <= userData[container].controls.length - 1 && tempZIndex > -1) {
      const index = userData[container].controls.findIndex(
        (val) => userData[val].properties!.TabIndex === tempZIndex
      )
      this.updatePropValue(userData[container].controls[index], 'TabIndex', swapTabIndex)
      this.updatePropValue(selected, 'TabIndex', tempZIndex)
    }
  }
  updateTabIndexValue (val: number) {
    const userData = this.userformData[this.userFormId]
    const container = this.selectedControls[this.userFormId].container[0]
    const selected = this.selectedControls[this.userFormId].selected[0]
    const containerControls = this.userformData[this.userFormId][container].controls
    const controlType = userData[selected].type
    if (controlType !== 'FDImage') {
      const tempControls = []
      for (const index in containerControls) {
        const cntrlData = this.userformData[this.userFormId][containerControls[index]]
        if (cntrlData.type === 'FDImage') {
          tempControls.push(containerControls[index])
        }
      }
      const lastControlId = tempControls.length > 0 ? this.getLowestIndex(tempControls, userData[container].controls.length - 1, false)
        : this.userformData[this.userFormId][container].controls.length - 1
      if (val <= lastControlId) {
        this.swapTabIndex(val)
        return true
      } else {
        return false
      }
    }
  }

  updateExtraProperty (arg: IupdateControlExtraData) {
    const selected = this.getSelectedControlsDatas!
    for (let i = 0; i < selected.length; i++) {
      this.updateControlExtraData({
        userFormId: this.userFormId,
        controlId: selected[i],
        propertyName: arg.propertyName,
        value: arg.value
      })
    }
  }
  updateProperty (arg: IPropControl) {
    const selected = this.getSelectedControlsDatas!
    for (let i = 0; i < selected.length; i++) {
      if (arg.propertyName === 'TabIndex') {
        return this.updateTabIndexValue(parseInt(arg.value))
      } else if (arg.propertyName === 'Index') {
        return this.updatePageIndex(parseInt(arg.value))
      } else if (arg.propertyName === 'Name') {
        return this.updateName(selected[i], arg.value)
      } else if (arg.propertyName === 'Value' && (this.userformData[this.userFormId][selected[i]].type === 'SpinButton' || this.userformData[this.userFormId][selected[i]].type === 'ScrollBar')) {
        return this.updateSpinButtonScrollBarValueProp(selected[i], arg.value)
      } else if (arg.propertyName === 'Value' && (this.userformData[this.userFormId][selected[i]].type === 'OptionButton') &&
      (this.userformData[this.userFormId][selected[i]].properties.GroupName !== '') && arg.value === 'True') {
        const groupName = this.userformData[this.userFormId][selected[i]].properties.GroupName!
        this.updateValueForCommonGroupName(selected[i], arg.value, groupName)
      } else if (arg.propertyName === 'Min' || arg.propertyName === 'Max') {
        this.updatePropValue(selected[i], arg.propertyName, arg.value)
        this.validateValueProperty(selected[i], arg.value)
      } else if (arg.propertyName === 'Cancel' || arg.propertyName === 'Default') {
        this.validateCancelDefaultProp(selected[i], arg.propertyName, arg.value)
      } else {
        this.updatePropValue(selected[i], arg.propertyName, arg.value)
      }
    }
  }
  updatePropValue (id: string, propName: keyof controlProperties, propValue: any) {
    this.updateControl({
      userFormId: this.userFormId,
      controlId: id,
      propertyName: propName,
      value: propValue
    })
  }
  updateValueForCommonGroupName (id: string, value: string, groupName: string) {
    const userData = Object.keys(this.userformData[this.userFormId])
    this.updatePropValue(id, 'Value', 'True')
    for (let i = 0; i < userData.length; i++) {
      const controlData = this.userformData[this.userFormId][userData[i]]
      if (controlData.type === 'OptionButton' && controlData.properties.GroupName === groupName && controlData.properties.ID !== id) {
        this.updatePropValue(controlData.properties.ID, 'Value', 'False')
      }
    }
  }
  validateValueProperty (id: string, value: number) {
    let actualValue = this.userformData[this.userFormId][id].properties.Value! as number
    if (!this.checkValueRangeMinMax(id, actualValue)) {
      this.updatePropValue(id, 'Value', value)
    }
  }
  updateSpinButtonScrollBarValueProp (id: string, value: number) {
    if (this.checkValueRangeMinMax(id, value)) {
      this.updatePropValue(id, 'Value', value)
      return true
    }
    return false
  }
  validateCancelDefaultProp (id: string, propName: keyof controlProperties, value: boolean) {
    const userFormControls = Object.keys(this.userformData[this.userFormId])
    for (let controlId of userFormControls) {
      if (this.userformData[this.userFormId][controlId].type === 'CommandButton') {
        if (controlId === id) {
          this.updatePropValue(id, propName, value)
        } else {
          this.updatePropValue(controlId, propName, false)
        }
      }
    }
  }
  checkValueRangeMinMax (id: string, value: number) {
    let maxValue = this.userformData[this.userFormId][id].properties.Max!
    let minValue = this.userformData[this.userFormId][id].properties.Min!
    let min = Math.min(minValue, maxValue)
    let max = Math.max(minValue, maxValue)
    if (value >= min && value <= max) {
      return true
    } else {
      return false
    }
  }
  updateName (id: string, value: string) {
    let userData
    let isUnique
    if (this.userformData[this.userFormId][id].type === 'Userform') {
      userData = Object.keys(this.userformData)
      isUnique = userData.findIndex(val => this.userformData[val][val].properties.Name === value && val !== id)
    } else {
      userData = Object.keys(this.userformData[this.userFormId])
      isUnique = userData.findIndex(val => this.userformData[this.userFormId][val].properties.Name === value && val !== id)
    }
    if (isUnique === -1 && value !== '') {
      this.updatePropValue(id, 'Name', value)
      return true
    } else {
      return false
    }
  }
  validateValuePropertyChboxOpbtnTglbtn (propertyName : keyof controlProperties, propertyValue : string) : string {
    let resultValue = ''
    if (!isNaN(parseInt(propertyValue))) {
      if (parseInt(propertyValue) === 0) {
        this.emitUpdateProperty(propertyName, 'False')
        resultValue = 'False'
      } else {
        this.emitUpdateProperty(propertyName, 'True')
        resultValue = 'True'
      }
    } else if (propertyValue.toLowerCase() === 'true') {
      this.emitUpdateProperty(propertyName, 'True')
      resultValue = 'True'
    } else if (propertyValue.toLowerCase() === 'false') {
      this.emitUpdateProperty(propertyName, 'False')
      resultValue = 'False'
    } else {
      this.emitUpdateProperty(propertyName, '')
      resultValue = ''
    }
    return resultValue
  }
  validateControlSourceProperty (propertyValue : string) {
    // should validate the propertyValue
    return true
  }
  validateRowSourceProperty (propertyValue : string) {
    // should validate the propertyValue
    return true
  }
  validateTextProperty (propertyValue : string) {
    const controlData = this.userformData[this.userFormId][this.getSelectedControlsDatas[0]]
    const rowSourceData = controlData.extraDatas!.RowSourceData!
    for (let i = 0; i < rowSourceData.length; i++) {
      if (propertyValue === rowSourceData[i][0]) {
        return true
      }
    }
    return false
  }
  updateAppearance (e: Event) {
    const propertyName: keyof controlProperties = (e.target as HTMLInputElement).name as keyof controlProperties
    const inputType = this.tableData[propertyName]!.type
    let propertyValue = (e.target as HTMLInputElement).value
    if (inputType === 'file') {
      if (propertyName === 'MouseIcon' || propertyName === 'Picture') {
        this.handleConvertionImageToBase64(e)
      }
    } else if (inputType === 'String') {
      if (propertyName === 'Accelerator' || propertyName === 'PasswordChar') {
        this.emitUpdateProperty(
          propertyName,
          propertyValue !== '' ? propertyValue[0] : propertyValue
        );
        (e.target as HTMLInputElement).value = propertyValue && propertyValue[0]
      } else if (propertyName === 'Name') {
        if (checkPropertyValue(propertyName, propertyValue)) {
          const isSuccess = this.updateProperty({ propertyName: propertyName, value: propertyValue })
          if (!isSuccess) {
            (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
            EventBus.$emit('showErrorPopup', true, 'invalid', 'Could not set the Name property. Ambiguous name.')
          }
        } else {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
          EventBus.$emit('showErrorPopup', true, 'invalid', `Not a legal object name: '${propertyValue}'`)
        }
      } else if (propertyName === 'Value') {
        const controlType = this.userformData[this.userFormId][this.getSelectedControlsDatas[0]].type
        if (controlType === 'CheckBox' || controlType === 'OptionButton' || controlType === 'ToggleButton') {
          const resultValue = this.validateValuePropertyChboxOpbtnTglbtn(propertyName, propertyValue);
          (e.target as HTMLInputElement).value = resultValue
        } else {
          this.emitUpdateProperty(propertyName, propertyValue)
        }
      } else if (propertyName === 'Text') {
        const controlType = this.userformData[this.userFormId][this.getSelectedControlsDatas[0]].type
        if (controlType === 'ListBox') {
          const isTextValid = this.validateTextProperty(propertyValue)
          if (isTextValid) {
            this.emitUpdateProperty(propertyName, propertyValue)
          } else {
            EventBus.$emit('showErrorPopup', true, 'invalid', `Could not set the ${propertyName} property. Invalid property Value`);
            (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
          }
        } else {
          this.emitUpdateProperty(propertyName, propertyValue)
        }
      } else if (propertyName === 'ControlSource') {
        const isValid = this.validateControlSourceProperty(propertyValue)
        if (isValid) {
          this.emitUpdateProperty(propertyName, propertyValue)
        } else {
          EventBus.$emit('showErrorPopup', true, 'invalid', 'Invalid property value.')
        }
      } else if (propertyName === 'RowSource') {
        const isValid = this.validateRowSourceProperty(propertyValue)
        if (isValid) {
          this.emitUpdateProperty(propertyName, propertyValue)
        } else {
          EventBus.$emit('showErrorPopup', true, 'invalid', 'Invalid property value.')
        }
      } else {
        this.emitUpdateProperty(propertyName, propertyValue)
      }
    } else if (inputType === 'number' || inputType === 'float') {
      const value = propertyValue.includes('.') ? parseFloat(propertyValue) : parseInt(propertyValue)
      if (propertyName === 'Height' || propertyName === 'Width') {
        if (checkPropertyValue(propertyName, value)) {
          if ((this.userformData[this.userFormId][this.getSelectedControlsDatas[0]].type === 'Userform')) {
            if (propertyName === 'Width') {
              if (value >= 0 && value < 103) {
                this.emitUpdateProperty(propertyName, 103);
                (e.target as HTMLInputElement).value = '103'
              } else {
                this.emitUpdateProperty(propertyName, value)
              }
            } else if (propertyName === 'Height') {
              if (value >= 0 && value < 30) {
                this.emitUpdateProperty(propertyName, 30);
                (e.target as HTMLInputElement).value = '30'
              } else {
                this.emitUpdateProperty(propertyName, value)
              }
            }
          } else {
            this.emitUpdateProperty(propertyName, value)
          }
        } else {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
          if (value > 32767) {
            EventBus.$emit('showErrorPopup', true, 'overflow', 'Overflow')
          } else {
            EventBus.$emit('showErrorPopup', true, 'invalid', `Could not set the ${propertyName} property. Invalid property value. Enter a value between 0 and 32768`)
          }
        }
      } else if (propertyName === 'Top' || propertyName === 'Left') {
        if (checkPropertyValue(propertyName, value)) {
          this.emitUpdateProperty(propertyName, value)
        } else {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
          EventBus.$emit('showErrorPopup', true, 'invalid', `Could not set the ${propertyName} property. Invalid property value.`)
        }
      } else if (propertyName === 'MaxLength') {
        if (checkPropertyValue(propertyName, value)) {
          this.emitUpdateProperty(propertyName, value)
        } else {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
          EventBus.$emit('showErrorPopup', true, 'invalid', `Could not set the ${propertyName} property. Invalid property value. Enter a value between 0 and  2147483647`)
        }
      } else if (propertyName === 'BoundColumn') {
        if (checkPropertyValue(propertyName, value)) {
          this.emitUpdateProperty(propertyName, value)
        } else {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
          EventBus.$emit('showErrorPopup', true, 'invalid', `Could not set the ${propertyName} property. Invalid property value. Enter a value between 0 and 65536`)
        }
      } else if (propertyName === 'ColumnCount') {
        if (checkPropertyValue(propertyName, value)) {
          this.emitUpdateProperty(propertyName, value)
        } else {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
          EventBus.$emit('showErrorPopup', true, 'invalid', `Could not set the ${propertyName} property. Invalid property value. Enter a value between -1 and 2147483647`)
        }
      } else if (propertyName === 'ListRows') {
        if (checkPropertyValue(propertyName, value)) {
          this.emitUpdateProperty(propertyName, value)
        } else {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
          EventBus.$emit('showErrorPopup', true, 'invalid', `Could not set the ${propertyName} property. Invalid property value. Enter a value between 0 and 2147483647`)
        }
      } else if (propertyName === 'TextColumn') {
        if (checkPropertyValue(propertyName, value)) {
          this.emitUpdateProperty(propertyName, value)
        } else {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
          EventBus.$emit('showErrorPopup', true, 'invalid', `Could not set the ${propertyName} property. Invalid property value. Enter a value between -1 and 32768`)
        }
      } else if (propertyName === 'Value') {
        const controlData = this.userformData[this.userFormId][this.getSelectedControlsDatas[0]]
        const controlType = controlData.type
        if (controlType === 'TabStrip') {
          if (value < -1 || value >= controlData.extraDatas!.Tabs!.length!) {
            (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
            EventBus.$emit('showErrorPopup', true, 'invalid', `Could not set the ${propertyName} property. Invalid property value.`)
          } else {
            this.emitUpdateProperty(propertyName, value)
          }
        } else if (controlType === 'MultiPage') {
          if (value < -1 || value >= controlData.controls.length) {
            (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
            EventBus.$emit('showErrorPopup', true, 'invalid', `Could not set the ${propertyName} property. Invalid property value.`)
          } else {
            this.emitUpdateProperty(propertyName, value)
          }
        } else if (controlType === 'SpinButton' || controlType === 'ScrollBar') {
          if (this.isDecimalNumber(propertyValue)) {
            (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
          } else {
            const isSuccess = this.updateProperty({ propertyName: propertyName, value: value })
            if (!isSuccess) {
              (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
              EventBus.$emit('showErrorPopup', true, 'invalid', `Could not set the ${propertyName} property. Invalid property value.`)
            }
          }
        } else {
          this.emitUpdateProperty(propertyName, propertyValue)
        }
      } else if (propertyName === 'Min' || propertyName === 'Max' || propertyName === 'SmallChange') {
        if (this.isDecimalNumber(propertyValue)) {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
        } else if (checkPropertyValue(propertyName, value)) {
          if (propertyName === 'Min' || propertyName === 'Max') {
            this.updateProperty({ propertyName: propertyName, value: value })
          } else {
            this.emitUpdateProperty(propertyName, value)
          }
        } else {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
          EventBus.$emit('showErrorPopup', true, 'invalid', `Invalid property value.`)
        }
      } else if (propertyName === 'TabFixedHeight' || propertyName === 'TabFixedWidth') {
        if (checkPropertyValue(propertyName, value)) {
          this.emitUpdateProperty(propertyName, value)
        } else {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
          EventBus.$emit('showErrorPopup', true, 'invalid', `Could not set the ${propertyName} property. Invalid property value. Enter a value between 0 and 7200`)
        }
      } else if (propertyName === 'TabIndex') {
        if (this.isDecimalNumber(propertyValue)) {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
        } else if (checkPropertyValue(propertyName, value)) {
          const isSuccess = this.updateProperty({ propertyName: propertyName, value: value })
          if (!isSuccess) {
            (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
          }
        } else {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
          EventBus.$emit('showErrorPopup', true, 'invalid', `Could not set the ${propertyName} property. Invalid property value. Enter a value greater than or equal to zero.`)
        }
      } else if (propertyName === 'Index') {
        if (this.isDecimalNumber(propertyValue)) {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
        } else if (value < 0) {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
          EventBus.$emit('showErrorPopup', true, 'invalid', `Could not set the ${propertyName} property. Invalid property value. Enter a value between 0 and 32767`)
        } else {
          const isSuccess = this.updateProperty({ propertyName: propertyName, value: propertyValue })
          if (!isSuccess) {
            (e.target as HTMLInputElement).value = this.tableData!.Index!.value! as string
            EventBus.$emit('showErrorPopup', true, 'invalid', `Could not set the ${propertyName} property. Invalid property value.`)
          }
        }
      } else if (propertyName === 'Delay') {
        if (this.isDecimalNumber(propertyValue)) {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
        } else if (checkPropertyValue(propertyName, propertyValue)) {
          this.emitUpdateProperty(propertyName, value)
        } else if (value < 0) {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
          EventBus.$emit('showErrorPopup', true, 'invalid', `Could not set the ${propertyName} property. Invalid property value. Enter a value greater than or equal to zero.`)
        } else {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
          EventBus.$emit('showErrorPopup', true, 'invalid', `Invalid property value.`)
        }
      } else if (propertyName === 'Zoom') {
        if (this.isDecimalNumber(propertyValue)) {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
        } else if (checkPropertyValue(propertyName, propertyValue)) {
          this.emitUpdateProperty(propertyName, value)
        } else {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
          EventBus.$emit('showErrorPopup', true, 'invalid', `Could not set the ${propertyName} property. Invalid property value. Enter a value between 10 and 400.`)
        }
      } else if (propertyName === 'DrawBuffer') {
        if (this.isDecimalNumber(propertyValue)) {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
        } else if (checkPropertyValue(propertyName, propertyValue)) {
          this.emitUpdateProperty(propertyName, value)
        } else {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
          EventBus.$emit('showErrorPopup', true, 'invalid', `Could not set the ${propertyName} property. Invalid property value. Enter a value between 16000 and 1048576.`)
        }
      } else if (propertyName === 'TransitionPeriod') {
        if (this.isDecimalNumber(propertyValue)) {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
        } else if (checkPropertyValue(propertyName, propertyValue)) {
          this.emitUpdateProperty(propertyName, value)
        } else {
          (e.target as HTMLInputElement).value = this.tableData![propertyName]!.value! as string
          EventBus.$emit('showErrorPopup', true, 'invalid', `Could not set the ${propertyName} property. Invalid property value. Enter a value between 0 and 10000.`)
        }
      } else {
        this.emitUpdateProperty(propertyName, value)
      }
    } else if (inputType === 'Boolean') {
      if ((propertyName === 'Cancel' || propertyName === 'Default') && ((e.target as HTMLInputElement).value) === 'true') {
        this.updateProperty({ propertyName: propertyName, value: true })
      } else {
        this.emitUpdateProperty(
          propertyName,
          (e.target as HTMLInputElement).value === 'true'
        )
      }
    } else if (inputType === 'array') {
      if ((e.target as HTMLInputElement).value !== '') {
        if (propertyName === 'BorderStyle' && propertyValue === '1') {
          this.emitUpdateProperty(
            propertyName,
            parseInt((e.target as HTMLInputElement).value)
          )
          this.emitUpdateProperty(
            'SpecialEffect',
            0
          )
        } else if (propertyName === 'SpecialEffect' && parseInt(propertyValue) > 0) {
          this.emitUpdateProperty(
            propertyName,
            parseInt((e.target as HTMLInputElement).value)
          )
          this.emitUpdateProperty(
            'BorderStyle',
            0
          )
        } else {
          this.emitUpdateProperty(
            propertyName,
            parseInt((e.target as HTMLInputElement).value)
          )
        }
      }
    }
  }

  isDecimalNumber (propValue : string) {
    if (propValue.indexOf('.') > -1) {
      EventBus.$emit('showErrorPopup', true, 'invalid', `Invalid property value.`)
      return true
    }
    return false
  }
  handleConvertionImageToBase64 (e: Event) {
    let that = this
    const reader = new FileReader()
    const fileInput = (e.target as HTMLInputElement)
    const filePath = fileInput.value
    let allowedExtensions
    // Allowed file type
    if (fileInput.name === 'Picture') {
      allowedExtensions = /(\.jpg|\.jpeg|\.bmp|\.ico|\.gif)$/i
    } else {
      allowedExtensions = /(\.bmp|\.ico|\.gif)$/i
    }
    if (!allowedExtensions.exec(filePath)) {
      fileInput.value = ''
      EventBus.$emit('showErrorPopup', true, 'invalid', 'Invalid Picture')
      return false
    } else {
      if (fileInput.files && fileInput.files[0]) {
        reader.onload = function (e) {
          that.emitUpdateProperty(fileInput.name as keyof controlProperties, reader.result)
        }
        reader.readAsDataURL(fileInput.files[0])
      }
    }
  }

  emitFont (tempVal: font) {
    this.emitUpdateProperty('Font', tempVal)
  }
  colorPaletteProp (selectedValue: ISelectedColoPaletteValue) {
    this.emitUpdateProperty(selectedValue.propertyName as keyof controlProperties, selectedValue.propertyValue)
  }
}
</script>

<style scoped>
.node {
  text-align: left;
}
table {
  margin-right: 0%;
  width: 100%;
  border: 1px solid black;
  border-collapse: collapse;
  color: black;
}
th,
td {
  text-align: left;
  font-size: 11px;
  margin-right: 0%;
  width: fit-content;
  border: 1px solid black;
  border-collapse: collapse;
  color: black;
}
th {
  width: 120%;
  font-size: 12px;
}
.form-control {
  float: left;
  width: 280px;
  cursor: pointer;
  background-color: rgb(190, 185, 185);
}
label {
  float: left;
}
select {
  width: 100%;
  padding: 1px 2px;
}
input {
  width: 100%;
}
.btn-group .button {
  border: none;
  color: black;
  text-align: center;
  text-decoration: none;
  display: inline-block;
  font-size: 12px;
  cursor: pointer;
}
.lefttd {
  width: 120px;
}
</style>
