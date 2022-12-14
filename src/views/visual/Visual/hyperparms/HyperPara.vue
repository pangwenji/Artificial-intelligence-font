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
  <div style="width: 100%; height: 100%;">
    <div id="hp" style="width: 100%; height: 55%;" />
    <div v-show="tableShow" id="grid">
      <el-table
        :data="data"
        :fit="choice"
        :default-sort="{ prop: 'accuracy', order: 'descending' }"
        :header-cell-style="{ background: 'rgb(224, 231, 250)', color: '#606266' }"
        :row-class-name="tableRowClassName"
        :row-style="{ height: '10px' }"
        :cell-style="{ padding: '8px 0' }"
        style="font-size: 12px;"
        @cell-mouse-enter="highlight"
        @cell-mouse-leave="unhighlight"
      >
        <el-table-column align="center" type="index" />
        <el-table-column
          v-for="(item, index) in keys"
          :key="index"
          :prop="item"
          :label="item"
          align="center"
          fit="true"
        />
      </el-table>
    </div>
  </div>
</template>

<script>
import 'element-ui/lib/theme-chalk/index.css';
import * as d3 from 'd3';
import { createNamespacedHelpers } from 'vuex';

const {
  mapMutations: mapHyperparmMutations,
  mapGetters: mapHyperparmGatters,
} = createNamespacedHelpers('Visual/hyperparm');
export default {
  name: 'HyperPara',
  data() {
    return {
      items: [], // 保存数值型数据的名称
      selected: '', // 保存选用的数值类型
      data: [], // 保存所有数据
      keys: [], // 保存所有名称
      choice: true,
      localAxisType: {},
      localSelectedDatas: [],
      width: '',
      tableShow: false,
    };
  },
  computed: {
    ...mapHyperparmGatters([
      'getAllData',
      'getGlobalChange',
      'getKey',
      'getSelected',
      'getAxisType',
      'getGlobalSelectedDatas',
      'getHypEmpty',
      'getMainParams',
    ]),
  },
  watch: {
    getHypEmpty(val) {
      this.tableShow = val;
    },
    selected(newValue) {
      this.drawPara(newValue);
    },
    getAllData(val) {
      if (val.length === 0) {
        this.items = [];
        this.data = [];
        this.keys = [];
        this.selected = '';
        this.localSelectedDatas = [];
        this.setHypEmpty(false);
        return;
      }
      this.setHypEmpty(true);
      this.data = JSON.parse(JSON.stringify(val));
      this.keys = Object.keys(this.data[0]);
      this.items = this.getMainParams;
      this.selected = this.getSelected;
      this.localAxisType = this.getAxisType;
      this.localSelectedDatas = this.data;
      this.drawPara(this.selected);
    },
    getSelected(val) {
      this.selected = val;
    },
    getGlobalChange() {
      this.localAxisType = this.getAxisType;
      this.drawPara(this.selected);
    },
    getGlobalSelectedDatas(val) {
      if (val === '') {
        val = this.getAllData;
      }
      this.localSelectedDatas = JSON.parse(JSON.stringify(val));
    },
  },
  mounted() {
    if (this.getAllData.length !== 0) {
      this.items = this.getMainParams;
      this.keys = Object.keys(this.getAllData[0]);
      this.data = this.getAllData;
      this.selected = this.getSelected;
      this.localAxisType = this.getAxisType;
      this.localSelectedDatas = this.getAllData;
      this.setHypEmpty(true);
    }
  },
  methods: {
    ...mapHyperparmMutations(['setFocusData', 'setGlobalSelectedDatas', 'setHypEmpty']),
    drawPara(colorItem) {
      d3.select('#hp')
        .selectAll('svg')
        .remove();
      if (this.data.length === 0) return;
      this.setGlobalSelectedDatas(this.getAllData);
      const { data } = this;
      const margin = {
        top: 20,
        right: 20,
        bottom: 50,
        left: 20,
      };
      const _this = this;
      const width = 600 - margin.left - margin.right;
      const height = 200 - margin.top - margin.bottom;

      const x = d3
        .scalePoint()
        .rangeRound([0, width])
        .padding(1);
      const y = {};
      const dragging = {};
      const line = d3.line().curve(d3.curveMonotoneX);
      const container = d3.select('#hp').attr('class', 'parcoords');

      const svg = container
        .append('svg')
        .attr('class', 'my_svg')
        .attr('width', '100%')
        .attr('height', '100%')
        .attr('preserveAspectRatio', 'xMidYMid meet')
        .attr('viewBox', `0 0 ${width} 200`)
        .append('g')
        .attr('transform', `translate(${-margin.left},${margin.top})`);

      function quantP(v) {
        return parseFloat(v) === v || v === '';
      }

      const maxData = d3.max(data, (d) => {
        return +d[colorItem];
      });
      const minData = d3.min(data, (d) => {
        return +d[colorItem];
      });
      const dimensions = d3.keys(data[0]);
      x.domain(dimensions);
      dimensions.forEach((d) => {
        const vals = data.map((p) => {
          return p[d];
        });
        if (vals.every(quantP)) {
          const tmp = d3.extent(data, (p) => {
            return +p[d];
          });
          if (_this.localAxisType[d] === 'log') {
            if (tmp[0] === tmp[1]) {
              y[d] = d3
                .scaleLog()
                .domain([tmp[0] * 0.9, tmp[1] * 1.1])
                .range([height - 0.5, 0])
                .nice();
            } else {
              y[d] = d3
                .scaleLog()
                .domain(tmp)
                .range([height - 0.5, 0])
                .nice();
            }
          } else if (tmp[0] === tmp[1]) {
            y[d] = d3
              .scaleLinear()
              .domain([tmp[0] - 1, tmp[1] + 1])
              .range([height, 0])
              .nice();
          } else {
            y[d] = d3
              .scaleLinear()
              .domain(tmp)
              .range([height, 0])
              .nice();
          }
        } else {
          vals.sort();
          y[d] = d3
            .scalePoint()
            .domain(
              vals.filter((v, i) => {
                return vals.indexOf(v) === i;
              })
            )
            .range([height, 0], 1);
        }
      });
      const extents = dimensions.map(() => {
        return [0, 0];
      });
      function position(d) {
        const v = dragging[d];
        return v == null ? x(d) : v;
      }

      function transition(g) {
        return g.transition().duration(500);
      }

      // Returns the path for a given data point.
      function path(d) {
        return line(
          dimensions.map((p) => {
            return [position(p), y[p](d[p])];
          })
        );
      }
      // Add grey background lines for context.
      const background = svg
        .append('g')
        .attr('class', 'background')
        .selectAll('path')
        .data(data)
        .enter()
        .append('path')
        .attr('d', path)
        .attr('fill', 'none');

      const linear = d3
        .scaleLinear()
        .domain([0, 50])
        .range([maxData, minData]);
      const colorMap = d3
        .scaleSequential()
        .domain([maxData, minData])
        .interpolator(d3.interpolateSpectral);
      const foreground = svg
        .append('g')
        .attr('class', 'foreground')
        .selectAll('path')
        .data(data)
        .enter()
        .append('path')
        .attr('d', path)
        .attr('stroke', (d) => {
          return colorMap(d[colorItem]);
        })
        .attr('fill', 'none')
        .on('mouseover', function _nonName(d) {
          this.parentNode.appendChild(this);
          d3.select(this)
            .transition()
            .duration('50')
            .attr('stroke-width', '2px');
          _this.setFocusData(JSON.parse(JSON.stringify(d)));
        })
        .on('mouseout', function _nonName() {
          d3.select(this)
            .transition()
            .duration('50')
            .attr('stroke-width', '1px');
        });

      function goBrush() {
        d3.event.sourceEvent.stopPropagation();
      }

      function localSelectedData() {
        const tempData = [];
        foreground.each((d) => {
          const isTrue = dimensions.every((p, i) => {
            if (extents[i][0] === 0 && extents[i][0] === 0) {
              return true;
            }
            return extents[i][1] <= d[p] && d[p] <= extents[i][0];
          });
          if (isTrue === true) {
            tempData.push(d);
          }
        });
        _this.setGlobalSelectedDatas(JSON.parse(JSON.stringify(tempData)));
      }
      function brushStart(selectionName) {
        foreground.style('display', 'none');

        const dimensionsIndex = dimensions.indexOf(selectionName);

        extents[dimensionsIndex] = [0, 0];

        foreground.style('display', (d) => {
          return dimensions.every((p, i) => {
            if (extents[i][0] === 0 && extents[i][0] === 0) {
              return true;
            }
            return extents[i][1] <= d[p] && d[p] <= extents[i][0];
          })
            ? null
            : 'none';
        });
        localSelectedData();
      }

      // Handles a brush event, toggling the display of foreground lines.
      function brushParallelChart() {
        for (let i = 0; i < dimensions.length; i += 1) {
          if (d3.event.target === y[dimensions[i]].brush) {
            extents[i] = d3.event.selection.map(y[dimensions[i]].invert, y[dimensions[i]]);
          }
        }
        foreground.style('display', (d) => {
          return dimensions.every((p, i) => {
            if (extents[i][0] === 0 && extents[i][0] === 0) {
              return true;
            }
            return extents[i][1] <= d[p] && d[p] <= extents[i][0];
          })
            ? null
            : 'none';
        });
        localSelectedData();
      }

      function brushEnd() {
        if (!d3.event.sourceEvent) return; // Only transition after input.
        if (!d3.event.selection) return; // Ignore empty selections.
        for (let i = 0; i < dimensions.length; i += 1) {
          if (d3.event.target === y[dimensions[i]].brush) {
            extents[i] = d3.event.selection.map(y[dimensions[i]].invert, y[dimensions[i]]);
            d3.select(this)
              .transition()
              .call(d3.event.target.move, extents[i].map(y[dimensions[i]]));
          }
        }
        localSelectedData();
      }
      //   brush for ordinal cases
      function brushParallel() {
        for (let i = 0; i < dimensions.length; i += 1) {
          if (d3.event.target === y[dimensions[i]].brush) {
            const yScale = y[dimensions[i]];
            const selected = yScale.domain().filter((d) => {
              const s = d3.event.selection;
              return s[0] <= yScale(d) && yScale(d) <= s[1];
            });
            const temp = selected.sort();
            extents[i] = [temp[temp.length - 1], temp[0]];
          }
        }
        foreground.style('display', (d) => {
          return dimensions.every((p, i) => {
            if (extents[i][0] === 0 && extents[i][0] === 0) {
              return true;
            }
            return extents[i][1] <= d[p] && d[p] <= extents[i][0];
          })
            ? null
            : 'none';
        });
      }
      function brushEndOrdinal() {
        if (!d3.event.sourceEvent) return; // Only transition after input.
        if (!d3.event.selection) return; // Ignore empty selections.
        for (let i = 0; i < dimensions.length; i += 1) {
          if (d3.event.target === y[dimensions[i]].brush) {
            const yScale = y[dimensions[i]];
            const selected = yScale.domain().filter((d) => {
              const s = d3.event.selection;
              return s[0] <= yScale(d) && yScale(d) <= s[1];
            });
            const temp = selected.sort();
            extents[i] = [temp[temp.length - 1], temp[0]];
            if (selected.length > 1) {
              d3.select(this)
                .transition()
                .call(d3.event.target.move, extents[i].map(y[dimensions[i]]));
            }
          }
        }
        localSelectedData();
      }

      // draw legned
      const minMaxFormat = d3.format('.2f');
      const legendBox = svg
        .append('g')
        .attr('class', 'legend')
        .attr(
          'transform',
          `translate(${x(dimensions[this.keys.length - 1]) + 40},${height +
            margin.top +
            margin.bottom -
            130})`
        );
      legendBox
        .selectAll('rect')
        .data(d3.range(50))
        .enter()
        .append('rect')
        .attr('x', '10px')
        .attr('y', (d, i) => i)
        .attr('width', 7)
        .attr('height', 10)
        .style('fill', (d) => colorMap(linear(d)));

      legendBox
        .append('g')
        .attr('class', 'legendText')
        .append('text')
        .attr('fill', 'black')
        .text(minMaxFormat(minData))
        .attr('x', '8px')
        .attr('y', '67px');

      legendBox
        .append('g')
        .attr('class', 'legendText')
        .append('text')
        .attr('fill', 'black')
        .text(minMaxFormat(maxData))
        .attr('x', '8px')
        .attr('y', '-3px');
      // Add a group element for each dimension.
      const g = svg
        .selectAll('.dimension')
        .data(dimensions)
        .enter()
        .append('g')
        .attr('class', 'dimension')
        .attr('transform', (d) => {
          return `translate(${x(d)})`;
        })
        .call(
          d3
            .drag()
            .subject((d) => {
              return {
                x: x(d),
              };
            })
            .on('start', (d) => {
              dragging[d] = x(d);
              background.attr('visibility', 'hidden');
            })
            .on('drag', (d) => {
              dragging[d] = Math.min(width, Math.max(0, d3.event.x));
              foreground.attr('d', path);
              dimensions.sort((a, b) => {
                return position(a) - position(b);
              });
              x.domain(dimensions);
              g.attr('transform', (d) => {
                return `translate(${position(d)})`;
              });
            })
            .on('end', function _nonName(d) {
              delete dragging[d];
              transition(d3.select(this)).attr('transform', `translate(${x(d)})`);
              transition(foreground).attr('d', path);
              background
                .attr('d', path)
                .transition()
                .delay(500)
                .duration(0)
                .attr('visibility', null);
            })
        );
      // Add an axis and title.
      const g2 = svg.selectAll('.dimension');
      g2.append('g')
        .attr('class', 'axis')
        .each(function _nonName(d) {
          d3.select(this).call(d3.axisLeft(y[d]).tickSize(3));
        })
        // text does not show up because previous line breaks somehow
        .append('text')
        .attr('class', 'axisText')
        .attr('fill', 'white')
        .style('text-anchor', 'middle')
        .attr('y', height + 18)
        .text((d) => {
          return d;
        });
      d3.selectAll('.dimension').each(function _nonName(d) {
        const t = d3
          .select(this)
          .select('text.axisText')
          .node()
          .getBBox();
        d3.select(this)
          .append('rect')
          .attr('class', 'backgroundRect')
          .attr('x', () => {
            return -(t.width + 10) / 2;
          })
          .attr('y', height + 10)
          .attr('height', t.height + 6)
          .attr('width', t.width + 9)
          .attr('rx', 3)
          .attr('ry', 3);
        d3.select(this)
          .append('text')
          .attr('class', 'backgroundText')
          .attr('fill', 'rgb(44, 99, 155)')
          .attr('font-size', '6px')
          .style('text-anchor', 'middle')
          .attr('y', height + 18)
          .text(d);
      });
      // Add and store a brush for each axis.
      g2.append('g')
        .attr('class', 'brush')
        .each(function _nonName(d) {
          if (_this.items.indexOf(d) <= -1) {
            d3.select(this).call(
              (y[d].brush = d3
                .brushY()
                .extent([
                  [-4, 0],
                  [4, height],
                ])
                .on('brush start', brushStart)
                .on('brush', goBrush)
                .on('brush', brushParallel)
                .on('end', brushEndOrdinal))
            );
          } else {
            d3.select(this).call(
              (y[d].brush = d3
                .brushY()
                .extent([
                  [-4, 0],
                  [4, height],
                ])
                .on('brush start', brushStart)
                .on('brush', goBrush)
                .on('brush', brushParallelChart)
                .on('end', brushEnd))
            );
          }
        })
        .selectAll('rect')
        .attr('x', -8)
        .attr('width', 16);
    },
    getRowDatas(row) {
      const selectedRowData = JSON.parse(JSON.stringify(row));
      this.highlight(selectedRowData);
    },
    unhighlight(d) {
      d3.select('.foreground')
        .selectAll('path')
        .each(function _nonName(s) {
          let orderedD = {};
          let orderedS = {};
          Object.keys(d)
            .sort()
            .forEach((key) => {
              orderedD[key] = d[key];
              orderedS[key] = s[key];
            });
          orderedS = JSON.stringify(orderedS);
          orderedD = JSON.stringify(orderedD);
          if (orderedS === orderedD) {
            d3.select(this).attr('stroke-width', '1px');
          }
        });
    },
    highlight(d) {
      this.setFocusData(JSON.parse(JSON.stringify(d)));
      d3.select('.foreground')
        .selectAll('path')
        .each(function _nonName(s) {
          let orderedD = {};
          let orderedS = {};
          Object.keys(d)
            .sort()
            .forEach((key) => {
              orderedD[key] = d[key];
              orderedS[key] = s[key];
            });
          orderedS = JSON.stringify(orderedS);
          orderedD = JSON.stringify(orderedD);
          if (orderedS === orderedD) {
            this.parentNode.appendChild(this);
            d3.select(this).attr('stroke-width', '2px');
          }
        });
    },
    tableRowClassName({ row }) {
      const d = JSON.stringify(row);
      const selectedDatas = JSON.parse(JSON.stringify(this.localSelectedDatas));
      for (let i = 0; i < selectedDatas.length; i += 1) {
        const t = JSON.stringify(selectedDatas[i]);
        if (t === d) {
          return 'success-row';
        }
      }
      return '';
    },
    leftTransform(leftDim, dimensions) {
      d3.selectAll('.dimension').each(function _nonName(d) {
        if (dimensions[0] === d) {
          d3.select(this).attr('transform', `translate(${leftDim})`);
        }
      });
    },
  },
};
</script>

<style lang="less" scroped>
svg {
  font: sans-serif;
}

#select {
  float: left;
  width: 100px;
  height: 100%;
  background: -o-linear-gradient();
  background-color: cadetblue;
}

#hp {
  float: left;
  width: 100%;
  height: 100%;
}

.background path {
  opacity: 0.3;
  fill: none;
  stroke: #ddd;
  stroke-width: 1px;
  shape-rendering: optimizeQuality;
}

.foreground path {
  fill: none;
  shape-rendering: optimizeQuality;
}

.brush .extent {
  fill-opacity: 0.3;
  stroke: #fff;
  shape-rendering: auto;
}

.axis line,
.axis path {
  fill: none;
  stroke: #000;
  shape-rendering: geometricPrecision;
}

.axis text {
  font-size: 5px;
  cursor: move;
}

.backgroundRect {
  cursor: move;
  opacity: 1;
  fill: rgb(233, 236, 253);
}

.backgroundText {
  cursor: move;
}

.legendText text {
  font-size: 6px;
  text-shadow: 0 1px 0 #fff, 1px 0 0 #fff, 0 -1px 0 #fff, -1px 0 0 #fff;
}

.el-table .success-row {
  background: rgb(245, 247, 250);
}

#grid {
  padding-top: 10%;
  padding-right: 10%;
  padding-bottom: 2%;
  padding-left: 10%;
}
</style>
