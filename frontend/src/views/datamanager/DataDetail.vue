<template>
  <div>
    <Row class="button-bar">
      <Col span="15" class="button-bar-line">
        <span v-for="(value, index) in nodeParents" :key="value.id">
          <v-icon v-if="!value.parent_id" small color="accent" @click="showNode(value)">mdi-home</v-icon>
          <a v-else @click="showNode(value)">{{value.name}}</a>
          <v-icon small v-show="index !== nodeParents.length-1">
            mdi-chevron-right
          </v-icon>
        </span>
      </Col>
      <Col span="8" offset="1" align="right" class="button-bar-line">
        <JsonPathBar/>
      </Col>
    </Row>
    <component v-if="nodeInfo.type" :is="getComponentByType(nodeInfo)" />
    <div v-else class="data-detail-empty">No selected data</div>
  </div>
</template>

<script>
import DataDetailHttpData from '@/views/datamanager/DataDetailHttpData.vue'
import DataDetailFolder from '@/views/datamanager/DataDetailFolder.vue'
import JsonPathBar from '@/views/datamanager/JsonPathBar.vue'
import { getGroupDetail } from '@/api'

export default {
  components: {
    DataDetailHttpData,
    DataDetailFolder,
    JsonPathBar
  },
  computed: {
    groupDetail () {
      return this.$store.state.dataManager.groupDetail
    },
    nodeInfo () {
      return this.$store.state.dataManager.focusNodeInfo
    },
    nodeParents () {
      return this.$store.state.dataManager.focusNodeInfo.parent
    }
  },
  methods: {
    getComponentByType (payload) {
      if (payload.type === 'data') {
        return 'DataDetailHttpData'
      } else if (payload.type === 'group') {
        return 'DataDetailFolder'
      } else {
        return ''
      }
    },
    showNode (payload) {
      if (payload.type === 'group') {
        getGroupDetail(payload.id)
          .then(response => {
            this.$store.commit('setGroupDetail', response.data.data)
            this.$store.commit('setFocusNodeInfo', response.data.data)
          })
          .catch(error => {
            bus.$emit('msg.error', 'Load group ' + payload.name + ' error: ' + error.data.message)
          })
      } else if (payload.type === 'data') {
        this.$store.dispatch('loadDataDetail', payload)
      } else { }
    }
  }
}
</script>

<style scoped>
.button-bar {
  height: 31px;
  display: flex;
  align-items: center;
  padding: 10px;
  border-bottom: 1px solid #ddd;
}
.button-bar-line {
  display: inline-block;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.small-tab > .ivu-tabs > .ivu-tabs-bar {
  margin-bottom: 0;
}
.data-detail-empty {
  position: absolute;
  top: 40%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
}
</style>
