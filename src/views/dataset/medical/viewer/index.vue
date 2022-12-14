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
  <div class="rel dwv-container">
    <div id="dwv" v-hotkey="keymap" class="dwv-wrapper viewer rel">
      <div class="dwv-page-header flex flex-between">
        <ToolBar
          :tools="state.actions"
          :activeTool="state.activeTool"
          :updateState="updateState"
          :wlPreset="state.wlPreset"
          :shape="state.shape"
          :precision="state.precision"
          :annotations="state.annotations"
          :status="state.caseInfo.status"
          @change="handleChangeTool"
          @open="handleOpenTool"
        />
        <Action
          :getApp="getApp"
          :medicalId="medicalId"
          :save="saveAction"
          :sliceDrawingMap="state.sliceDrawingMap"
          :changedDrawId="state.changedDrawId"
          :rawAutoAnnotationIds="state.rawAutoAnnotationIds"
        />
      </div>
      <div class="layer-content flex flex-beween">
        <div class="rel flex flex-beween" style="flex: 1;">
          <div style="width: 100%;">
            <div class="layerContainer mx-auto rel" :class="state.loading ? 'hidden' : ''">
              <canvas ref="canvasRef" class="imageLayer"></canvas>
              <div class="drawDiv"></div>
            </div>
            <InfoLayer
              v-if="state.curInstanceID && state.showInfo"
              :seriesInfo="state.seriesDicomInfo"
              :curInstanceID="state.curInstanceID"
              :overlayInfo="state.overlayInfo"
              :stack="state.stack"
            />
          </div>
          <div v-if="state.loading" class="flex flex-center dwv-placeholder">?????????...</div>
        </div>
        <div v-if="state.showLesionInfo" class="settingsInfo" style="width: 22%;">
          <LesionInfo
            :lesions="state.lesions"
            :deleteDraw="deleteDraw"
            :setCurrentSlice="setCurrentSlice"
            :editDrawDetail="editDrawDetail"
            :editLesionItem="editLesionItem"
            :editLesionOrder="editLesionOrder"
            :getApp="getApp"
          />
        </div>
      </div>
      <div style="height: 16px;"></div>
      <portal-target name="toolsAction" />
    </div>
  </div>
</template>
<script>
import dwv from '@wulucxy/dwv';
import { onMounted, reactive, ref, computed, onBeforeUnmount } from '@vue/composition-api';
import { findIndex, uniq, isNil, max } from 'lodash';

import { toFixed, remove, replace, noop, mergeArrayByKey, generateUuid } from '@/utils';
import {
  getCaseInfo,
  queryAutoResult,
  queryManualResult,
  save,
  saveLesions,
  queryLesions,
} from '@/api/preparation/medical';
import { datasetStatusMap } from '@/views/dataset/util';
import LesionInfo from './lesionInfo';
import InfoLayer from './infoLayer';
import ToolBar from './toolbar';
import Action from './action';
import actions, { viewerCommands } from '../lib/actions';
import {
  dwc,
  getImageIdsForSeries,
  getSOPInstanceUID,
  genDrawingFromAnnotations,
  buildWADOUrl,
  getDrawLayer,
} from '../lib';
import { getAnnotateType } from '../constant';
import '../lib/gui';

export default {
  name: 'DatasetMedicalViewer',
  components: {
    InfoLayer,
    ToolBar,
    Action,
    LesionInfo,
  },
  setup(props, ctx) {
    const { $route, $router } = ctx.root;
    const { params = {} } = $route;
    const canvasRef = ref(null);

    // ?????? id
    const lesionOrderMap = {
      count: 0,
    };

    const state = reactive({
      actions,
      activeTool: 'Scroll',
      annotations: null,
      caseInfo: {}, // studyID ??? SeriesId
      loading: false,
      loadItemCount: 0,
      loadProgress: 0, // ????????????
      receivedError: 0, // ????????????
      seriesMetadata: [], // ???????????????
      curInstanceID: undefined, // ???????????? ID
      seriesDicomInfo: {}, // ???????????? dicom ??????
      showInfo: true, // ????????????????????????
      overlayInfo: {
        // ???????????????
        zoom: { scale: 1 },
      },
      stack: {
        imageIds: [],
        currentImageIdIndex: 0,
      },
      shape: 'Rectangle', // ??????????????????
      wlPreset: '', // ????????????????????????
      autoAnnotationIds: [], // ????????????????????????
      rawAutoAnnotationIds: [], // ????????????????????????
      sliceDrawingMap: {}, // ?????? slice ????????? Id ?????? map
      changedDrawId: [], // ?????????????????????????????? drawId
      precision: 0.5, // ?????????????????????50%???????????????????????????????????????50%????????????
      showLesionInfo: false, // ????????????????????????
      lesions: [], // ??????????????????????????????
    });

    const tools = {
      Scroll: {},
      ZoomAndPan: {},
      WindowLevel: {},
      Draw: {
        options: ['Roi', 'Rectangle'],
        type: 'factory',
        events: ['draw-create', 'draw-change', 'draw-move', 'draw-delete'],
      },
    };

    let dwvApp = null;

    // ?????????????????? tools ??????
    const isTool = (tool) => {
      return Object.keys(tools).includes(tool);
    };

    // ??????????????????????????????
    const updateState = (params) => {
      // ????????????????????????????????????
      if (typeof params === 'function') {
        const next = params(state);
        Object.assign(state, next);
        return;
      }
      // ????????????
      Object.assign(state, params);
    };

    const loadInfo = (event) => {
      const { data } = event;
      const SOPInstanceUID = getSOPInstanceUID(data);
      // ?????????????????????????????????
      if (state.seriesDicomInfo[SOPInstanceUID]) return;
      const nextInfo = {
        ...state.seriesDicomInfo,
        [SOPInstanceUID]: data,
      };
      Object.assign(state, {
        seriesDicomInfo: nextInfo,
      });
    };

    const changeShape = (shape, tool) => {
      const activeTool = tool || state.activeTool;
      if (dwvApp && activeTool === 'Draw' && shape !== '') {
        dwvApp.setDrawShape(shape);
      }
    };

    const toggleCanvasListener = (canvas, bool) => {
      if (bool === false) {
        canvas.setAttribute('style', 'pointer-events: none;');
      } else {
        // todo: draw layer
        canvas.setAttribute('style', 'pointer-events: auto;');
      }
    };

    // ?????? tool ????????????
    const toggleToolEvent = (tool, bool) => {
      const drawLayer = getDrawLayer(dwvApp);
      toggleCanvasListener(canvasRef.value, bool);
      // ?????? drawlayer ???????????????
      const enable = tool.command === 'Draw' || tool.type !== 'tool' ? bool : !bool;
      toggleCanvasListener(drawLayer.parent.content, enable);
    };

    // ???????????????draw ???????????????????????????
    const handleOpenTool = (isOpen, tool) => {
      if (isOpen) {
        if (tool.command === 'Draw') {
          if (state.shape) {
            dwvApp.setTool(tool.command);
            changeShape(state.shape, 'Draw');
            toggleToolEvent(tool, true);
          }
        }
        updateState({
          activeTool: tool.command,
        });
      }
    };

    const handleChangeTool = (tool) => {
      if (tool.type === 'tool') {
        dwvApp.setTool(tool.command);
        if (tool.command === 'Draw') {
          changeShape(tool.value, 'Draw');
          tool.value &&
            updateState({
              shape: tool.value,
            });
        }
        // ?????????command ????????????????????????????????????
        if (!isTool(state.activeTool)) {
          // ??? tool ???????????? canvas ??????
          toggleToolEvent(tool, true);
        }
      } else {
        const selectedTool = dwvApp.getToolboxController().getSelectedTool();
        if (selectedTool) {
          // ??? tool ???????????? canvas ??????
          toggleToolEvent(tool, false);
        }
        viewerCommands[tool.command](dwvApp, updateState, tool, state);
      }
      updateState({
        activeTool: tool.command,
      });
    };

    const updateOverlayInfo = (info) => {
      if (typeof info === 'function') {
        const next = info(state.overlayInfo);
        Object.assign(state, {
          overlayInfo: {
            ...state.overlayInfo,
            ...next,
          },
        });
        return;
      }
      Object.assign(state, {
        overlayInfo: {
          ...state.overlayInfo,
          ...info,
        },
      });
    };

    const updateWwwc = (event) => {
      updateOverlayInfo({
        ww: event.ww,
        wc: event.wc,
      });
    };

    const updateZoom = (event) => {
      if (event.type === 'zoom-change') {
        updateOverlayInfo({
          zoom: {
            ...event,
            scale: toFixed(event.scale, 0, 2),
          },
        });
      }
    };

    // ??????????????????id
    const addChangedId = (id) => {
      if (!state.changedDrawId.includes(id)) {
        state.changedDrawId.push(id);
      }
    };

    const removeAnnotation = () => {
      // todo ??????????????? mac backspace ??????
      // const drawLayer = dwvApp.getDrawController().getDrawLayer();
      // const currentGroup = dwvApp.getDrawController().getCurrentPosGroup();
    };

    const onSliceChange = (event) => {
      updateState((state) => ({
        stack: {
          ...state.stack,
          currentImageIdIndex: event.value,
        },
      }));
    };

    // ???????????? 1. ???????????????2. ??????????????????????????????
    const saveAction = ({ drawing }) => {
      const saveDrawingPromise = save(drawing);
      const promises = [saveDrawingPromise];
      // ??????????????????
      if (state.showLesionInfo) {
        const saveLesionPromise = saveLesions(params.medicalId, state.lesions);
        promises.push(saveLesionPromise);
      }

      return Promise.all(promises);
    };

    // ??????????????????
    const updateLesionCount = () => {
      const counts = state.lesions.reduce((acc, d) => {
        if (!isNil(d.lesionOrder)) {
          acc.push(d.lesionOrder);
        }
        return acc;
      }, []);

      // ?????? count ??????
      // eslint-disable-next-line
      lesionOrderMap.count = counts.length
        ? isNil(max(counts))
          ? 1
          : max(counts)
        : state.lesions.length;
    };

    // ?????????????????????????????????
    const applyState = async (app) => {
      const { caseInfo } = state;
      switch (datasetStatusMap[caseInfo.status]?.status) {
        case 'AUTO_ANNOTATED': {
          // ??????????????????
          const autoResult = await queryAutoResult(caseInfo.id);
          const {
            drawings,
            drawingsDetails,
            drawingIds,
            sliceDrawingMap,
          } = genDrawingFromAnnotations(JSON.parse(autoResult.annotations), state.precision);
          Object.assign(state, {
            sliceDrawingMap,
            annotations: autoResult.annotations,
            autoAnnotationIds: drawingIds,
            rawAutoAnnotationIds: drawingIds, // ??????????????????????????????????????????????????????????????????
          });
          app.setDrawings(drawings, drawingsDetails);
          break;
        }
        // ??????????????????????????????
        case 'ANNOTATED':
        case 'ANNOTATING': {
          const result = await queryManualResult(caseInfo.id);
          const { drawings, drawingsDetails, sliceDrawingMap } = result;
          Object.assign(state, {
            sliceDrawingMap,
          });
          app.setDrawings(drawings, drawingsDetails);
          break;
        }
        // ?????????  ???????????????
        case 'UNANNOTATED':
          break;
        // ???????????????????????????????????????
        default:
          setTimeout(() => {
            $router.replace('/data/datasets/medical');
          }, 2000);
          throw new Error('????????????????????????????????????');
      }

      if (state.showLesionInfo) {
        const drawDetails = app.getDrawDisplayDetails();
        queryLesions(params.medicalId).then((res) => {
          try {
            const lesions = res.map((d) => {
              const rawList = JSON.parse(d.list).map((d) => ({
                ...d,
                id: d.drawId,
              }));
              return {
                ...d,
                list: mergeArrayByKey(rawList, drawDetails, 'id'),
              };
            });
            Object.assign(state, {
              lesions,
            });
            // ???????????? count
            updateLesionCount();
          } catch (err) {
            console.error(err);
            throw err;
          }
        });
      }
    };

    // ????????????????????????
    const moveDrawIdBy = (source, findBy) => {
      const changedIndex = findBy(state[source]);
      if (changedIndex > -1) {
        const updateArr = remove(state[source], changedIndex);
        Object.assign(state, {
          [source]: updateArr,
        });
      }
    };

    const deleteLesionInfo = (id) => {
      if (state.showLesionInfo && id) {
        // ??????????????? drawId ????????? lesions ?????????
        moveDrawIdBy('lesions', (arr) =>
          findIndex(arr, (o) => {
            return (o.list || []).map((d) => d.drawId).includes(id);
          })
        );
      }
    };

    // ?????? draw ??????
    const deleteDrawInfo = (id) => {
      if (!id) return;
      // ??????????????? drawId ????????? changedDrawId????????????
      moveDrawIdBy('changedDrawId', (arr) => findIndex(arr, (o) => o === id));
      deleteLesionInfo(id);
    };

    // ??????????????????
    const deleteDrawShape = (id) => {
      if (!id) return;
      const drawLayer = getDrawLayer(dwvApp);
      const groups = drawLayer.getChildren();
      for (const group of groups) {
        const groupToDelete = group.getChildren((node) => node.id() === id);
        if (groupToDelete.length === 1) {
          dwvApp.getDrawController().deleteDrawGroup(groupToDelete[0], noop, noop);
        }
      }
    };

    // ???????????????????????????
    const deleteDraw = (row = {}) => {
      // ??????????????????
      const drawIds = (row.list || []).map((d) => d.drawId).filter((d) => !!d);
      for (const id of drawIds) {
        deleteDrawShape(id);
        deleteDrawInfo(id);
      }
      // TODO??????????????? api
      // ???????????????????????????
      // deleteLesion(row.id).then(() => {
      //   Message.success('??????????????????');
      // });
    };

    // ???????????????slice
    const setCurrentSlice = (k) => {
      dwvApp.getViewController().setCurrentSlice(k);
    };

    // ?????? draw ??????
    const updateDrawColor = (drawIds, color) => {
      const drawDetails = dwvApp.getDrawDisplayDetails();
      drawIds.forEach((drawId) => {
        const drawDetail = drawDetails.find((d) => d.id === drawId);
        if (drawDetail) {
          drawDetail.color = color;
          addChangedId(drawId);
          dwvApp.updateDraw(drawDetail);
        }
      });
    };

    // ???????????? id
    const editLesionOrder = (value, row) => {
      const index = findIndex(state.lesions, (o) => o.lesionOrder === Number(value));
      // ?????????????????????????????? lesionOrder
      if (index > -1) {
        const rawlesion = state.lesions[index];
        const sliceDesc = String(rawlesion.sliceDesc)
          .split(',')
          .concat(String(row.sliceDesc).split(','));

        // ???????????????
        const rawColor = rawlesion.list[0].color;
        // ???????????? draw ??????
        const nextList = rawlesion.list.concat(row.list);

        let nextLesions = replace(state.lesions, index, {
          ...rawlesion,
          sliceDesc: uniq(sliceDesc)
            .sort()
            .join(','),
          list: nextList,
        });

        // ????????????????????????????????????
        const mergedRowIndex = findIndex(nextLesions, (o) => o.id === row.id);
        if (mergedRowIndex > -1) {
          nextLesions = remove(nextLesions, mergedRowIndex);
        }

        Object.assign(state, {
          lesions: nextLesions,
        });
        const drawIds = nextList.map((d) => d.drawId);
        updateDrawColor(drawIds, rawColor);
      } else {
        const curIndex = findIndex(state.lesions, (o) => o.id === row.id);
        const nextRow = {
          ...row,
          lesionOrder: Number(value), // lesionOrder ???????????????
        };
        const nextLesions = replace(state.lesions, curIndex, nextRow);
        Object.assign(state, {
          lesions: nextLesions,
        });
      }
      // ??????????????????
      updateLesionCount();
    };

    // eslint-disable-next-line
    const editDrawDetail = (type, value, row) => {
      switch (type) {
        case 'color': {
          const drawIds = row.list.map((d) => d.drawId);
          updateDrawColor(drawIds, value);
          break;
        }
        default:
          return false;
      }
    };

    // ??????????????????
    const editLesionItem = (value, row) => {
      const index = findIndex(state.lesions, (o) => o.id === row.id);
      const nextLesions = replace(state.lesions, index, {
        ...row,
        sliceDesc: value,
      });
      Object.assign(state, {
        lesions: nextLesions,
      });
    };

    // ?????????
    const keymap = computed(() => ({
      delete: removeAnnotation,
      backspace: removeAnnotation,
      esc: deleteDraw,
    }));

    onMounted(async () => {
      dwvApp = new dwv.App();

      dwvApp.init({
        containerDivId: 'dwv',
        tools,
      });

      // ?????????????????????????????? studyId ??? seriesId
      const caseInfo = await getCaseInfo(Number(params.medicalId));
      state.caseInfo = caseInfo;
      const { annotateType } = caseInfo;
      // ???????????????????????????
      const showLesionInfo = getAnnotateType(annotateType) === 1;

      const seriesMetadata = await dwc.retrieveSeriesMetadata(caseInfo);
      if (!seriesMetadata || !seriesMetadata.length) {
        throw new Error('???????????????');
      }
      const imageInstances = getImageIdsForSeries(seriesMetadata);
      const imageIds = imageInstances.map(buildWADOUrl);
      Object.assign(state, {
        caseInfo,
        showLesionInfo,
        seriesMetadata,
        stack: {
          ...state.stack,
          imageIds,
        },
      });
      dwvApp.addEventListener('load-start', () => {
        Object.assign(state, {
          loadItemCount: 0,
          loading: true,
        });
      });
      dwvApp.addEventListener('load-progress', (event) => {
        Object.assign(state, {
          loadProgress: event.loaded,
        });
      });
      dwvApp.addEventListener('load-item', (event) => {
        Object.assign(state, {
          loadItemCount: state.loadItemCount + 1,
        });
        // ?????? meta ??????
        if (event.loadtype === 'image') {
          loadInfo(event);
        }
      });

      dwvApp.addEventListener('load', (event) => {
        const firstObjectId = new URL(imageIds[0]).searchParams.get('objectUID');
        // ??????????????????
        if (event.source.length) {
          // ?????????????????????????????????
          Object.assign(state, {
            curInstanceID: firstObjectId,
          });
        }
      });

      // ????????????
      dwvApp.addEventListener('load-end', async () => {
        const nextState = {
          loading: false,
          loadProgress: 100,
        };
        if (state.nReceivedError > 0) {
          Object.assign(nextState, {
            loadProgress: 0,
            loadItemCount: 0,
          });
        }
        // ????????? tool
        dwvApp.setTool(state.activeTool);

        dwvApp.getViewController().goFirstSlice();
        // ????????????
        await applyState(dwvApp);
        Object.assign(state, nextState);
      });

      dwvApp.addEventListener('error', (event) => {
        console.error('load error', event);
        Object.assign(state, {
          receivedError: state.receivedError + 1,
        });
      });

      // ???????????? id ???????????????
      dwvApp.addEventListener('draw-move', ({ id }) => {
        // ???????????????????????? drawId
        addChangedId(id);
      });

      dwvApp.addEventListener('draw-create', ({ id }) => {
        const drawDetails = dwvApp.getDrawDisplayDetails();
        const drawInfo = drawDetails.find((d) => d.id === id) || {};
        // ????????????????????????????????????????????????
        if (state.showLesionInfo) {
          const lesionOrder = (lesionOrderMap.count += 1);
          const sliceIndex = state.stack.currentImageIdIndex + 1;
          // ??????????????????
          state.lesions.push({
            id: generateUuid(),
            lesionOrder,
            sliceDesc: sliceIndex,
            list: [
              {
                drawId: id,
                sliceNumber: sliceIndex,
                ...drawInfo,
              },
            ],
          });
        }
      });

      dwvApp.addEventListener('draw-delete', ({ id }) => {
        deleteDrawInfo(id);
      });

      // todo: ???????????? ww,wc?
      // ?????????????????????
      dwvApp.addEventListener('wl-width-change', updateWwwc);
      dwvApp.addEventListener('wl-center-change', updateWwwc);

      // ????????????
      dwvApp.addEventListener('zoom-change', updateZoom);
      dwvApp.addEventListener('slice-change', onSliceChange);

      dwvApp.addEventListener('keydown', (event) => {
        if (event.keyCode === 39) {
          event.preventDefault();
          dwvApp.getViewController().incrementSliceNb();
        }
        if (event.keyCode === 37) {
          event.preventDefault();
          dwvApp.getViewController().decrementSliceNb();
        }
      });

      dwvApp.loadURLs(imageIds, {
        batchSize: 5,
      });
    });

    onBeforeUnmount(() => {
      dwvApp.abortLoad();
    });

    return {
      state,
      medicalId: params.medicalId,
      updateState,
      keymap,
      canvasRef,
      handleChangeTool,
      handleOpenTool,
      saveAction,
      deleteDraw,
      setCurrentSlice,
      editLesionOrder,
      editLesionItem,
      editDrawDetail,
      getApp: () => dwvApp,
    };
  },
};
</script>
<style lang="scss">
.fullBg {
  background-color: #fff;
  opacity: 0.3;
}
</style>
<style lang="scss" scoped>
.dwv-wrapper {
  justify-content: center;
  height: 100%;
  padding: 16px 20px 0 0;
  color: #9ccef9;
  user-select: none;
  background: #000;

  .dwv-page-header {
    padding-bottom: 12px;
    border-bottom: 1px solid rgba(173, 216, 230, 0.5);
  }

  .layer-content {
    height: calc(100vh - 80px);
  }

  .settingsInfo {
    padding-top: 12px;
    padding-left: 12px;
    border-left: 1px solid rgba(173, 216, 230, 0.5);
  }

  .drawDiv {
    position: absolute;
    top: 0;
    left: 0;
    pointer-events: none;
  }
}

.dwv-container {
  height: 100vh;
}

.dwv-placeholder {
  position: absolute;
  top: 80px;
  left: 0;
  width: 100%;
  height: calc(100vh - 80px);
  color: #9ccef9;
}
</style>
