<template>
  <Row type="flex" align="top" @mouseover.native="isMouseOver=true" @mouseout.native="isMouseOver=false" style="margin-bottom:10px;word-break:break-all;">
    <Col span="4" align="right" style="padding:0px 10px 0px 0px">
      <Tooltip :content="deletable ? 'Delete': 'Undeletable key'" :delay="500" placement="bottom-start">
        <Icon 
          type="md-remove-circle" 
          :class="buttonClass" 
          v-show="isMouseOver" 
          @click.native="deleteInfoKey" 
          style="padding:0px 10px"
        />
      </Tooltip>
      <span>{{infoKey}}</span>
    </Col>
    <Col span="20" style="padding:0px 0px 0px 10px">
      <span v-if="inputValueType === 'link'">
        <Input v-model="inputValue" type="textarea" :autosize="{ minRows: 1 }" size="small" style="width:calc(100% - 30px);"/>
        <Poptip
          placement="bottom"
          :width="getQrcodeImgWidth()"
          word-wrap
          @on-popper-show="loadQrcodeImg"
          v-model="isDisplayPoptip"
          transfer
        >
          <svg-icon class="ivu-icon" name="qrcode" scale="2.8" style="margin-left:3px;cursor: pointer;"/>
          <div slot="content">
            <img :src="imgData" style="width:100%">
          </div>
        </Poptip>
      </span>
      <span v-else-if="inputValueType === 'label'">
        <LabelDropdown :initLabels="infoValue" :placement="'bottom-start'" @onLabelChange="editLabel">
          <template #dropdownButton>
            <span v-for="(label, index) in infoValue">
              <span class="data-label" :style="'background-color:'+(label.color?label.color:'#808695')">{{label.name}}</span>
            </span>
            <Icon type="md-settings" size="14" style="padding-left:5px"/>
          </template>
        </LabelDropdown>
      </span>
      <span v-else-if="inputValueType === 'category'">
        <Select v-model="infoValue.selected" multiple size="small">
          <Option v-for="item in infoValue.allItem" :value="item.value" :key="item.value">{{ item.value }}</Option>
        </Select>
      </span>
      <span v-else-if="inputValueType === 'longList'">
        <div class="row no-gutters mb-1" v-for="listItem in longListValue">
          <div class="col-5">
            <span>{{listItem.id}}</span>
          </div>
          <div class="col-7 text-truncate">
            <span>{{listItem.name}}</span>
          </div>
        </div>
        <v-btn v-show="infoValue.length > longListShowLessCount" plain class="px-0" height="20" color="primary" @click="changeTextLongListShowAll">
          <span>{{isLongListShowAll ? 'Show Less': 'Show More'}}</span>
        </v-btn>
      </span>
      <span v-else-if="inputValueType === 'input'">
        <Input v-model="inputValue" type="textarea" :autosize="{ minRows: 1 }" size="small"/>
      </span>
      <span v-else-if="inputValueType === 'textWithCopy'">
        {{inputValue}}
        <v-btn icon x-small title='Copy'>
          <v-icon
            x-small
            color="context"
            v-clipboard:copy="inputValue"
            v-clipboard:success="onUrlCopy"
            v-clipboard:error="onUrlCopyError"
          >{{copyIcon}}</v-icon>
        </v-btn>
      </span>
      <span v-else>
        {{inputValue}}
      </span>
    </Col>
  </Row>
</template>

<script>
import svgIcon from 'vue-svg-icon/Icon.vue'
import { getQrcodeImg } from '@/api'
import LabelDropdown from '@/components/LabelDropdown.vue'

export default {
  props: ['infoKey', 'editable', 'deletable'],
  components: {
    svgIcon,
    LabelDropdown
  },
  data() {
    return {
      imgData: '',
      isDisplayPoptip: false,
      isMouseOver: false,
      isLongListShowAll: false,
      longListShowLessCount: 5,
      copyIcon: 'mdi-content-copy'
    }
  },
  computed: {
    inputValue: {
      get () {
        let infoValue = this.$store.state.dataManager.groupDetail[this.infoKey]
        if (infoValue === null) {
          return infoValue
        } else if (typeof infoValue === 'object') {
          return JSON.stringify(infoValue)
        } else {
          return infoValue
        }
      },
      set (val) {
        this.$store.commit('setGroupDetailItem', { key: this.infoKey, value: val })
      }
    },
    infoValue () {
      return this.$store.state.dataManager.groupDetail[this.infoKey]
    },
    longListValue () {
      if (this.isLongListShowAll) {
        return this.infoValue
      }
      return this.infoValue.slice(0, this.longListShowLessCount)
    },
    buttonClass () {
      if (this.deletable) {
        return ['enable-button']
      } else {
        return ['disable-button']
      }
    },
    inputValueType () {
      if (String(this.inputValue).match('(?=.*://)')) {
        if (this.$store.state.snapshot.groupDetailDisplayKey === this.infoKey ) {
          this.isDisplayPoptip = true
          this.$store.commit('clearGroupDetailDisplayKey')
          this.loadQrcodeImg ()
        }
        return 'link'
      } else if (this.infoKey === 'label') {
        return 'label'
      } else if (this.infoKey === 'category') {
        return 'category'
      } else if (this.infoKey === 'super_by') {
        return 'longList'
      } else if (this.$store.state.dataManager.displayCopyKey.indexOf(this.infoKey) !== -1) {
        return 'textWithCopy'
      } else if (this.editable) {
        return 'input'
      } else {
        return 'text'
      }
    }
  },
  methods: {
    deleteInfoKey() {
      if (this.deletable) {
        this.$store.commit('deleteGroupDetailItem', this.infoKey)
      }
    },
    editLabel (payload) {
      let labels = this.$store.state.dataManager.groupDetail[this.infoKey]
      // Value of manually entered label is empty string
      if (labels === null || labels === '') {
        this.$store.state.dataManager.groupDetail[this.infoKey] = []
        labels = this.$store.state.dataManager.groupDetail[this.infoKey]
      }
      if (payload.action === 'add') {
        let labelInfo = this.$store.state.dataManager.labels[payload.id]
        labels.push({
          name: labelInfo.name,
          color: labelInfo.color,
          description: labelInfo.description
        })
      } else if (payload.action === 'remove') {
        let target = ''
        for (const index in labels) {
          if (labels[index].name === payload.name) {
            target = index
            break
          }
        }
        labels.splice(target, 1)
      } else { }
      this.$store.commit('setGroupDetailItem', { key: this.infoKey, value: labels })
      this.$store.dispatch('loadDataLabel')
    },
    loadQrcodeImg () {
      this.$store.dispatch('activateGroup', this.$store.state.dataManager.focusNodeInfo)
      this.imgData = ''
      getQrcodeImg(this.inputValue)
        .then(response => {
          this.imgData = response.data.img
        })
        .catch(error => {
          this.$bus.$emit('msg.error', 'Make QRCode error: ' + error.data.message)
        })
    },
    getQrcodeImgWidth () {
      if (!this.inputValue) {
        return 0
      } else if (this.inputValue.length < 1024) {
        return 250
      } else {
        return 382
      }
    },
    changeTextLongListShowAll () {
      this.isLongListShowAll = !this.isLongListShowAll
    },
    onUrlCopy () {
      const originCopyIcon = this.copyIcon
      this.copyIcon = 'mdi-check'
      setTimeout(() => {
        this.copyIcon = originCopyIcon
      }, 2 * 1000)
    },
    onUrlCopyError (e) {
      this.$bus.$emit('msg.error', 'Copy url error:' + e)
    }
  }
}
</script>

<style>
.enable-button {
  cursor: pointer;
}
.disable-button {
  color: #c5c8ce;
}
.data-label {
  font-size: 12px;
  padding: 0px 6px;
  margin: 0px 3px;
  color:white;
  border-radius: 10px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  display: inline-block;
  max-width: 200px;
  vertical-align: bottom;
  font-weight: 500;
}
</style>
