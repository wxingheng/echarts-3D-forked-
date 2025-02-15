<template>
    <div
     ref="chartPanel"
     id="chart-panel"
     style="width: 100%; height: 388px"
 ></div>
</template>

<script>
import * as echarts from "echarts";
import "echarts-gl";
// 生成模拟 3D 饼图的配置项
// pieData（object）：饼图数据
// internalDiameterRatio（0~1之间的浮点数）：内径/外径的值（默认值 1/2），当该值等于 0 时，
// heigth配置每个数据生成的高度
/**
 * 生成3D饼图的核心配置函数
 * @param {Array} pieData - 饼图数据数组
 * @param {Number} internalDiameterRatio - 内径/外径的比值(0~1之间)
 * @param {Number} height - 控制3D饼图的高度系数
 */
export function getPie3D(pieData, internalDiameterRatio, height) {
    let series = [];  // 存储每个扇形的系列配置
    let sumValue = 0; // 数据总和
    let startValue = 0; // 起始值，用于计算每个扇形的起始角度
    let endValue = 0;   // 结束值，用于计算每个扇形的结束角度
    let legendData = []; // 图例数据数组
    
    // 计算内外径比例系数k
    let k = typeof internalDiameterRatio !== "undefined"
        ? (1 - internalDiameterRatio) / (1 + internalDiameterRatio)
        : 1 / 3;

    // 计算所有数据的总和，用于后续计算比例
    let total = 0;
    for (let i = 0; i < pieData.length; i++) {
        pieData[i].value = Number(pieData[i].value);
        total += pieData[i].value;
    }

    // 计算每个数据项的占比
    for (let i = 0; i < pieData.length; i++) {
        pieData[i].proportion = parseFloat(pieData[i].value / total).toFixed(4);
    }

    // 为每个扇形生成surface配置
    for (let i = 0; i < pieData.length; i++) {
        sumValue += pieData[i].value;
        
        // 构建每个扇形的基础配置
        let seriesItem = {
            name: typeof pieData[i].name === "undefined" ? `series${i}` : pieData[i].name,
            type: "surface", // 使用surface类型来实现3D效果
            parametric: true, // 使用参数方程
            wireframe: {
                show: false, // 不显示网格线
            },
            pieData: pieData[i],
            pieStatus: {
                selected: pieData[i].selected ? pieData[i].selected : false,
                hovered: pieData[i].hovered ? pieData[i].hovered : false,
                k: k,
            },
        };

        // 设置扇形的样式
        if (typeof pieData[i].itemStyle != "undefined") {
            let itemStyle = {};
            if (typeof pieData[i].itemStyle.color != "undefined") {
                itemStyle.color = pieData[i].itemStyle.color;
            }
            if (typeof pieData[i].itemStyle.opacity != "undefined") {
                itemStyle.opacity = pieData[i].itemStyle.opacity;
            }
            seriesItem.itemStyle = itemStyle;
        }
        
        series.push(seriesItem);
    }

    // 计算每个扇形的起始比例和结束比例，并生成参数方程
    for (let i = 0; i < series.length; i++) {
        endValue = startValue + series[i].pieData.value;
        series[i].pieData.startRatio = startValue / sumValue;
        series[i].pieData.endRatio = endValue / sumValue;
        
        // 生成扇形的参数方程，这决定了扇形的形状和高度
        series[i].parametricEquation = getParametricEquation(
            series[i].pieData.startRatio,
            series[i].pieData.endRatio,
            series[i].pieStatus.selected,
            series[i].pieStatus.hovered,
            k,
            series[i].pieData.value * height  // 高度与数值成正比
        );
        
        startValue = endValue;
        legendData.push(series[i].name);
    }

    return series;
}

// startRatio（浮点数）: 当前扇形起始比例，取值区间[0, endRatio)
// endRatio（浮点数）: 当前扇形结束比例，取值区间(startRatio, 1]
// isSelected（布尔值）: 是否选中，效果参照二维饼图选中效果（单选）
// isHovered（布尔值）: 是否放大，效果接近二维饼图高亮（放大）效果（未能实现阴影）
// k（0~1之间的浮点数）：用于参数方程的一个参数，取值 0~1 之间，通过「内径 / 外径」的值换算而来。
//height配置3d扇形高度
/**
 * 生成3D扇形的参数方程
 * @param {Number} startRatio - 扇形的起始比例
 * @param {Number} endRatio - 扇形的结束比例
 * @param {Boolean} isSelected - 是否被选中
 * @param {Boolean} isHovered - 是否处于悬停状态
 * @param {Number} k - 内外径比例系数
 * @param {Number} height - 扇形的高度
 */
export function getParametricEquation(
    startRatio,
    endRatio,
    isSelected,
    isHovered,
    k,
    height
) {
    // 计算
    let midRatio = (startRatio + endRatio) / 2;

    let startRadian = startRatio * Math.PI * 2;
    let endRadian = endRatio * Math.PI * 2;
    let midRadian = midRatio * Math.PI * 2;

    // 如果只有一个扇形，则不实现选中效果。
    if (startRatio === 0 && endRatio === 1) {
        isSelected = false;
    }

    // 通过扇形内径/外径的值，换算出辅助参数 k（默认值 1/3）
    k = typeof k !== "undefined" ? k : 1 / 3;

    // 计算选中效果分别在 x 轴、y 轴方向上的位移（未选中，则位移均为 0）
    let offsetX = isSelected ? Math.cos(midRadian) * 0.2 : 0;
    let offsetY = isSelected ? Math.sin(midRadian) * 0.2 : 0;

    // 计算高亮效果的放大比例（未高亮，则比例为 1）
    let hoverRate = isHovered ? 1.05 : 1;

    // 返回曲面参数方程
    return {
        u: {
            min: -Math.PI,
            max: Math.PI * 3,
            step: Math.PI / 32,
        },

        v: {
            min: 0,
            max: Math.PI * 2,
            step: Math.PI / 20,
        },

        x: function (u, v) {
            if (u < startRadian) {
                return (
                    offsetX +
                    Math.cos(startRadian) * (1 + Math.cos(v) * k) * hoverRate
                );
            }
            if (u > endRadian) {
                return (
                    offsetX +
                    Math.cos(endRadian) * (1 + Math.cos(v) * k) * hoverRate
                );
            }
            return offsetX + Math.cos(u) * (1 + Math.cos(v) * k) * hoverRate;
        },

        y: function (u, v) {
            if (u < startRadian) {
                return (
                    offsetY +
                    Math.sin(startRadian) * (1 + Math.cos(v) * k) * hoverRate
                );
            }
            if (u > endRadian) {
                return (
                    offsetY +
                    Math.sin(endRadian) * (1 + Math.cos(v) * k) * hoverRate
                );
            }
            return offsetY + Math.sin(u) * (1 + Math.cos(v) * k) * hoverRate;
        },

        z: function (u, v) {
            if (u < -Math.PI * 0.5) {
                return Math.sin(u);
            }
            if (u > Math.PI * 2.5) {
                return Math.sin(u);
            }
            return Math.sin(v) > 0 ? 1 * height : -1;
        },
    };
}


export default {
  name: "App",
  components: {},
  data() {
    return {
      chart: null,
  optionData: [
        {
          name: "合适的表扬",
          value: 50,
          itemStyle: {
            opacity: 0.2,
            color: "#D6476C",
          },
        },
        {
          name: "过分的表扬",
          value: 35,
          itemStyle: {
            opacity: 0.2,
            color: "#017DC1",
          },
        },
        {
          name: "反馈支持",
          value: 15,
          itemStyle: {
            opacity: 0.2,
            color: "#804BC6",
          },
        },
      ],
    };
  },
  mounted() {
    this.draw3d();
    this.$nextTick(() => {
      let parent = document.getElementById("chart-panel"); // 获取父元素
      let canvas = parent.getElementsByTagName("canvas"); // 获取父元素下面的所有canvas元素
      console.log(canvas);
      canvas[1].style.transform = "rotateX(30deg)";
    });
  },
  methods: {

  draw3d() {
      // 基于准备好的dom，初始化echarts实例
      let chartPanel = echarts.init(document.getElementById("chart-panel"));
      for (let i = 0; i < this.optionData.length; i++) {
        delete this.optionData[i].itemStyle.opacity;
      }
      
      // 定义legendData数组
      let legendData = this.optionData.map(item => item.name);
      
      // 传入数据生成 option
      let series = getPie3D(this.optionData, 2, 0.05 );  
      let option = {
        tooltip: {
          formatter: (params) => {
            // console.log(params)
            if (
              params.seriesName !== "mouseoutSeries" &&
              params.seriesName !== "pie2d"
            ) {
              return `<div style="padding:0 10px">${params.seriesName}：${(
                option.series[params.seriesIndex].pieData.proportion * 100
              ).toFixed(2)}%</div>`;
            }
          },
        },
        legend: {
          show: true,
          data: legendData,  // 使用我们定义的legendData
          width: "90%",
          itemGap: 25,
          bottom: 0,
          textStyle: {
            color: "#fff",
            fontSize: 14
          },
          itemWidth: 10,
          itemHeight: 10,
          icon: 'circle'
        },
        xAxis3D: {
          min: -1,
          max: 1,
        },
        yAxis3D: {
          min: -1,
          max: 1,
        },
        zAxis3D: {
          min: -1,
          max: 1,
        },
        grid3D: {
          show: false, //是否显示三维笛卡尔坐标系。
          boxHeight: 20, //三维笛卡尔坐标系在三维场景中的高度
          top: "-12.5%",
          // bottom: "80%",
          // environment: "#021041", //背景
          viewControl: {
            //用于鼠标的旋转，缩放等视角控制
            alpha: 50, //角度
            distance: 250, //调整视角到主体的距离，类似调整zoom 重要
            rotateSensitivity: 0, //设置为0无法旋转
            zoomSensitivity: 0, //设置为0无法缩放
            panSensitivity: 0, //设置为0无法平移
            autoRotate: false, //自动旋转
          },
        },
        series: series,
      };
      chartPanel.setOption(option);

      //是否需要label指引线，如果要就添加一个透明的2d饼状图并调整角度使得labelLine和3d的饼状图对齐，并再次setOption
      option.series.push({
        name: "pie2d",
        type: "pie",
        label: {
          color: "#fff",
          fontSize: 16,
          formatter: (item) => {
            return [
              `{name|${item.data.name}}`,
              `{value|${item.data.value}家}`
            ].join('\n');
          },
          rich: {
            name: {
              fontSize: 14,
              color: '#ffffff',
              padding: [0, 0, 5, 0]
            },
            value: {
              fontSize: 12,
              color: '#0f0',
              padding: [5, 0, 0, 0],
              borderBottom: '1px solid #fff'
            }
          },
          backgroundColor: 'rgba(0,0,0,0.3)',
          borderRadius: 4,
          padding: [5, 10]
        },
        labelLine: {
          length: 25,
          length2: 25,
          lineStyle: {
            color: "rgba(255,255,255,0.6)",
            width: 1
          }
        },
        startAngle: 321, //起始角度，支持范围[0, 360]。 //重要
        clockwise: false, //饼图的扇区是否是顺时针排布。上述这两项配置主要是为了对齐3d的样式
        radius: ["25%", "50%"],
        center: ["50%", "50%"],
        data: this.optionData,
        itemStyle: {
          opacity: 0,
        },
        top: "-20%",
        avoidLabelOverlap: true, //防止标签重叠
      });
      chartPanel.setOption(option);
   }
  },
};
</script>

<style>
#app {
  width: 300px;
  height: 300px;
}
</style>

