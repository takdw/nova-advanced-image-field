<template>
    <default-field
        :field="field"
        :errors="errors"
        :full-width-content="true"
        :show-help-text="!isReadonly && showHelpText"
    >
        <template slot="field">
            <image-viewer
                @image-deleted="imageDeleted"
                v-show="!imgSrc"
                :field="field"
                :resourceId="resourceId"
                :resourceName="resourceName"
                :relatedResourceId="relatedResourceId"
                :relatedResourceName="relatedResourceName"
                :viaRelationship="viaRelationship"
            ></image-viewer>
            
            <template v-if="field.croppable" v-show="imgSrc">
                <div v-if="isRange" class="max-w-xl mb-4">
                    <img :src="imgSrc" :alt="filename" ref="cropper">
                </div>

                <vue-cropper
                    v-else
                    class="mb-4"
                    ref='cropper'
                    :view-mode="1"
                    :aspect-ratio="field.aspectRatio || NaN"
                    :src="imgSrc"
                ></vue-cropper>
            </template>

            <p
                v-if="imgSrc"
                class="mt-3 mb-6 flex items-center text-sm"
            >
                <Button
                    type="restore"
                    @click="cancel"
                >
                    <span class="class ml-2 mt-1">
                        {{__('Cancel')}}
                    </span>
                </Button>
            </p>

            <span class="form-file mr-4">
                <input
                    ref="fileField"
                    :dusk="field.attribute"
                    class="form-file-input"
                    type="file"
                    :id="idAttr"
                    name="name"
                    :accept="field.acceptedTypes"
                    @change="fileChange"
                />
                <label :for="labelFor" class="form-file-btn btn btn-default btn-primary">
                    {{imgSrc ? __('Change File') : __('Choose File')}}
                </label>
            </span>
            <span class="text-gray-50">
                {{ currentLabel }}
            </span>

            <p v-if="hasError" class="text-xs mt-2 text-danger">
                {{ firstError }}
            </p>
        </template>
    </default-field>
</template>

<script>
import 'cropperjs/dist/cropper.css'
import VueCropper from 'vue-cropperjs'
import Cropper from 'cropperjs'
import { FormField, HandlesValidationErrors, Errors } from 'laravel-nova'

import Button from '@/components/Button/Button'
import ImageViewer from '@/components/Image/ImageViewer'

export default {
    props: ['field', 'resourceId', 'resourceName', 'relatedResourceId', 'relatedResourceName', 'viaRelationship'],

    mixins: [HandlesValidationErrors, FormField],

    components: { VueCropper, Button, ImageViewer },

    data: () => ({
        imgSrc: '',
        file: null,
        fileName: '',
        uploadErrors: new Errors(),
        cropper: null,
    }),

    mounted() {
        if (this.field.croppable && this.isRange)
            this.cropper = this.cropperWithMinMaxAspectRatio()
    },

    methods: {
        /**
         * Build a Cropper instance with a minimum and maxium aspect ratio
         */
        cropperWithMinMaxAspectRatio() {
            let minAspectRatio = parseFloat(this.field.aspectRatio)
            let maxAspectRatio = parseFloat(this.field.maxAspectRatio)

            return new Cropper(this.$refs.cropper, {
                cropmove(event) {
                    if (isNaN(minAspectRatio) || isNaN(maxAspectRatio)) {
                        return
                    }

                    let cropBoxData = this.cropper.getCropBoxData()
                    let cropBoxWidth = cropBoxData.width
                    let aspectRatio = cropBoxWidth / cropBoxData.height

                    if (aspectRatio < minAspectRatio) {
                        this.cropper.setCropBoxData({
                            height: cropBoxWidth / minAspectRatio
                        })
                    } else if (aspectRatio > maxAspectRatio) {
                        this.cropper.setCropBoxData({
                            height: cropBoxWidth / maxAspectRatio
                        })
                    }
                }
            })
        },

        /**
         * Fill the attributes on form submit
         */
        fill(formData) {
            if (this.file) {
                formData.append(this.field.attribute, this.file, this.fileName)
                if (this.field.croppable) {
                    formData.append(this.field.attribute + '_data', JSON.stringify(this.cropperObj.getData(true)))
                }
            }
        },

        /**
         * Cancel the new selected image
         */
        cancel() {
            if (this.field.croppable) {
                this.cropperObj.destroy()
            }
            this.imgSrc = ''
            this.file = null
            this.fileName = ''
        },

        /**
         * Respond to the file change
         * Set the data and init the crop box if the image is croppable
         */
        fileChange(e) {
            this.uploadErrors.clear(this.fieldAttribute)
            let path = e.target.value
            let fileName = path.match(/[^\\/]*$/)[0]
            this.fileName = fileName
            this.file = this.$refs.fileField.files[0]

            const file = e.target.files[0]
            if (!file.type.includes('image/')) {
                this.uploadErrors.record({
                    [this.fieldAttribute]: [this.__('Please select an image file')],
                })
                this.cancel()
                return
            }

            const split = fileName.split('.')
            const supportedTypesSplit = this.field.acceptedTypes.split(',')
            const supportedTypesSplitLength = supportedTypesSplit.length

            if (supportedTypesSplitLength > 1) {
                let supportedStr = ''
                supportedTypesSplit.forEach((type, index) => {
                    supportedStr += type.substr(1).toUpperCase()
                    if (supportedTypesSplitLength - 1 === index) {
                        return
                    } else if (supportedTypesSplitLength - 2 === index) {
                        supportedStr += ' or '
                    } else {
                        supportedStr += ', '
                    }
                })

                if (! supportedTypesSplit.includes('.' + split.pop().toLowerCase())) {
                    this.cancel()
                    this.uploadErrors.record({
                        [this.fieldAttribute]: [this.__(`Please select a ${supportedStr} image type.`)],
                    })
                    return
                }
            }

            if (this.field.croppable) {
                if (typeof FileReader === 'function') {
                    const reader = new FileReader()
                    reader.onload = (event) => {
                        this.imgSrc = event.target.result
                        this.cropperObj.replace(event.target.result)
                    }
                    reader.readAsDataURL(file)
                } else {
                    alert(this.__('Sorry, FileReader API not supported'))
                }
            }
        },

        /**
         * Inform the parent component that the file has been deleted
         * This event allows to update the `lastRetrievedAt` timestamp for further model changes
         */
        imageDeleted() {
            this.$emit('file-deleted')
        },
    },

    computed: {
        /**
         * Determine whether the image field has errors
         */
        hasError() {
            return this.uploadErrors.has(this.fieldAttribute)
        },

        /**
         * The first error, if any, of the image field
         */
        firstError() {
            if (this.hasError) {
                return this.uploadErrors.first(this.fieldAttribute)
            }
        },

        /**
         * The current label of the image field
         */
        currentLabel() {
            return this.fileName || this.__('no file selected')
        },

        /**
         * The ID attribute to use for the image field
         */
        idAttr() {
            return this.labelFor
        },

        /**
         * The label attribute to use for the image field
         */
        labelFor() {
            return `advanced-image-${this.field.attribute}`
        },

        /**
         * Is the aspect ratio provided a range?
         */
        isRange() {
            return this.maxAspectRatio !== null
        },

        /*
         * Get the cropper object
         */
        cropperObj() {
            return this.isRange ? this.cropper : this.$refs.cropper
        }
    },
}
</script>
