<template>
    <div
        :class="classes"
        :style="style"
        ref="dragHandle"
    >
        <slot></slot>
        <div
            class="resize-handle"
            ref="resizeHandle"
        ></div>
    </div>
</template>

<style>
    .dnd-grid-box {
        position: absolute;
        z-index: 1;
        box-sizing: border-box;
    }

    .dnd-grid-box.dragging,
    .dnd-grid-box.resizing {
        z-index: 2;
        opacity: 0.7;
    }

    .dnd-grid-box:not(.dragging):not(.resizing) {
        transition: transform ease-out 0.1s, width ease-out 0.1s, height ease-out 0.1s;
    }

    .dnd-grid-box .resize-handle {
        position: absolute;
        right: -5px;
        bottom: -5px;
        width: 15px;
        height: 15px;
        cursor: se-resize;
    }
</style>

<script>
    import { List as ContainerList } from './Container'
    import * as utils from './utils'

    const dragEvent = evt => {
        // Handle clicks on other area's than the drag bar
        if (!utils.matchesSelector(evt.target, this.dragSelector)) {
            return
        }

        evt.preventDefault()
        this.dragging = true
        this.$emit('dragStart')
        let xCo = evt.clientX || evt.touches[0].pageX
        let yCo = evt.clientY || evt.touches[0].pageY

        const handleEnd = evt => {
            window.removeEventListener('mouseup', handleEnd, true)
            window.removeEventListener('touchend', handleEnd, true)
            window.removeEventListener('mousemove', handleMove, true)
            window.removeEventListener('touchmove', handleMove, true)

            this.dragging = false

            const offset = {
                x: (evt.clientX || evt.changedTouches[0].pageX) - xCo,
                y: (evt.clientY || evt.changedTouches[0].pageY) - yCo
            }
            this.$emit('dragEnd', {offset})
        }

        const handleMove = evt => {
            const offset = {
                x: (evt.clientX || evt.touches[0].pageX) - xCo,
                y: (evt.clientY || evt.touches[0].pageY) - yCo
            }
            this.$emit('dragUpdate', {offset})
        }

        window.addEventListener('mouseup', handleEnd, true)
        window.addEventListener('touchend', handleEnd, true)
        window.addEventListener('mousemove', handleMove, true)
        window.addEventListener('touchmove', handleMove, true)
    }

    const resizeEvent = evt => {
        evt.preventDefault()
        evt.stopPropagation()
        this.resizing = true
        this.$emit('resizeStart')
        let mouseX = evt.clientX
        let mouseY = evt.clientY

        const handleMouseUp = evt => {
            window.removeEventListener('mouseup', handleMouseUp, true)
            window.removeEventListener('touchend', handleMouseUp, true)
            window.removeEventListener('mousemove', handleMouseMove, true)
            window.removeEventListener('touchmove', handleMouseMove, true)

            this.resizing = false

            var offset = {
                x: evt.clientX - mouseX,
                y: evt.clientY - mouseY
            }
            this.$emit('resizeEnd', {offset})
        }

        const handleMouseMove = evt => {
            var offset = {
                x: evt.clientX - mouseX,
                y: evt.clientY - mouseY
            }
            this.$emit('resizeUpdate', {offset})
        }

        window.addEventListener('mouseup', handleMouseUp, true)
        window.addEventListener('touchend', handleMouseUp, true)
        window.addEventListener('mousemove', handleMouseMove, true)
        window.addEventListener('touchmove', handleMouseMove, true)
    }

    export default {
        name: 'DndGridBox',
        props: {
            boxId: {
                required: true
            },
            dragSelector: {
                type: String,
                default: '*'
            }
        },
        data() {
            return {
                container: null,
                dragging: false,
                resizing: false
            }
        },
        computed: {
            style() {
                if (this.container && this.container.isBoxVisible(this.boxId)) {
                    var pixelPosition = this.container.getPixelPositionById(this.boxId)
                    return {
                        display: 'block',
                        width: pixelPosition.w + 'px',
                        height: pixelPosition.h + 'px',
                        transform: `translate(${pixelPosition.x}px, ${pixelPosition.y}px)`
                    }
                }

                return {
                    display: 'none'
                }
            },
            classes() {
                return {
                    'dnd-grid-box': true,
                    'dragging': this.dragging,
                    'resizing': this.resizing
                }
            }
        },
        methods: {
            findContainer() {
                let current = this
                while (current.$parent) {
                    current = current.$parent
                    if (ContainerList.has(current)) {
                        return current
                    }
                }
                return null
            }
        },
        mounted() {
            this.container = this.findContainer()
            if (!this.container) {
                throw new Error('Can not find container')
            }

            // register component on parent
            this.container.registerBox(this)

            // moving: Add listener to drag handle for moving the box around
            this.$dragHandle = this.$el || this.$refs.dragHandle
            this.$dragHandle.addEventListener('mousedown', dragEvent)
            this.$dragHandle.addEventListener('touchstart', dragEvent)

            // resizing: Add listener to resize handle for resizing the box
            this.$resizeHandle = this.$refs.resizeHandle
            if (this.$resizeHandle) {
                this.$resizeHandle.addEventListener('mousedown', resizeEvent)
                this.$resizeHandle.addEventListener('touchstart', resizeEvent)
            }
        },
        beforeDestroy() {
            // register component on parent
            if (this.container) {
                this.container.unregisterBox(this)
            }
        }
    }
</script>
