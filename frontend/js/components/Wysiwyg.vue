<template>
  <a17-inputframe :error="error" :note="note" :label="label" :locale="locale" @localize="updateLocale" :size="size"
                  :name="name" :required="required">
    <div class="wysiwyg__outer" :class="textfieldClasses">
      <input :name="name" type="hidden" v-model="value"/>
      <template v-if="editSource">
        <div class="wysiwyg" :class="textfieldClasses" v-show="!activeSource" :dir="dirLocale">
          <div class="wysiwyg__editor" :class="{ 'wysiwyg__editor--limitHeight' : limitHeight }" ref="editorcontainer">
            <div class="wysiwyg__editor-inner" ref="editor"></div>
          </div>
          <span v-if="shouldShowCounter" class="wysiwyg__limit f--tiny" :class="limitClasses">{{ counter }}</span>
        </div>
        <div class="form__field form__field--textarea" v-show="activeSource" :dir="dirLocale">
          <textarea :placeholder="placeholder" :autofocus="autofocus" v-model="value" @change="updateSourcecode"
                    :style="textareaHeight"></textarea>
        </div>
        <a17-button variant="ghost" @click="toggleSourcecode" class="wysiwyg__button">Source code</a17-button>
      </template>
      <template v-else>
        <div class="wysiwyg" :class="textfieldClasses" :dir="dirLocale">
          <div class="wysiwyg__editor" :class="{ 'wysiwyg__editor--limitHeight' : limitHeight }" ref="editorcontainer">
            <div class="wysiwyg__editor-inner" ref="editor"></div>
          </div>
          <span v-if="shouldShowCounter" class="wysiwyg__limit f--tiny" :class="limitClasses">{{ counter }}</span>
        </div>
      </template>
    </div>
  </a17-inputframe>
</template>

<script>
  import { mapState } from 'vuex'

  import 'quill/dist/quill.snow.css'
  import 'quill/dist/quill.bubble.css'
  import 'quill/dist/quill.core.css'

  import QuillConfiguration from '@/libs/Quill/QuillConfiguration'

  import debounce from 'lodash/debounce'

  import InputMixin from '@/mixins/input'
  import FormStoreMixin from '@/mixins/formStore'
  import InputframeMixin from '@/mixins/inputFrame'
  import LocaleMixin from '@/mixins/locale'

  import { loadScript } from '@/utils/loader'

  const HIGHLIGHT = '//cdn.jsdelivr.net/gh/highlightjs/cdn-release@9.12.0/build/highlight.min.js'

  export default {
    name: 'A17Wysiwyg',
    mixins: [InputMixin, InputframeMixin, LocaleMixin, FormStoreMixin],
    props: {
      editSource: {
        type: Boolean,
        default: false
      },
      showCounter: {
        type: Boolean,
        default: true
      },
      type: {
        type: String,
        default: 'text'
      },
      prefix: {
        type: String,
        default: ''
      },
      maxlength: {
        type: Number,
        default: 0
      },
      initialValue: {
        default: ''
      },
      limitHeight: {
        type: Boolean,
        default: false
      },
      options: {
        type: Object,
        required: false,
        default: function () {
          return {}
        }
      }
    },
    computed: {
      textareaHeight: function () {
        return {
          height: this.editorHeight
        }
      },
      textfieldClasses: function () {
        return {
          's--disabled': this.disabled,
          's--focus': this.focused
        }
      },
      hasMaxlength: function () {
        return this.maxlength > 0
      },
      shouldShowCounter: function () {
        return this.hasMaxlength && this.showCounter
      },
      limitClasses: function () {
        return {
          'wysiwyg__limit--red': this.counter < 10
        }
      },
      ...mapState({
        baseUrl: state => state.form.baseUrl
      })
    },
    data: function () {
      return {
        value: this.initialValue,
        editorHeight: 50,
        toolbarHeight: 52,
        focused: false,
        activeSource: false,
        quill: null,
        counter: 0,
        defaultModules: {
          toolbar: ['bold', 'italic', 'underline', 'link'],
          // Complete Toolbar example :
          // ['blockquote', 'code-block', 'strike'],
          // [{ 'header': 1 }, { 'header': 2 }],
          // [{ 'list': 'ordered'}, { 'list': 'bullet' }],
          // [{ 'script': 'sub'}, { 'script': 'super' }],
          // [{ 'indent': '-1'}, { 'indent': '+1' }],
          // [{ 'direction': 'rtl' }],
          // [{ 'size': ['small', false, 'large', 'huge'] }],
          // [{ 'header': [1, 2, 3, 4, 5, 6, false] }],
          // [{ 'color': [] }, { 'background': [] }],
          // [{ 'font': [] }],
          // [{ 'align': [] }],
          // ['clean'],
          // ['link', 'image', 'video']
          clipboard: {
            matchVisual: false,
            matchers: [
              QuillConfiguration.lineBreak.clipboard
            ]
          },
          keyboard: {
            bindings: {
              lineBreak: QuillConfiguration.lineBreak.handle
            }
          },
          syntax: false
        }
      }
    },
    methods: {
      initQuill () {
        // init Quill
        this.quill = new QuillConfiguration.Quill(this.$refs.editor, this.options)

        // set editor content
        if (this.value) this.updateEditor(this.value)

        // update model if text changes
        this.quill.on('text-change', (delta, oldDelta, source) => {
          let html = this.$refs.editor.children[0].innerHTML
          if (html === '<p><br></p>') html = ''
          this.value = html

          this.$emit('input', this.value)
          this.$emit('change', this.value)

          this.updateCounter(this.getTextLength())

          if (source === 'user') this.textUpdate()
        })

        // focus / blur event
        this.quill.on('selection-change', (range, oldRange, source) => {
          if (range) {
            this.focused = true
            this.$emit('focus')
          } else {
            this.focused = false

            // save value in store
            if (source === 'user') this.saveIntoStore()
            this.$emit('blur')
          }
        })

        // disabled
        if (this.disabled) {
          this.quill.enable(false)
        }

        // Change the link placeholder + add a checkbox to add or not the target blank attribute to current url
        if (this.baseUrl) {
          const tooltip = this.quill.theme.tooltip
          const rootElement = tooltip.root

          if (rootElement) {
            const input = rootElement.querySelector('input[data-link]')
            if (input) input.setAttribute('data-link', this.baseUrl)
          }
        }

        // set Quill direction
        if (this.dirLocale === 'rtl') {
          this.quill.format('direction', 'rtl')
          this.quill.format('align', 'right')
        }

        // check text length
        if (this.hasMaxlength && this.showCounter) {
          this.updateCounter(this.getTextLength())
        }

        // emit ready
        this.$emit('ready', this.quill)
      },
      anchorHandler (value) {
        if (value === true) {
          value = prompt('Enter anchor:')
        } else {
          const range = this.quill.getSelection()
          const id = this.quill.getFormat(range).anchor || ''
          value = prompt('Edit anchor:', id)
        }
        this.quill.format('anchor', value)
      },
      updateEditor: function (newValue) {
        // convert string to HTML and update the content silently
        const htmlData = this.quill.clipboard.convert(newValue)
        this.quill.setContents(htmlData, 'silent')
      },
      updateFromStore: function (newValue) { // called from the formStore mixin
        if (typeof newValue === 'undefined') newValue = ''

        if (this.value !== newValue) {
          this.value = newValue
          this.updateEditor(newValue)
        }
      },
      textUpdate: function () {
        this.preventSubmit()
        this._textUpdateInternal()
      },
      _textUpdateInternal: debounce(function () {
        this.saveIntoStore()
        this.allowSubmit()
      }, 600),
      toggleSourcecode: function () {
        this.editorHeight = (Math.max(50, this.$refs.editor.clientHeight) + this.toolbarHeight - 1) + 'px'
        this.activeSource = !this.activeSource

        this.updateSourcecode()
      },
      updateSourcecode: function () {
        // set editor content
        this.updateEditor(this.value)
        this.saveIntoStore() // see formStore mixin
      },
      updateCounter: function (newValue) {
        if (this.showCounter && this.hasMaxlength) {
          this.counter = this.maxlength - newValue
        }
      },
      getTextLength: function () {
        // see https://quilljs.com/docs/api/#getlength
        return this.quill.getLength() - (this.value.length === 0 ? 2 : 1)
      },
      preventEditorScroll: function () {
        // see https://github.com/quilljs/quill/issues/482
        this.$nextTick(() => {
          const tooltips = document.querySelectorAll('.ql-tooltip')
          tooltips.forEach((tooltip) => {
            const action = tooltip.querySelector('.ql-action')
            let scrollPosition
            action.addEventListener('mouseover', () => {
              scrollPosition = this.$refs.editorcontainer.scrollTop
            })
            tooltip.addEventListener('mouseup', (event) => {
              setTimeout(() => {
                this.$refs.editorcontainer.scrollTop = scrollPosition
              }, 0)
              event.preventDefault()
              event.stopPropagation()
            })
          })
        })
      }
    },
    mounted: function () {
      this.preventEditorScroll()

      if (this.quill) return

      /* global hljs */
      this.options.theme = this.options.theme || 'snow'
      this.options.boundary = this.options.boundary || document.body
      this.options.modules = this.options.modules || this.defaultModules
      const toolbar = {
        container: this.options.modules.toolbar !== undefined ? this.options.modules.toolbar : this.defaultModules.toolbar,
        handlers: {}
      }
      this.options.modules.clipboard = this.options.modules.clipboard !== undefined ? this.options.modules.clipboard : this.defaultModules.clipboard
      this.options.modules.keyboard = this.options.modules.keyboard !== undefined ? this.options.modules.keyboard : this.defaultModules.keyboard
      this.options.modules.syntax = this.options.modules.syntax !== undefined && this.options.modules.syntax ? { highlight: text => hljs.highlightAuto(text).value } : this.defaultModules.syntax
      this.options.placeholder = this.options.placeholder || this.placeholder
      this.options.readOnly = this.options.readOnly !== undefined ? this.options.readOnly : this.readonly
      this.options.formats = QuillConfiguration.getFormats(this.options.modules.toolbar) // Formats are based on current toolbar configuration
      this.options.bounds = this.$refs.editor

      // Ensure pasting content do not make editor scroll to the top
      // @see https://github.com/quilljs/quill/issues/1374#issuecomment-545112021
      this.options.scrollingContainer = 'html'

      // register custom handlers
      // register anchor toolbar handler
      if (toolbar.container.includes('anchor')) {
        toolbar.handlers.anchor = this.anchorHandler
      }

      this.options.modules.toolbar = toolbar

      if (this.options.modules.syntax && typeof hljs === 'undefined') {
        const id = 'highlight-js-script'
        loadScript(id, HIGHLIGHT, 'text/javascript').then(() => {
          this.initQuill()
        })
      } else {
        this.initQuill()
      }
    },
    beforeDestroy () {
      this.quill = null
    }
  }
</script>

<style lang="scss" scoped>
  .wysiwyg__button {
    margin-top: 20px;
  }

  .wysiwyg__editor--limitHeight {
    max-height: calc(100vh - 250px);
    overflow-y: scroll;
    min-height: 142px;
    border: 1px solid $color__fborder;
    border-top: none;
    scroll-behavior: smooth;
    margin-top: 32px;
    .input--error & {
      border-color: $color__error;
      border-top: none;
    }
    .s--focus & {
      border-color: $color__fborder--hover;
      border-top: none;
    }
  }
</style>
<style lang="scss">
  /* Not scoped style here because we want to overwrite default style of the wysiwig */

  $height_input: 45px;
  .wysiwyg__limit {
    height: $height_input - 2px;
    line-height: $height_input - 2px;
    color: $color__text--light;
    user-select: none;
    pointer-events: none;
    position: absolute;
    right: 15px;
    bottom: 0;
  }

  .wysiwyg__limit--red {
    color: red;
  }

  /* RTL Direction */
  .wysiwyg[dir='rtl'] .wysiwyg__limit {
    left:15px;
    right:auto;
  }

</style>
<style lang="scss">
  /* Not scoped style here because we want to overwrite default style of the wysiwig */
  .a17 {
    .ql-toolbar.ql-snow {
      border-top-left-radius: 2px;
      border-top-right-radius: 2px;
      background-color: $color__f--bg;
      font-family: inherit;
    }

    .wysiwyg__editor--limitHeight .ql-toolbar {
      z-index: 1;
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
    }

    .ql-editor.ql-blank::before {
      font-style: normal;
      color: $color__f--placeholder;
      @include font-regular;
    }

    .ql-container.ql-snow {
      border-bottom-left-radius: 2px;
      border-bottom-right-radius: 2px;
    }

    .ql-editor {
      background-color: $color__f--bg;
      min-height: 15px * 6;
      caret-color: $color__action;
      color: $color__text--forms;
      overflow: visible;

      &:hover,
      &:focus {
        background: $color__background;
      }
    }

    *[dir='rtl'] .ql-editor {
      direction: rtl;
      text-align: right;
    }

    .wysiwyg__editor--limitHeight .ql-editor {
      min-height: 15px * 10;
    }

    /* Default content styling */
    .ql-snow .ql-editor {
      h1, h2, h3, h4, h5, h6 {
        font-weight: 700;
      }

      b, p b, p strong, strong {
        font-weight:700;
      }

      i, p i, li i, em, p em, li em {
        font-style: italic;
      }

      u, p u, li u {
        text-decoration: underline;
      }

      p, ul, ol, h1, h2, h3, h4, h5 {
        margin-bottom: 1em;
      }

      h1 {
        font-size: 2em;
        line-height: 1.25em;
      }

      h2 {
        font-size: 1.66em;
        line-height: 1.25em;
      }

      h3 {
        font-size: 1.33em;
        line-height: 1.25em;
      }

      h4 {
        font-size: 1.25em;
        line-height: 1.25em;
      }

      h5 {
        font-size: 1em;
        line-height: 1.25em;
      }

      // default code syntax hightlighting is github
      pre {
        color: $color__wysiwyg-codeText;
        padding: 15px;
        overflow: auto;
        background-color: $color__wysiwyg-codeBg;
        border-radius: 3px;
        font-family: "SFMono-Regular", Consolas, "Liberation Mono", Menlo, Courier, monospace;
      }

      sup {
        vertical-align: super;
        font-size: smaller;
      }

      sub {
        vertical-align: sub;
        font-size: smaller;
      }
    }

    .ql-toolbar.ql-snow {
      border-color: $color__fborder;
      border-bottom-color: $color__border--light;
    }

    .ql-container.ql-snow {
      border-color: $color__fborder;
      .wysiwyg__editor--limitHeight {
        border: none;
      }
    }

    .wysiwyg__editor--limitHeight .ql-container.ql-snow {
      border: none;
    }

    .input--error {
      .ql-toolbar.ql-snow {
        border-color: $color__error;
        border-bottom-color: $color__border--light;
      }

      .ql-container.ql-snow {
        border-color: $color__error;
      }

      .wysiwyg__editor--limitHeight .ql-container.ql-snow {
        border: none;
      }
    }

    .s--focus {
      .ql-toolbar.ql-snow {
        border-color: $color__fborder--hover;
        border-bottom-color: $color__border--light;
      }

      .ql-container.ql-snow {
        border-color: $color__fborder--hover;
      }

      .wysiwyg__editor--limitHeight .ql-container.ql-snow {
        border: none;
      }
    }

    .ql-snow a {
      color: $color__link;
    }

    .ql-editor .ql-anchor {
      text-decoration: underline $color__link;
    }

    // Ensure pasting content do not make editor scroll to the top
    // @see https://github.com/quilljs/quill/issues/1374#issuecomment-545112021
    .ql-clipboard {
      position: fixed;
    }

    .ql-snow.ql-toolbar {
      padding: 13px 8px;

      button,
      .ql-align {
        width: 24px;
        margin-right: 35px - 6px - 6px - 6px - 6px;
        text-align: center;
      }

      button.ql-underline {
        top: 1px;
      }

      button.ql-link {
        width: 24px + 9px;
      }

      .icon {
        position: relative;
      }
    }

    .ql-snow.ql-toolbar .ql-formats {
      border-right: 1px solid $color__border--light;

      &:last-child {
        border-right: none;
      }
    }

    .ql-snow.ql-toolbar,
    .ql-snow .ql-toolbar {
      button:hover,
      button:focus,
      button.ql-active,
      .ql-picker-label:hover,
      .ql-picker-label.ql-active,
      .ql-picker-item:hover,
      .ql-picker-item.ql-selected {
        color: $color__link;
      }
    }

    .ql-snow.ql-toolbar,
    .ql-snow .ql-toolbar {
      button:hover .ql-stroke,
      button:focus .ql-stroke,
      button.ql-active .ql-stroke,
      .ql-picker-label:hover .ql-stroke,
      .ql-picker-label.ql-active .ql-stroke,
      .ql-picker-item:hover .ql-stroke,
      .ql-picker-item.ql-selected .ql-stroke,
      button:hover .ql-stroke-miter,
      button:focus .ql-stroke-miter,
      button.ql-active .ql-stroke-miter,
      .ql-picker-label:hover .ql-stroke-miter,
      .ql-picker-label.ql-active .ql-stroke-miter,
      .ql-picker-item:hover .ql-stroke-miter,
      .ql-picker-item.ql-selected .ql-stroke-miter {
        color: $color__link;
      }
    }

    /* dropdown style */
    .ql-toolbar.ql-snow .ql-header,
    .ql-toolbar.ql-snow .ql-size,
    .ql-toolbar.ql-snow .ql-font {
      .ql-picker-label {
        border: 0 none;
        position: relative;
        /*padding-right: 30px;*/

        &::after {
          content: " ";
          position: absolute;
          top: 50%;
          right: 1em;
          z-index: 2;
          position: absolute;
          width: 0;
          height: 0;
          margin-top: -3px;
          border-width: 4px 4px 0;
          border-style: solid;
          border-color: $color__text transparent transparent;
        }

        svg {
          opacity: 0;
        }
      }

      .ql-picker-options {
        background: rgba($color__background, 0.98);
        border-radius: 2px;
        box-shadow: $box-shadow;
        padding: 10px 0;
        border: 0 none;
        margin-top: 6px;

        .ql-picker-item {
          display: block;
          color: $color__text--light;
          padding: 0 15px;
          padding-right: 50px;
          height: 40px;
          line-height: 40px;
          text-decoration: none;
          white-space: nowrap;
          font-family: inherit;

          &:hover {
            color: $color__text;
            background: $color__light;
          }
        }
      }
    }

    .ql-toolbar.ql-snow .ql-picker {
      font-size: 1em;
    }

    .ql-toolbar.ql-snow .ql-picker .ql-picker-label {
      white-space: nowrap;

      &::before {
        line-height: 24px
      }
    }

    .ql-snow .ql-picker.ql-header {
      width: auto;
      min-width: 120px;

      .ql-picker-item,
      .ql-picker-item[data-value="1"],
      .ql-picker-item[data-value="2"],
      .ql-picker-item[data-value="3"],
      .ql-picker-item[data-value="4"],
      .ql-picker-item[data-value="5"] {
        &::before {
          font-weight: normal;
          font-size: 1em;
          white-space: nowrap;
        }
      }
    }
  }
</style>
