<template>
    <div class="m-slider" ref="slider">
        <div class="slider-wrapper" ref="warpper"
             :class="direction == 'vertical' ? 'slider-wrapper-vertical' : ''"
             :style="dragStyleObject"
        >
            <div class="slider-item" :style="itemHeight" v-html="lastItem"></div>
            <slot></slot>
            <div class="slider-item" :style="itemHeight" v-html="firtstItem"></div>
        </div>
        <div class="slider-pagination" :class="direction == 'vertical' ? 'slider-pagination-vertical' : ''">
            <span class="slider-pagination-item"
                  v-for="(t, i) in itemNums"
                  :class="paginationIndex == i ? 'slider-pagination-item-active': ''"
            ></span>
        </div>
    </div>
</template>

<script type="text/babel">
    export default {
        name: 'yd-slider',
        data() {
            return {
                firtstItem: '',
                lastItem: '',
                itemNums: 0,
                index: 1,
                itemsArr: [],
                autoPlayTimer: null,
                paginationIndex: 0,
                itemHeight: {
                    height: null
                },
                dragStyleObject: {
                    transform: 0,
                    speed: 0
                },
                touches: {
                    moveTag: 0,
                    moveOffset: 0,
                    touchStartTime: 0,
                    isTouchEvent: false,
                    allowClick: false,
                    isDraging: false
                }
            }
        },
        props: {
            speed: {
                default: 300,
                validator(val) {
                    return /^\d*$/.test(val);
                }
            },
            autoplay: {
                default: 0,
                validator(val) {
                    return /^\d*$/.test(val);
                }
            },
            direction: {
                validator(val) {
                    return ['horizontal', 'vertical'].indexOf(val) > -1;
                },
                default: 'horizontal'
            }
        },
        watch: {
            index() {
                const index = this.index;
                const itemNums = this.itemNums;
                const tm = (index - 1) % itemNums;
                this.paginationIndex = tm < 0 ? itemNums - 1 : tm;
            }
        },
        methods: {
            init() {
                this.destroy();

                this.isVertical = this.direction == 'vertical';

                this.itemsArr = this.$children.filter(item => item.$options.name === 'yd-slider-item');

                this.itemNums = this.itemsArr.length;

                if (this.isVertical) {
                    this.$refs.slider.style.height = '100%';
                    const height = this.$el.clientHeight;
                    this.itemHeight.height = height + 'px';
                    this.setTranslate(0, -height);
                    this.itemsArr.forEach((item) => {
                        item.$el.style.height = height + 'px';
                    });
                } else {
                    this.setTranslate(0, -this.$refs.warpper.offsetWidth);
                }
                this.cloneItem();
                this.bindEvents();

                this.autoPlay();
            },
            cloneItem() {
                const itemArr = this.itemsArr;

                this.firtstItem = itemArr[0].$el.innerHTML;
                this.lastItem = itemArr[itemArr.length - 1].$el.innerHTML;
            },
            touchStartHandler(event) {
                const touches = this.touches;

                touches.allowClick = true;

                touches.isTouchEvent = event.type === 'touchstart';

                if (!touches.isTouchEvent && 'which' in event && event.which === 3) return;

                if (touches.moveTag == 0) {
                    touches.moveTag = 1;

                    touches.startX = event.touches ? event.touches[0].clientX : event.clientX;
                    touches.startY = event.touches ? event.touches[0].clientY : event.clientY;

                    touches.touchStartTime = Date.now();

                    const itemNums = this.itemNums;

                    if (this.index == 0) {
                        this.index = itemNums;
                        this.setTranslate(0, -itemNums * (this.isVertical ? this.$el.clientHeight : this.$refs.warpper.offsetWidth));
                        return;
                    }

                    if (this.index > itemNums) {
                        this.index = 1;
                        this.setTranslate(0, this.isVertical ? -this.$el.clientHeight : -this.$refs.warpper.offsetWidth);
                    }
                }
            },
            touchMoveHandler(event) {
                if (!this.supportTouch || this.isVertical) {
                    event.preventDefault();
                }

                const touches = this.touches;

                touches.allowClick = false;

                if (touches.isTouchEvent && event.type === 'mousemove') return;

                const currentY = event.touches ? event.touches[0].clientY : event.clientY;
                const currentX = event.touches ? event.touches[0].clientX : event.clientX;

                const touchAngle = Math.atan2(Math.abs(currentY - touches.startY), Math.abs(currentX - touches.startX)) * 180 / Math.PI;

                if ((!this.isVertical ? touchAngle > 45 : (90 - touchAngle > 45)) && this.supportTouch) {
                    touches.moveTag = 3;
                    this.stopAutoplay();
                    this.setTranslate(0, -this.index * (this.isVertical ? this.$el.clientHeight : this.$refs.warpper.offsetWidth));
                    return;
                }

                touches.isDraging = true;

                const deltaSlide = touches.moveOffset = this.isVertical ? (currentY - touches.startY) : (currentX - touches.startX);

                if (deltaSlide != 0 && touches.moveTag != 0) {
                    if (touches.moveTag == 1) {
                        this.stopAutoplay();
                        touches.moveTag = 2;
                    }

                    if (touches.moveTag == 2) {
                        this.setTranslate(0, -this.index * (this.isVertical ? this.$el.clientHeight : this.$refs.warpper.offsetWidth) + deltaSlide);
                    }
                }
            },
            touchEndHandler() {
                const touches = this.touches;
                const moveOffset = touches.moveOffset;
                const warpperSize = this.isVertical ? this.$el.clientHeight : this.$refs.warpper.offsetWidth;

                if (touches.moveTag == 1) {
                    touches.moveTag = 0;
                }

                setTimeout(() => {
                    touches.allowClick = true;
                touches.isDraging = false;
            }, this.speed);

                if (touches.moveTag == 2) {
                    touches.moveTag = 0;

                    const timeDiff = Date.now() - touches.touchStartTime;

                    if (timeDiff > 300 && Math.abs(moveOffset) <= warpperSize * .5) {
                        this.setTranslate(this.speed, -this.index * warpperSize);
                    } else {
                        this.setTranslate(this.speed, -((moveOffset > 0 ? --this.index : ++this.index) * warpperSize));
                    }
                    this.autoPlay();
                    return;
                }

                if (touches.moveTag == 3) {
                    touches.moveTag = 0;
                    this.autoPlay();
                }
            },
            autoPlay() {
                if (this.autoplay <= 0) return;

                this.autoPlayTimer = setInterval(() => {
                    const size = this.isVertical ? this.$el.clientHeight : this.$refs.warpper.offsetWidth;

                    if (this.index > this.itemNums) {
                        this.index = 1;
                        this.setTranslate(0, -size);
                        setTimeout(() => {
                            this.setTranslate(this.speed, -(++this.index * size));
                        }, 100);
                        return;
                    }
                    this.setTranslate(this.speed, -(++this.index * size));

                }, this.autoplay);
            },
            stopAutoplay() {
                clearInterval(this.autoPlayTimer);
            },
            stopDrag(event) {
                this.touches.isDraging && event.preventDefault();
            },
            bindEvents() {
                const _events = this.touchEvents();

                this.$el.addEventListener(_events.start, this.touchStartHandler);
                this.$el.addEventListener(_events.move, this.touchMoveHandler);
                this.$el.addEventListener(_events.end, this.touchEndHandler);

                this.$el.addEventListener('click', (e) => {
                    if (!this.touches.allowClick) {
                    e.preventDefault();
                }
            });

                window.addEventListener('resize', this.resizeSlides);

                document.body.addEventListener('touchmove', this.stopDrag);
            },
            unbindEvents() {
                const _events = this.touchEvents();

                this.$el.removeEventListener(_events.start, this.touchStartHandler);
                this.$el.removeEventListener(_events.move, this.touchMoveHandler);
                this.$el.removeEventListener(_events.end, this.touchEndHandler);

                window.removeEventListener('resize', this.resizeSlides);

                document.body.removeEventListener('touchmove', this.stopDrag);
            },
            touchEvents() {
                const supportTouch = this.supportTouch = (window.Modernizr && !!window.Modernizr.touch) || (function () {
                            return !!(('ontouchstart' in window) || window.DocumentTouch && document instanceof DocumentTouch);
                        })();

                return {
                    start: supportTouch ? 'touchstart' : 'mousedown',
                    move: supportTouch ? 'touchmove' : 'mousemove',
                    end: supportTouch ? 'touchend' : 'mouseup'
                };
            },
            setTranslate(speed, translate) {
                this.dragStyleObject.transitionDuration = speed + 'ms';
                if (this.isVertical) {
                    this.dragStyleObject.transform = 'translate3d(0, ' + translate + 'px, 0)';
                } else {
                    this.dragStyleObject.transform = 'translate3d(' + translate + 'px, 0, 0)';
                }
            },
            resizeSlides() {
                if (this.isVertical) {
                    const height = this.$el.clientHeight;
                    this.dragStyleObject.transform = 'translate3d(0, ' + -this.index * height + 'px, 0)';
                } else {
                    const width = this.$refs.warpper.offsetWidth;
                    this.dragStyleObject.transform = 'translate3d(' + -this.index * width + 'px, 0, 0)';
                }
            },
            destroy() {
                this.unbindEvents();
                this.stopAutoplay();
            }
        },
        destroyed() {
            this.destroy();
        }
    }
</script>

<style lang="less">
    @import "../../../styles/components/slider.less";
</style>
