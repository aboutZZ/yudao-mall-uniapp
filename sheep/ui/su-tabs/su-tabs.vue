<template>
  <view class="u-tabs">
    <view class="u-tabs__wrapper">
      <slot name="left" />
      <view class="u-tabs__wrapper__scroll-view-wrapper">
        <scroll-view
          :scroll-x="scrollable"
          :scroll-left="scrollLeft"
          scroll-with-animation
          enable-flex
          class="u-tabs__wrapper__scroll-view white-space"
          :show-scrollbar="false"
          ref="u-tabs__wrapper__scroll-view"
        >
          <view class="u-tabs__wrapper__nav" ref="u-tabs__wrapper__nav">
            <view
              class="u-tabs__wrapper__nav__item"
              v-for="(item, index) in list"
              :key="index"
              @tap="clickHandler(item, index)"
              :ref="`u-tabs__wrapper__nav__item-${index}`"
              :style="[addStyle(itemStyle), { flex: scrollable ? '' : 1 }]"
              :class="[
                `u-tabs__wrapper__nav__item-${index}`,
                item.disabled && 'u-tabs__wrapper__nav__item--disabled',
              ]"
            >
              <text
                :class="[item.disabled && 'u-tabs__wrapper__nav__item__text--disabled']"
                class="u-tabs__wrapper__nav__item__text"
                :style="[textStyle(index)]"
                >{{ item[keyName] }}</text
              >
            </view>
            <!-- #ifdef APP-NVUE -->
            <view
              class="u-tabs__wrapper__nav__line"
              ref="u-tabs__wrapper__nav__line"
              :style="[
                {
                  width: addUnit(lineWidth),
                  height: addUnit(lineHeight),
                  background: lineColor ? lineColor : 'var(--ui-BG-Main)',
                  backgroundSize: lineBgSize,
                },
              ]"
            ></view>
            <!-- #endif -->
            <!-- #ifndef APP-NVUE -->
            <view
              class="u-tabs__wrapper__nav__line"
              ref="u-tabs__wrapper__nav__line"
              :style="[
                {
                  width: addUnit(lineWidth),
                  transform: `translate(${lineOffsetLeft}px)`,
                  transitionDuration: `${firstTime ? 0 : duration}ms`,
                  height: addUnit(lineHeight),
                  background: lineColor ? lineColor : 'var(--ui-BG-Main)',
                  backgroundSize: lineBgSize,
                },
              ]"
            >
            </view>
            <!-- #endif -->
          </view>
        </scroll-view>
      </view>
      <slot name="right" />
    </view>
  </view>
</template>

<script>
  import { deepMerge, addStyle, addUnit, sleep, getPx } from '@/sheep/helper';
  // #ifdef APP-NVUE
  const animation = uni.requireNativePlugin('animation');
  const dom = uni.requireNativePlugin('dom');
  // #endif

  /**
   * Tabs 标签
   * @description tabs标签组件，在标签多的时候，可以配置为左右滑动，标签少的时候，可以禁止滑动。 该组件的一个特点是配置为滚动模式时，激活的tab会自动移动到组件的中间位置。
   * @tutorial https://www.uviewui.com/components/tabs.html
   * @property {String | Number}	duration			滑块移动一次所需的时间，单位秒（默认 200 ）
   * @property {String | Number}	swierWidth			swiper的宽度（默认 '750rpx' ）
   * @property {String}	keyName	 从`list`元素对象中读取的键名（默认 'name' ）
   * @event {Function(index)} change 标签改变时触发 index: 点击了第几个tab，索引从0开始
   * @event {Function(index)} click 点击标签时触发 index: 点击了第几个tab，索引从0开始
   * @example <u-tabs :list="list" :is-scroll="false" :current="current" @change="change"></u-tabs>
   */
  export default {
    name: 'su-tabs',
    data() {
      return {
        addStyle,
        addUnit,
        firstTime: true,
        scrollLeft: 0,
        scrollViewWidth: 0,
        lineOffsetLeft: 0,
        tabsRect: {
          left: 0,
        },
        innerCurrent: 0,
        moving: false,
      };
    },
    props: {
      // 滑块的移动过渡时间，单位ms
      duration: {
        type: Number,
        default: 300,
      },
      // tabs标签数组
      list: {
        type: Array,
        default: [],
      },
      // 滑块颜色
      lineColor: {
        type: String,
        default: '',
      },
      // 菜单选择中时的样式
      activeStyle: {
        type: [String, Object],
        default() {
          return {
            color: '#303133',
          };
        },
      },
      // 菜单非选中时的样式
      inactiveStyle: {
        type: [String, Object],
        default() {
          return {
            color: '#606266',
          };
        },
      },
      // 滑块长度
      lineWidth: {
        type: [String, Number],
        default: 20,
      },
      // 滑块高度
      lineHeight: {
        type: [String, Number],
        default: 3,
      },
      // 滑块背景显示大小，当滑块背景设置为图片时使用
      lineBgSize: {
        type: String,
        default: 'cover',
      },
      // 菜单item的样式
      itemStyle: {
        type: [String, Object],
        default() {
          return {
            height: '44px',
          };
        },
      },
      // 菜单是否可滚动
      scrollable: {
        type: Boolean,
        default: true,
      },
      // 当前选中标签的索引
      current: {
        type: [Number, String],
        default: 0,
      },
      // 默认读取的键名
      keyName: {
        type: String,
        default: 'name',
      },
    },
    watch: {
      current: {
        immediate: true,
        handler(newValue, oldValue) {
          // 内外部值不相等时，才尝试移动滑块
          if (newValue !== this.innerCurrent) {
            this.innerCurrent = newValue;
            this.$nextTick(() => {
              this.resize();
            });
          }
        },
      },
      // list变化时，重新渲染list各项信息
      list() {
        this.$nextTick(() => {
          this.resize();
        });
      },
    },
    computed: {
      textStyle() {
        return (index) => {
          const style = {};
          // 取当期是否激活的样式
          const customeStyle =
            index === this.innerCurrent ? addStyle(this.activeStyle) : addStyle(this.inactiveStyle);
          // 如果当前菜单被禁用，则加上对应颜色，需要在此做处理，是因为nvue下，无法在style样式中通过!import覆盖标签的内联样式
          if (this.list[index].disabled) {
            style.color = '#c8c9cc';
          }
          return deepMerge(customeStyle, style);
        };
      },
    },
    async mounted() {
      this.init();
    },
    methods: {
      $uGetRect(selector, all) {
        return new Promise((resolve) => {
          uni
            .createSelectorQuery()
            .in(this)
            [all ? 'selectAll' : 'select'](selector)
            .boundingClientRect((rect) => {
              if (all && Array.isArray(rect) && rect.length) {
                resolve(rect);
              }
              if (!all && rect) {
                resolve(rect);
              }
            })
            .exec();
        });
      },
      setLineLeft() {
        const tabItem = this.list[this.innerCurrent];
        if (!tabItem) {
          return;
        }
        // 获取滑块该移动的位置
        let lineOffsetLeft = this.list
          .slice(0, this.innerCurrent)
          .reduce((total, curr) => total + curr.rect.width, 0);
        // 获取下划线的数值px表示法
        const lineWidth = getPx(this.lineWidth);
        this.lineOffsetLeft = lineOffsetLeft + (tabItem.rect.width - lineWidth) / 2;
        // #ifdef APP-NVUE
        // 第一次移动滑块，无需过渡时间
        this.animation(this.lineOffsetLeft, this.firstTime ? 0 : parseInt(this.duration));
        // #endif

        // 如果是第一次执行此方法，让滑块在初始化时，瞬间滑动到第一个tab item的中间
        // 这里需要一个定时器，因为在非nvue下，是直接通过style绑定过渡时间，需要等其过渡完成后，再设置为false(非第一次移动滑块)
        if (this.firstTime) {
          setTimeout(() => {
            this.firstTime = false;
          }, 10);
        }
      },
      // nvue下设置滑块的位置
      animation(x, duration = 0) {
        // #ifdef APP-NVUE
        const ref = this.$refs['u-tabs__wrapper__nav__line'];
        animation.transition(ref, {
          styles: {
            transform: `translateX(${x}px)`,
          },
          duration,
        });
        // #endif
      },
      // 点击某一个标签
      clickHandler(item, index) {
        // 因为标签可能为disabled状态，所以click是一定会发出的，但是change事件是需要可用的状态才发出
        this.$emit('click', {
          ...item,
          index,
        });
        // 如果disabled状态，返回
        if (item.disabled) return;
        this.innerCurrent = index;
        this.resize();
        this.$emit('change', {
          ...item,
          index,
        });
      },
      init() {
        sleep().then(() => {
          this.resize();
        });
      },
      setScrollLeft() {
        // 当前活动tab的布局信息，有tab菜单的width和left(为元素左边界到父元素左边界的距离)等信息
        const tabRect = this.list[this.innerCurrent];
        // 累加得到当前item到左边的距离
        const offsetLeft = this.list.slice(0, this.innerCurrent).reduce((total, curr) => {
          return total + curr.rect.width;
        }, 0);
        // 此处为屏幕宽度
        const windowWidth = uni.getWindowInfo().windowWidth;
        // 将活动的tabs-item移动到屏幕正中间，实际上是对scroll-view的移动
        let scrollLeft =
          offsetLeft -
          (this.tabsRect.width - tabRect.rect.width) / 2 -
          (windowWidth - this.tabsRect.right) / 2 +
          this.tabsRect.left / 2;
        // 这里做一个限制，限制scrollLeft的最大值为整个scroll-view宽度减去tabs组件的宽度
        scrollLeft = Math.min(scrollLeft, this.scrollViewWidth - this.tabsRect.width);
        this.scrollLeft = Math.max(0, scrollLeft);
      },
      // 获取所有标签的尺寸
      resize() {
        // 如果不存在list，则不处理
        if (this.list.length === 0) {
          return;
        }
        Promise.all([this.getTabsRect(), this.getAllItemRect()]).then(
          ([tabsRect, itemRect = []]) => {
            this.tabsRect = tabsRect;
            this.scrollViewWidth = 0;
            itemRect.map((item, index) => {
              // 计算scroll-view的宽度，这里
              this.scrollViewWidth += item.width;
              // 另外计算每一个item的中心点X轴坐标
              this.list[index].rect = item;
            });
            // 获取了tabs的尺寸之后，设置滑块的位置
            this.setLineLeft();
            this.setScrollLeft();
          },
        );
      },
      // 获取导航菜单的尺寸
      getTabsRect() {
        return new Promise((resolve) => {
          this.queryRect('u-tabs__wrapper__scroll-view').then((size) => resolve(size));
        });
      },
      // 获取所有标签的尺寸
      getAllItemRect() {
        return new Promise((resolve) => {
          const promiseAllArr = this.list.map((item, index) =>
            this.queryRect(`u-tabs__wrapper__nav__item-${index}`, true),
          );
          Promise.all(promiseAllArr).then((sizes) => resolve(sizes));
        });
      },
      // 获取各个标签的尺寸
      queryRect(el, item) {
        // #ifndef APP-NVUE
        // $uGetRect为uView自带的节点查询简化方法，详见文档介绍：https://www.uviewui.com/js/getRect.html
        // 组件内部一般用this.$uGetRect，对外的为uni.$u.getRect，二者功能一致，名称不同
        return new Promise((resolve) => {
          this.$uGetRect(`.${el}`).then((size) => {
            resolve(size);
          });
        });
        // #endif

        // #ifdef APP-NVUE
        // nvue下，使用dom模块查询元素高度
        // 返回一个promise，让调用此方法的主体能使用then回调
        return new Promise((resolve) => {
          dom.getComponentRect(item ? this.$refs[el][0] : this.$refs[el], (res) => {
            resolve(res.size);
          });
        });
        // #endif
      },
    },
  };
</script>

<style lang="scss" scoped>
  .u-tabs {
    background: #fff;
    border-bottom: 2rpx solid #eee;

    &__wrapper {
      @include flex;
      align-items: center;

      &__scroll-view-wrapper {
        flex: 1;
        /* #ifndef APP-NVUE */
        overflow: auto hidden;
        /* #endif */
      }

      &__nav {
        @include flex;
        position: relative;

        &__item {
          padding: 0 11px;
          @include flex;
          align-items: center;
          justify-content: center;

          &--disabled {
            /* #ifndef APP-NVUE */
            cursor: not-allowed;
            /* #endif */
          }

          &__text {
            font-size: 14px;
            color: #606266;
            white-space: nowrap !important;

            &--disabled {
              color: #c8c9cc !important;
            }
          }
        }

        &__line {
          height: 3px;
          background: #3c9cff;
          width: 30px;
          position: absolute;
          bottom: 2px;
          border-radius: 100px;
          transition-property: transform;
          transition-duration: 300ms;
        }
      }
    }
  }
</style>
