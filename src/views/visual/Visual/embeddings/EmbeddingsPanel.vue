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
  <div class="embeddingpanel">
    <div class="panel">
      <el-card class="father">
        <div class="info">
          <div class="infoTitle">
            <span i class="iconfont icon-ziyuan40">&nbsp;&nbsp;控制面板</span>
          </div>
          <div class="infoContent">
            <div class="infoItem">
              <el-row type="flex" justify="space-between">
                <el-col :span="8">
                  <div>
                    <span class="midFont">标&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;签</span>
                  </div>
                </el-col>
                <el-col :span="16">
                  <div class="center">
                    <el-select
                      v-model="curTag"
                      class="tagSelect histmodeselect"
                      size="small"
                      placeholder="请选择"
                    >
                      <el-option
                        v-for="(item, index) in curTags"
                        :key="index"
                        :value="item"
                        :label="item"
                      />
                    </el-select>
                  </div>
                </el-col>
              </el-row>
            </div>
            <div class="infoItem">
              <el-row type="flex" justify="space-between">
                <el-col :span="8">
                  <div>
                    <span>降维方法</span>
                  </div>
                </el-col>
                <el-col :span="16">
                  <div class="center">
                    <el-select v-model="curMethod" class="tagSelect histmodeselect" size="small">
                      <el-option
                        v-for="item in allMethods"
                        :key="item.value"
                        :label="item.label"
                        :value="item.value"
                      />
                    </el-select>
                  </div>
                </el-col>
              </el-row>
            </div>
            <div class="infoItem">
              <el-row type="flex" justify="space-between">
                <el-col :span="8">
                  <div>
                    <span>维度选择</span>
                  </div>
                </el-col>
                <el-col :span="16">
                  <div class="center">
                    <el-select v-model="curDim" class="tagSelect histmodeselect" size="small">
                      <el-option
                        v-for="(item, index) in allDims"
                        :key="index"
                        :value="item"
                        :label="item"
                      />
                    </el-select>
                  </div>
                </el-col>
              </el-row>
            </div>
            <div v-if="getReceivedQuestionInfo && getReceivedCurInfo" class="infoItem">
              <el-row type="flex" justify="space-around" class="row-bg">
                <el-col :span="5">
                  <el-button
                    background-color="#736FBC"
                    type="primary"
                    size="medium"
                    :icon="playAction ? 'iconfont icon-zanting' : 'iconfont icon-ziyuan74'"
                    @click="playActionClick()"
                  />
                </el-col>
                <el-col :span="11">
                  <el-slider
                    v-model="curStep"
                    :min="curMin"
                    :max="curMapMax"
                    :disabled="curMapMax === 0"
                    input-size="small"
                    class="rangeNumber"
                    :show-tooltip="false"
                  />
                </el-col>
                <el-col :span="8">
                  <div class="grid-content">
                    <span text-align="center" class="midFont">{{ curMapStep }} / {{ curMax }}</span>
                  </div>
                </el-col>
              </el-row>
            </div>
          </div>
        </div>
      </el-card>
      <el-card class="father">
        <div class="info">
          <div class="infoTitle">
            <span i class="iconfont icon-ziyuan41">&nbsp;&nbsp;数据信息栏</span>
          </div>
          <div class="infoContent">
            <div class="infoItem">
              <div v-if="getMessage == ''">
                <span>暂无信息</span>
              </div>
              <el-card
                v-if="getMessage != ''"
                :body-style="{ padding: '0px' }"
                class="infoCard"
                display="none"
              >
                <div
                  v-if="getQuestionInfo[userSelectRunFile][getCurInfo.curTag]['sample']"
                  class="showBox"
                >
                  <div
                    v-if="
                      getQuestionInfo[userSelectRunFile][getCurInfo.curTag]['sample_type'] ==
                        'text' && getMessage != ''
                    "
                    class="image"
                  >
                    <el-scrollbar style="height: 100%;">
                      <p>{{ getPanelSampleData['url'] }}</p>
                    </el-scrollbar>
                  </div>
                  <div
                    v-if="
                      getQuestionInfo[userSelectRunFile][getCurInfo.curTag]['sample_type'] ==
                        'image' && getMessage != ''
                    "
                    class="image"
                  >
                    <el-scrollbar style="height: 100%;">
                      <el-image
                        :src="getPanelSampleData['url']"
                        :preview-src-list="[getPanelSampleData['url']]"
                        class="image"
                      />
                    </el-scrollbar>
                  </div>
                  <div
                    v-if="
                      getQuestionInfo[userSelectRunFile][getCurInfo.curTag]['sample_type'] ==
                        'audio' && getMessage != ''
                    "
                    class="image"
                  >
                    <el-scrollbar style="height: 100%;">
                      <AudioContainer
                        v-if="
                          getQuestionInfo[userSelectRunFile][getCurInfo.curTag]['sample_type'] ==
                            'audio' && getMessage != ''
                        "
                        :theUrl="getPanelSampleData['url']"
                        theControlList="noMuted noSpeed onlyOnePlaying"
                        :index="1000"
                      />
                    </el-scrollbar>
                  </div>
                </div>
                <div style="padding: 14px;" class="imageSpan">
                  <span>{{ getMessage[1] }}</span>
                </div>
              </el-card>
            </div>
          </div>
        </div>
      </el-card>
    </div>
  </div>
</template>

<script>
import { createNamespacedHelpers } from 'vuex';

const {
  mapGetters: mapEmbeddingGetters,
  mapMutations: mapEmbeddingMutations,
  mapActions: mapEmbeddingActions,
} = createNamespacedHelpers('Visual/embedding');
const { mapState: mapLayoutStates, mapGetters: mapLayoutGetters } = createNamespacedHelpers(
  'Visual/layout'
);
export default {
  components: {},
  data() {
    return {
      // ----------------------------- Tag选择 -----------------------------
      curTag: '',
      curTags: [],
      // ----------------------------- method选择 -----------------------------
      curMethod: '',
      allMethods: [
        {
          value: 'PCA',
          label: 'PCA',
        },
        {
          value: 'TSNE',
          label: 'TSNE',
        },
      ],
      // ----------------------------- 维度信息选择选择 -----------------------------
      curDim: '',
      allDims: ['2维', '3维'],
      // ----------------------------- 播放动作 -----------------------------
      playAction: false,
      curStep: 0,
      curMapStep: 0,
      curMin: 0, // 可能稍后赋值同步信息
      curMax: 100,
      curMapMax: 100,
      // ----------------------------- 概率密度 -----------------------------
      checkedLabels: [],
      lableTypes: [],
      curLineWidth: 1,
    };
  },
  computed: {
    ...mapEmbeddingGetters([
      'getCurInfo',
      'getCategoryInfo',
      'getQuestionInfo',
      'getReceivedCategoryInfo',
      'getReceivedCurInfo',
      'getReceivedQuestionInfo',
      'getCurData',
      'getReceivedCurData',
      'getPanelSampleData',
      'getMessage',
      'getInitStateFlag',
      'getErrorMessage',
      'getLineWidth',
    ]),
    ...mapLayoutStates(['userSelectRunFile']),
    ...mapLayoutGetters(['getParams']),
  },
  watch: {
    curLineWidth() {
      this.setLineWidth(this.curLineWidth);
    },
    getReceivedCurInfo() {
      // 只触发一次第一次
      if (this.userSelectRunFile === '') {
        return;
      }
      this.setCurInfo(['received', false]); // 屏蔽别的请求
      for (let i = 0; i < this.getCategoryInfo.curRuns.length; i += 1) {
        if (this.userSelectRunFile === this.getCategoryInfo.curRuns[i]) {
          this.curTags = this.getCategoryInfo.curTags[i].slice(0);
        }
      }
      const someIndex = 0;
      this.curTag = this.curTags[someIndex];
      this.curMethod = this.getCurInfo.curMethod;
      this.curDim = this.getCurInfo.curDim;
      this.curStep = this.getCurInfo.curStep;
      this.curMax = this.getQuestionInfo[this.userSelectRunFile][this.curTag].allSteps[
        this.getQuestionInfo[this.userSelectRunFile][this.curTag].curMax
      ];
      this.curMapMax = this.getQuestionInfo[this.userSelectRunFile][this.curTag].curMax;
      this.curMin = 0;
      this.curMapStep = this.getQuestionInfo[this.userSelectRunFile][this.curTag].allSteps[
        this.curStep
      ];
      const param = {
        run: this.userSelectRunFile,
        tag: this.curTag,
        step: this.curMapStep,
        method: this.curMethod.toLowerCase(),
        dims: parseInt(this.curDim, 10),
      };
      this.fetchDataPower(param);
    },
    curTag() {
      this.setCurInfo(['curTag', this.curTag]);
      this.fetchData();
    },
    curMethod() {
      this.setCurInfo(['curMethod', this.curMethod]);
      this.fetchData();
    },
    curDim() {
      this.setCurInfo(['curDim', this.curDim]);
      this.fetchData();
    },
    curStep() {
      this.setCurInfo(['curStep', this.curStep]);
      this.curMapStep = this.getQuestionInfo[this.userSelectRunFile][this.curTag].allSteps[
        this.curStep
      ];
    },
    curMapStep() {
      this.setCurInfo(['curMapStep', this.curMapStep]);
      this.fetchData();
    },
    getReceivedQuestionInfo() {
      for (let i = 0; i < this.getCategoryInfo.curRuns.length; i += 1) {
        if (this.userSelectRunFile === this.getCategoryInfo.curRuns[i]) {
          this.curTags = this.getCategoryInfo.curTags[i].slice(0);
        }
      }
    },
    checkedLabels(val) {
      this.setCheckLabels(val);
    },
    getMessage() {
      if (
        this.getQuestionInfo[this.userSelectRunFile][this.getCurInfo.curTag].sample &&
        this.getMessage.length > 0
      ) {
        this.fetchSampleData(this.getMessage[0]);
      }
    },
    getReceivedCurData() {
      if (parseInt(this.curDim, 10) > 3) {
        this.lableTypes = this.getCurData.labelType.slice(0);
      }
      const vm = this;
      if (this.playAction) {
        setTimeout(() => {
          if (vm.playAction) {
            let curSteptmp = vm.curStep;
            curSteptmp += 1;
            if (curSteptmp > vm.curMapMax) {
              vm.playAction = false;
            } else {
              vm.curStep += 1;
            }
          }
        }, 2000);
      }
    },
    userSelectRunFile() {
      this.setMessage('');
      if (!this.getReceivedQuestionInfo) {
        return;
      }
      this.setCurInfo(['received', false]); // 屏蔽别的请求
      for (let i = 0; i < this.getCategoryInfo.curRuns.length; i += 1) {
        if (this.userSelectRunFile === this.getCategoryInfo.curRuns[i]) {
          this.curTags = this.getCategoryInfo.curTags[i].slice(0);
        }
      }
      const someIndex = 0;
      this.curTag = this.curTags[someIndex];
      this.curMethod = 'PCA';
      this.curDim = '3维';
      this.curStep = 0;
      this.curMax = this.getQuestionInfo[this.userSelectRunFile][this.curTag].allSteps[
        this.getQuestionInfo[this.userSelectRunFile][this.curTag].curMax
      ];
      this.curMapMax = this.getQuestionInfo[this.userSelectRunFile][this.curTag].curMax;
      this.curMin = 0;
      this.curMapStep = this.getQuestionInfo[this.userSelectRunFile][this.curTag].allSteps[
        this.curStep
      ];
      const param = {
        run: this.userSelectRunFile,
        tag: this.curTag,
        step: this.curMapStep,
        method: this.curMethod.toLowerCase(),
        dims: parseInt(this.curDim, 10),
      };
      this.fetchDataPower(param);
    },
    getErrorMessage(val) {
      this.$message({
        message: val.split('_')[0],
        type: 'error',
      });
    },
  },
  created() {
    // 每次加载的时候都会触发
    this.curLineWidth = this.getLineWidth;
    if (!this.getInitStateFlag) {
      if (this.getReceivedCategoryInfo) {
        this.fetchAllStep();
      }
    } else {
      this.setInitStateFlag(false);
    }
    if (this.getReceivedQuestionInfo) {
      this.setCurInfo(['received', false]); // 屏蔽别的请求
      for (let i = 0; i < this.getCategoryInfo.curRuns.length; i += 1) {
        if (this.userSelectRunFile === this.getCategoryInfo.curRuns[i]) {
          this.curTags = this.getCategoryInfo.curTags[i].slice(0);
        }
      }
      const someIndex = 0;
      this.curTag = this.curTags[someIndex];
      this.curMethod = this.getCurInfo.curMethod;
      this.curDim = this.getCurInfo.curDim;
      this.curStep = this.getCurInfo.curStep;
      this.curMax = this.getQuestionInfo[this.userSelectRunFile][this.curTag].allSteps[
        this.getQuestionInfo[this.userSelectRunFile][this.curTag].curMax
      ];
      this.curMapMax = this.getQuestionInfo[this.userSelectRunFile][this.curTag].curMax;
      // this.curMin = this.getQuestionInfo[this.userSelectRunFile][this.curTag].allSteps[0]
      this.curMin = 0;
      this.curMapStep = this.getQuestionInfo[this.userSelectRunFile][this.curTag].allSteps[
        this.curStep
      ];
      const param = {
        run: this.userSelectRunFile,
        tag: this.curTag,
        step: this.curMapStep,
        method: this.curMethod.toLowerCase(),
        dims: parseInt(this.curDim, 10),
      };
      this.fetchDataPower(param);
    }
  },
  methods: {
    ...mapEmbeddingMutations([
      'setCheckLabels',
      'setCurInfo',
      'setMessage',
      'setInitStateFlag',
      'setLineWidth',
    ]),
    ...mapEmbeddingActions(['fetchSampleData', 'featchData', 'fetchAllStep']),
    playActionClick() {
      this.playAction = !this.playAction; // 取非
      if (this.playAction) {
        if (this.curStep === 0) {
          this.fetchData();
        } else {
          this.curStep = 0;
        }
      }
    },
    fetchData() {
      if (!this.getReceivedCurInfo || !this.getCurInfo.received) {
        // 只要数据未准备好就不触发请求
        return;
      }
      // console.log('this.getCurInfo.curMethod', this.getCurInfo.curMethod)
      const param = {
        run: this.userSelectRunFile,
        tag: this.getCurInfo.curTag,
        step: this.getCurInfo.curMapStep,
        method: this.getCurInfo.curMethod.toLowerCase(),
        dims: parseInt(this.getCurInfo.curDim, 10),
      };
      this.featchData(param);
    },
    fetchDataPower(param) {
      this.featchData(param);
    },
  },
};
</script>
<style lang="less" scoped>
.typeselect {
  height: 30%;
}

select {
  width: 70%;
  height: 30%;
  margin-top: 20%;
}

.panel {
  /deep/ .el-card {
    margin: 3.5% 5% 4% 0%;
    border-top: 0;
  }

  /deep/ .el-card__body {
    padding: 0;
    border-radius: 0 0 3px 3px;
  }

  .info {
    .infoTitle {
      span {
        font-size: 12px;
      }

      padding: 2% 2% 2% 5%;
      color: white;
      text-align: left;
      background-color: #625eb3;
      border-bottom: 1px solid #8f8ad7;

      .iconfont {
        margin-right: 7px;
        font-size: 12px;
        font-style: normal;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;
      }
    }
  }

  /deep/ .el-icon-circle-close {
    color: white;
  }

  /deep/ .el-image-viewer__img {
    height: 100%;
  }

  /deep/ .infoContent {
    @backgroundColorList: #ef6f38, #efdd79, #c5507a, #9359b0, #525c99, #47c1d6, #b5d4e8, #15746c,
      #81c19c, #a08983;
    .backgroundcard(@className, @backgroundColorList,@i) {
      .@{className}@{i} .el-checkbox__inner {
        //属性名称 可以直接拼接属性
        background: @backgroundColorList;
        opacity: 0.5;
      }
    }
    @checkboxClass: checkboxx;
    .loop(0);
    // 选中状态下的透明度得更改
    .backgroundchecked(@className, @backgroundColorList,@i) {
      .@{className}@{i}.is-checked .el-checkbox__inner {
        //属性名称 可以直接拼接属性
        opacity: 1;
      }
    }
    .loop(@i) when(@i < 10) {
      // extract 是取出列表中的对应元素
      .backgroundcard(@checkboxClass, extract(@backgroundColorList, @i+1), @i);
      .backgroundchecked(@checkboxClass, extract(@backgroundColorList, @i+1), @i);
      .loop(@i+1);
    }
    .loop(0);

    padding: 2% 10% 2% 10%;

    .image {
      display: block;
      width: 100%;
      height: 300px;

      p {
        margin-top: 2%;
        margin-right: 2%;
        margin-bottom: 2%;
        margin-left: 2%;
      }
    }

    .infoItem {
      justify-content: center;
      margin: 5% 0 5% 0;
      font-size: 11px;
      text-align: left;

      .center {
        text-align: center;
      }

      span {
        display: flex;
        align-items: center; /* 定义body的元素垂直居中 */
        line-height: 30px;
        text-align: center;
      }
    }

    .el-select {
      border-color: #8f95ad;
    }

    .infoItem .el-select {
      width: 100%;
    }

    .el-slider__bar {
      background-color: #625eb3;
    }

    .el-input__inner {
      height: 30px;
      font-size: 11px;
      line-height: 30px;
      color: #625eb3;
      border-color: #8f95ad;
      border-radius: 50px;
    }

    .el-input.is-focus {
      border-color: #7f7cc1;
    }

    .el-select:hover .el-input__inner {
      border-color: #7f7cc1;
    }

    .el-select__caret {
      font-size: 20px;
      font-weight: 900;
      color: #7f7cc1;
    }

    .imageSpan {
      span {
        display: block !important;
        margin: 0 auto;
        text-align: center;
      }
    }

    .row-bg {
      button {
        color: #7f7cc1;
        background-color: white;
        border-color: white;
      }

      .el-slider__button {
        width: 12px;
        height: 12px;
        border-color: #7f7cc1;
      }

      .el-slider__runway {
        height: 4px;
      }

      .el-slider__bar {
        height: 4px;
      }

      .iconfont {
        font-size: 11px;
      }

      .grid-content {
        height: 38px;
        line-height: 38px;

        span {
          display: block;
          margin: 0 auto;
          line-height: 38px;
          text-align: center;
        }
      }
    }

    .ProbabilityDensitySec {
      width: 93%;

      .ProbabilityDensity {
        margin-bottom: -10px;
        color: #8f8ad7;
      }

      .el-checkbox__label {
        color: #8f8ad7;
      }

      .el-checkbox {
        color: #8f8ad7;
      }

      .el-checkbox__inner {
        border-radius: 50%;
      }

      .leftInline-block {
        display: -webkit-flex; /* Safari */
        display: flex;
        flex-wrap: wrap;
        align-items: center;
        justify-content: flex-start;

        .el-checkbox {
          display: inline-block;

          span {
            display: inline-block;
          }
        }

        .el-checkbox__inner {
          border-color: #dcdfe6;
        }

        .el-checkbox__inner:hover {
          border-color: #dcdfe6;
        }

        .el-checkbox__input.is-checked .el-checkbox__inner::after {
          border-color: white;
        }

        .el-checkbox__input.is_focus .el-checkbox__inner {
          border-color: #dcdfe6;
        }

        .el-checkbox__input.is-checked .el-checkbox__inner {
          border-color: #dcdfe6;
        }
      }

      hr {
        background-color: #8f8ad7;
        border-color: #8f8ad7;
        border-width: 0.1;
        opacity: 0.6;
      }
    }
  }
}

.el-select-dropdown__item.selected {
  color: #625eb3;
}

.el-select-dropdown__item {
  font-size: 11px;
}
</style>
