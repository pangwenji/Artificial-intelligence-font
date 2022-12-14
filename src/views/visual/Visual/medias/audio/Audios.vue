/** Copyright 2020 Tianshu AI Platform. All Rights Reserved.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 * =============================================================
 */

<template>
  <div class="audioscontainer">
    <div :class="['audios-title', showFlag ? 'titleStyle' : '']" @click="showContent()">
      <div :class="[showFlag ? 'sub' : 'sub1']">
        <div class="my-label">
          <div class="my-text"><span>音频</span></div>
          <div class="circle-father"><div class="circle" /></div>
          <div class="triangle-father">
            <div class="triangle" />
          </div>
        </div>
      </div>
      <div class="line1" />
      <div :class="['line2', showFlag ? 'linestyle' : '']" />
    </div>
    <div :class="[showFlag ? 'showClass' : '']">
      <div class="audioscontent">
        <el-row :gutter="20">
          <audio-container
            v-for="(item, i) in detailData[subname]"
            v-show="showrun[item.run]"
            :key="item.index"
            :content="item"
            :index="i"
            :parentComponent="parentComponent"
          />
        </el-row>
      </div>
    </div>
  </div>
</template>

<script>
import { createNamespacedHelpers } from 'vuex';
import { audioContainer } from './audioContainer';

const {
  mapGetters: mapMediaGetters,
  mapActions: mapMediaActions,
  mapMutations: mapMediaMutations,
} = createNamespacedHelpers('Visual/media');
const { mapState: mapLayoutStates } = createNamespacedHelpers('Visual/layout');
export default {
  components: {
    audioContainer,
  },
  props: {
    subname: String,
    index: Number,
    value: Array,
  },
  data() {
    return {
      showFlag: true, // 刚开始 都先不显示 点击之后 请求数据 再显示
      parentComponent: true,
    };
  },
  computed: {
    ...mapMediaGetters(['detailData', 'categoryInfo', 'showrun', 'getShowFlag']),
    ...mapLayoutStates(['userSelectRunFile']),
  },
  watch: {
    userSelectRunFile(val) {
      this.setshowrun(val);
      if (this.showFlag === false) {
        this.showFlag = true;
        this.showFlag = false;
      }
    },
    showFlag() {
      this.setShowFlag([this.subname, this.showFlag]);
    },
  },
  created() {
    this.setshowrun(this.userSelectRunFile);
    if (this.index === 0 && this.getShowFlag.firstTime) {
      this.showFlag = false;
      this.setFreshInfo([this.subname, false]);
      this.getData([this.subname, this.value]);
      this.setShowFlag(['firstTime', false]);
    }
    if (!this.getShowFlag.firstTime) {
      if (typeof this.getShowFlag[this.subname] !== 'undefined') {
        this.showFlag = this.getShowFlag[this.subname];
      }
    }
  },
  methods: {
    ...mapMediaActions(['getData']),
    ...mapMediaMutations(['setDetailData', 'setshowrun', 'setFreshInfo', 'setShowFlag']),
    showContent() {
      if (this.showFlag) {
        this.showFlag = false;
        this.setFreshInfo([this.subname, false]);
        if (this.index !== 0) {
          this.getData([this.subname, this.value]);
        }
      } else {
        this.showFlag = true;
        this.setFreshInfo([this.subname, true]);
      }
    },
  },
};
</script>

<style lang="less" scoped>
.my-label {
  display: flex;
  width: 100%;
  cursor: pointer;

  .triangle {
    position: absolute;
    width: 0;
    height: 0;
    overflow: hidden;
    border-color: transparent transparent transparent #7f7cc1;
    border-style: dashed dashed dashed solid;
    border-top-width: 15px;
    border-right-width: 18px;
    border-bottom-width: 15px;
    border-left-width: 18px;
  }

  .triangle-father {
    position: relative;
  }

  .circle-father {
    position: relative;
    width: 15%;
    height: 30px;
    background-color: #7f7cc1;
  }

  .circle {
    position: absolute;
    top: 50%;
    left: 50%;
    width: 8px;
    height: 8px;
    background-color: white;
    border-radius: 50%;
    transform: translateX(-50%) translateY(-50%);
  }

  .my-text {
    width: 70%;
    height: 30px;
    text-align: center;
    vertical-align: center;
    background-color: #7f7cc1;
  }

  span {
    align-items: center;
    line-height: 30px;
    color: white;
  }
}

.audioscontainer {
  .audios-title {
    display: flex;
    align-items: center;
    height: auto;
    color: white;
    background-color: white;

    span {
      font-weight: 700;
      // margin-left: 29%;
      line-height: 30px;
    }
  }
}

.audioscontent {
  padding: 2%;
}

.showClass {
  display: none;
}

.sub .triangle {
  border-color: transparent transparent transparent #b8c6ff;
}

.sub .circle-father {
  background-color: #b8c6ff;
}

.sub .my-text {
  background-color: #b8c6ff;
}

.sub {
  display: flex;
  align-items: center;
  width: 9.5%;
  height: 30px;
  font-family: sans-serif;
  font-size: 12px;
  color: #fff;
  // background: url(../../../assets/tag2.png);
  // background: deeppink; b8c6ff
  background-size: 100% 100%;
}

.sub1 {
  display: flex;
  align-items: center;
  width: 9.5%;
  height: 30px;
  font-family: sans-serif;
  font-size: 12px;
  color: #fff;
  background-size: 100% 100%;
}

.line1 {
  width: 88%;
  height: 1px;
  background-color: #f4f6ff;
}

.line2 {
  width: 2.5%;
  height: 5px;
  background-color: #625eb3;
}

.linestyle {
  background-color: #b9c6ff;
}
</style>
