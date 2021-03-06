<template>
    <figure
        class="b-image-wrapper"
        :class="figureClasses"
        :style="figureStyles"
    >
        <transition name="fade">
            <img
                v-if="isDisplayed"
                :srcset="computedSrcset"
                :src="computedSrc"
                :class="imgClasses"
                @load="onLoad"
                :width="computedWidth"
                :sizes="computedSizes"
            >
        </transition>
        <transition name="fade">
            <slot
                v-if="isPlaceholderDisplayed"
                name="placeholder"
            >
                <img
                    :src="computedPlaceholder"
                    :class="imgClasses"
                    class="placeholder"
                >
            </slot>
        </transition>
    </figure>
</template>

<script>
import config from '../../utils/config'
import { isWebpSupported } from '../../utils/helpers'

export default {
    name: 'BImage',
    props: {
        src: String,
        webpFallback: {
            type: String,
            default: () => {
                return config.defaultImageWebpFallback
            }
        },
        lazy: {
            type: Boolean,
            default: () => {
                return config.defaultImageLazy
            }
        },
        responsive: {
            type: Boolean,
            default: () => {
                return config.defaultImageResponsive
            }
        },
        ratio: {
            type: String,
            default: () => {
                return config.defaultImageRatio
            }
        },
        placeholder: String,
        srcset: String,
        srcsetSizes: Array,
        srcsetFormatter: {
            type: Function,
            default: (src, size, vm) => {
                if (typeof config.defaultImageSrcsetFormatter === 'function') {
                    return config.defaultImageSrcsetFormatter(src, size)
                } else {
                    return vm.formatSrcset(src, size)
                }
            }
        },
        rounded: {
            type: Boolean,
            default: false
        }
    },
    data() {
        return {
            clientWidth: 0,
            webpSupportVerified: false,
            webpSupported: false,
            observer: null,
            inViewPort: false,
            bulmaKnownRatio: ['square', '1by1', '5by4', '4by3', '3by2', '5by3', '16by9', 'b2y1', '3by1', '4by5', '3by4', '2by3', '3by5', '9by16', '1by2', '1by3'],
            loaded: false
        }
    },
    computed: {
        ratioPattern() {
            return new RegExp(/([0-9]+)by([0-9]+)/)
        },
        hasRatio() {
            return this.ratio && this.ratioPattern.test(this.ratio)
        },
        figureClasses() {
            const classes = { image: this.responsive }
            if (this.hasRatio && this.bulmaKnownRatio.indexOf(this.ratio) >= 0) {
                classes[`is-${this.ratio}`] = true
            }
            return classes
        },
        figureStyles() {
            if (
                this.hasRatio &&
                this.bulmaKnownRatio.indexOf(this.ratio) < 0
            ) {
                const ratioValues = this.ratioPattern.exec(this.ratio)
                return {
                    paddingTop: `${(ratioValues[2] / ratioValues[1]) * 100}%`
                }
            }
        },
        imgClasses() {
            return {
                'is-rounded': this.rounded,
                'has-ratio': this.hasRatio
            }
        },
        srcExt() {
            return this.getExt(this.src)
        },
        isWepb() {
            return this.srcExt === 'webp'
        },
        computedSrc() {
            if (!this.webpSupported && this.isWepb && this.webpFallback) {
                if (this.webpFallback.startsWith('.')) {
                    return this.src.replace(/\.webp/gi, `${this.webpFallback}`)
                }
                return this.webpFallback
            }
            return this.src
        },
        computedWidth() {
            if (this.responsive && this.clientWidth > 0) {
                return this.clientWidth
            }
        },
        isDisplayed() {
            return (
                (this.webpSupportVerified || !this.isWepb) &&
                (this.inViewPort || !this.lazy)
            )
        },
        placeholderExt() {
            if (this.placeholder) {
                return this.getExt(this.placeholder)
            }
        },
        isPlaceholderWepb() {
            if (this.placeholder) {
                return this.placeholderExt === 'webp'
            }
        },
        computedPlaceholder() {
            if (!this.webpSupported && this.isPlaceholderWepb && this.webpFallback && this.webpFallback.startsWith('.')) {
                return this.placeholder.replace(/\.webp/gi, `${this.webpFallback}`)
            }
            return this.placeholder
        },
        isPlaceholderDisplayed() {
            return (
                !this.loaded &&
                (
                    this.$slots.placeholder || (
                        this.placeholder &&
                        (this.webpSupportVerified || !this.isPlaceholderWepb)
                    )
                )
            )
        },
        computedSrcset() {
            if (this.srcset) {
                if (!this.webpSupported && this.isWepb && this.webpFallback && this.webpFallback.startsWith('.')) {
                    return this.srcset.replace(/\.webp/gi, `${this.webpFallback}`)
                }
                return this.srcset
            }
            if (
                this.srcsetSizes && Array.isArray(this.srcsetSizes) && this.srcsetSizes.length > 0
            ) {
                return this.srcsetSizes.map((size) => {
                    return `${this.srcsetFormatter(this.computedSrc, size, this)} ${size}w`
                }).join(',')
            }
        },
        computedSizes() {
            if (this.computedSrcset && this.computedWidth) {
                return `${this.computedWidth}px`
            }
        }
    },
    methods: {
        getExt(filename, clean = true) {
            if (filename) {
                const noParam = clean ? filename.split('?')[0] : filename
                return noParam.split('.').pop()
            }
            return ''
        },
        setWidth() {
            this.clientWidth = this.$el.clientWidth
        },
        formatSrcset(src, size) {
            const ext = this.getExt(src, false)
            const name = src.split('.').slice(0, -1).join('.')
            return `${name}-${size}.${ext}`
        },
        onLoad(event) {
            this.loaded = true
            const { target } = event
            this.$emit('load', event, target.currentSrc || target.src || this.computedSrc)
        }
    },
    created() {
        if (this.isWepb) {
            isWebpSupported().then((supported) => {
                this.webpSupportVerified = true
                this.webpSupported = supported
            })
        }
        if (this.lazy && typeof window !== 'undefined' && 'IntersectionObserver' in window) {
            this.observer = new IntersectionObserver((events) => {
                const {target, isIntersecting} = events[0]
                if (isIntersecting && !this.inViewPort) {
                    this.inViewPort = true
                    this.observer.unobserve(target)
                }
            })
        }
    },
    mounted() {
        if (this.lazy && this.observer) {
            this.observer.observe(this.$el)
        }
        this.setWidth()
        if (typeof window !== 'undefined') {
            window.addEventListener('resize', this.setWidth)
        }
    },
    beforeDestroy() {
        if (this.observer) {
            this.observer.disconnect()
        }
        if (typeof window !== 'undefined') {
            window.removeEventListener('resize', this.setWidth)
        }
    }
}
</script>
