<template>
  <div id="lv-viewer" ref="container">

    <navbar
      v-if="navigation.length > 0"
      :navigation="navigation">
    </navbar>

    <main class="lv-container">
      <div id="lv-viewer-wrapper">
        <div id="hud" ref="hud">
          <toolbar-controls
            v-if="currentTask"
            :task="currentTask"
            :viewer="viewer"
            :buttons="mergedButtons"
            :helpButton="viewerOpts.helpButton"
            :infoButton="viewerOpts.infoButton"
            @helpclicked="showHelpModal = true"
            @infoclicked="showInfoModal = true"
            @browseclicked="showBrowseModal = true"
            @shareclicked="showShareModal = true"
            @likeclicked="emitTaskLiked"
            @fullscreenclicked="toggleFullScreen">
          </toolbar-controls>

          <pan-controls
            :viewer="viewer"
            :panBy="panBy">
          </pan-controls>

          <zoom-controls
            :viewer="viewer"
            :zoomBy="zoomBy">
          </zoom-controls>

          <info-modal
            v-if="($slots.info || currentTask) && mergedButtons.info"
            :task="currentTask"
            :show="showInfoModal"
            @hide="showInfoModal = false">
            <slot name="info"></slot>
          </info-modal>

          <help-modal
            v-if="($slots.help || currentTask) && mergedButtons.help"
            :task="currentTask"
            :buttons="mergedButtons"
            :show="showHelpModal"
            :browsable="browsable"
            :selections-editable="selectionsEditable"
            :disableComplete="disableComplete"
            @hide="showHelpModal = false">
            <slot name="help"></slot>
          </help-modal>

          <share-modal
            v-if="$slots.share && mergedButtons.share"
            :show="showShareModal"
            @hide="showShareModal = false">
            <slot name="share"></slot>
          </share-modal>

          <browse-modal
            v-if="mergedButtons.browse && browsable"
            :tasks="tasks"
            :show="showBrowseModal"
            :disableComplete="disableComplete"
            @hide="showBrowseModal = false"
            @taskclick="setCurrentTask">
          </browse-modal>

          <button
            v-if="browsable"
            :disabled="previousBtnDisabled"
            class="lv-btn lv-btn-hud"
            id="lv-browse-previous"
            @click="previousTask">
            <icon name="chevron-left" scale="1.5"></icon>
          </button>
          <button
            v-if="browsable"
            :disabled="nextBtnDisabled"
            class="lv-btn lv-btn-hud"
            id="lv-browse-next"
            @click="nextTask">
            <icon name="chevron-right" scale="1.5"></icon>
          </button>
        </div>

        <div :id="viewerOpts.id"></div>

        <transition name="fade">
          <div id="lv-viewer-disabled-overlay" v-if="viewerDisabled"></div>
        </transition>

        <selector
          v-if="selectorEnabled"
          :viewer="viewer"
          :task="currentTask"
          :selectionRect="selectionRect"
          @selection="handleSelection">
        </selector>

      </div>

      <base-sidebar
        v-if="currentTask"
        :task="currentTask"
        :showNote="showNote"
        :commentAnnotation="commentAnnotation"
        :disableComplete="disableComplete"
        :confirmOnSubmit="confirmOnSubmit"
        :buttons="mergedButtons"
        :display-xs="currentTask.mode === 'transcribe'"
        @noteupdated="updateNote"
        @submit="submitTask"
        @disableviewer="viewerDisabled = true"
        @enableviewer="viewerDisabled = false">

        <select-sidebar-item
          v-if="taskLoaded && currentTask.mode === 'select'"
          :task="currentTask"
          :tags="tags"
          :disableComplete="disableComplete"
          :selectionsEditable="selectionsEditable"
          :viewer="viewer"
          @edit="editTag"
          @delete="deleteTag">
        </select-sidebar-item>

        <transcribe-sidebar-item
          v-if="currentTask.mode === 'transcribe'"
          :task="currentTask"
          :disableComplete="disableComplete"
          @update="updateForm"
          @submit="submitTask"
          @inputfocus="onTranscribeInputFocus"
          @inputblur="onTranscribeInputBlur">
        </transcribe-sidebar-item>
      </base-sidebar>
    </main>

  </div>
</template>

<script>
import Icon from 'vue-awesome/components/Icon'
import 'vue-awesome/icons/chevron-left'
import 'vue-awesome/icons/chevron-right'
import OpenSeadragon from 'openseadragon'
import InfoModal from '@/components/modals/Info'
import HelpModal from '@/components/modals/Help'
import ShareModal from '@/components/modals/Share'
import BrowseModal from '@/components/modals/Browse'
import ToolbarControls from '@/components/controls/Toolbar'
import PanControls from '@/components/controls/Pan'
import ZoomControls from '@/components/controls/Zoom'
import BaseSidebar from '@/components/sidebars/Base'
import Navbar from '@/components/navigation/Navbar'
import SelectSidebarItem from '@/components/sidebars/items/Select'
import TranscribeSidebarItem from '@/components/sidebars/items/Transcribe'
import Selector from '@/components/Selector'
import Task from '@/model/Task'
import Annotator from '@/model/Annotator'
import getRectFromFragment from '@/utils/getRectFromFragment'
import toggleFullScreen from '@/utils/toggleFullScreen'
import drawOverlay from '@/utils/drawOverlay'
import deleteOverlay from '@/utils/deleteOverlay'

export default {
  data () {
    return {
      viewer: {},
      selectionRect: {},
      annotator: new Annotator({
        creator: this.creator,
        generator: this.generator
      }),
      viewerOpts: {
        id: 'lv-viewer-container',
        crossOriginPolicy: 'Anonymous',
        showNavigationControl: false,
        helpButton: 'show-help',
        infoButton: 'show-info',
        panVertical: false,
        panHorizontal: false,
        gestureSettingsMouse: {
          clickToZoom: false
        },
        gestureSettingsTouch: {
          dblClickToZoom: false
        },
        gestureSettingsPen: {
          dblClickToZoom: false
        }
      },
      defaultButtons: {
        fullscreen: 'Fullscreen',
        help: 'Help',
        info: 'Details',
        browse: 'Browse Tasks',
        like: ['Like', 'Unlike'],
        share: 'Share',
        download: 'Download',
        note: 'Add a note',
        submit: 'Save'
      },
      showInfoModal: false,
      showHelpModal: this.showHelpOnMount,
      showShareModal: false,
      showBrowseModal: false,
      viewerDisabled: false,
      tasks: [],
      currentTask: null,
      taskLoaded: false
    }
  },

  props: {
    taskOpts: {
      type: Array,
      required: true
    },
    confirmBeforeUnload: {
      type: Boolean,
      default: true
    },
    disableComplete: {
      type: Boolean,
      default: true
    },
    showNote: {
      type: Boolean,
      default: true
    },
    showFormErrors: {
      type: Boolean,
      default: true
    },
    panBy: {
      type: Number,
      default: 0.1
    },
    zoomBy: {
      type: Number,
      default: 0.2
    },
    creator: {
      type: Object,
      default: null
    },
    generator: {
      type: Object,
      default: null
    },
    showRelatedTasks: {
      type: Boolean,
      default: false
    },
    nextOnSubmit: {
      type: Boolean,
      default: true
    },
    navigation: {
      type: Array,
      default: () => []
    },
    confirmOnSubmit: {
      type: Boolean,
      default: false
    },
    buttons: {
      type: Object,
      default: () => ({}) // Defaults set in defaultToolbarButtons
    },
    selectionsEditable: {
      type: Boolean,
      default: true
    },
    browsable: {
      type: Boolean,
      default: true
    },
    showHelpOnMount: {
      type: Boolean,
      default: false
    },
    beforeSubmit: {
      type: Function,
      default: () => new Promise((resolve, reject) => resolve())
    }
  },

  components: {
    InfoModal,
    HelpModal,
    ShareModal,
    BrowseModal,
    ToolbarControls,
    PanControls,
    ZoomControls,
    BaseSidebar,
    Navbar,
    Selector,
    SelectSidebarItem,
    TranscribeSidebarItem,
    Icon
  },

  computed: {
    previousBtnDisabled () {
      if (!this.currentTask) {
        return true
      }
      return !(this.tasks.indexOf(this.currentTask) > 0)
    },

    nextBtnDisabled () {
      if (!this.currentTask) {
        return true
      }
      return this.tasks.indexOf(this.currentTask) >= this.tasks.length - 1
    },

    tags () {
      return this.annotator.getSelectAnnotations(this.currentTask)
    },

    commentAnnotation () {
      return this.annotator.getCommentAnnotation(this.currentTask)
    },

    selectorEnabled () {
      return (
        this.currentTask &&
        this.currentTask.mode === 'select' &&
        (!this.currentTask.complete || !this.disableComplete)
      )
    },

    mergedButtons () {
      let merged = JSON.parse(JSON.stringify(this.defaultButtons))
      for (let key in merged) {
        if (this.buttons.hasOwnProperty(key)) {
          merged[key] = this.buttons[key]
        }
      }
      if (!this.browsable) {
        merged.browse = false
      }
      if (!this.$slots.share) {
        merged.share = false
      }
      return merged
    }
  },

  methods: {
    /**
     * Emit the taskliked event.
     */
    emitTaskLiked (task, status) {
      task.liked = status
      this.$emit('taskliked', task)
    },

    /**
     * Toggle fullscreen mode.
     */
    toggleFullScreen () {
      const el = this.$refs.container
      toggleFullScreen(el)
    },

    /**
     * Draw overlay and add tag when a selection is made.
     * @param {Task} task
     *   The task.
     * @param {Object} rect
     *   The viewport rectangle.
     */
    handleSelection (task, rect) {
      const vp = this.viewer.viewport
      const imgRect = vp.viewportToImageRectangle(rect)
      const anno = this.annotator.createSelectAnnotation(task, imgRect)
      this.drawSelectionOverlay(task, anno)
      this.$emit('create', task, anno)
    },

    /**
     * Draw a selection overlays from an annotation.
     * @param {Task} task
     *   The Task.
     * @param {Annotation} annotation
     *   The Annotation.
     */
    drawSelectionOverlay (task, anno) {
      const vp = this.viewer.viewport
      const imgRect = getRectFromFragment(anno.target.selector.value)
      const vpRect = vp.imageToViewportRectangle(imgRect)
      const overlay = drawOverlay(this.viewer, anno.id, vpRect, 'selection')
      overlay.addEventListener('click', (evt) => {
        if (!task.complete && !this.disableComplete) {
          this.editTag(task, anno.id)
        }
      })
    },

    /**
     * Draw a highlight.
     * @param {Object} rect
     *   The image rectangle.
     * @param {String} id
     *   A ID for the highlight element.
     * @param {String} overlayCls
     *   Additional overlay class.
     * @param {Function} onClick
     *   A callback to trigger on click (or tap).
     */
    drawHighlight (rect, id, overlayCls = 'selection', onClick = null) {
      const vp = this.viewer.viewport
      const imgRect = new OpenSeadragon.Rect(
        rect.x,
        rect.y,
        rect.width,
        rect.height
      )
      const vpRect = vp.imageToViewportRectangle(imgRect)
      drawOverlay(this.viewer, id, vpRect, overlayCls, onClick)
    },

    /**
     * Draw all highlights for the task.
     * @param {Task} task
     *   The task.
     */
    drawHighlights (task) {
      for (let i = 0; i < task.highlights.length; i++) {
        this.drawHighlight(task.highlights[i], `highlight-${i}`)
      }
    },

    /**
     * Draw all highlights for a task, where clicking moves to that task.
     * @param {Task} task
     *   The task.
     */
    drawRelatedTaskHighlights (task) {
      // Merge general and specific fragment highlights
      let allHighlights = JSON.parse(JSON.stringify(task.highlights))
      if (task.form && 'fragments' in task.form) {
        for (let key in task.form.fragments) {
          allHighlights.push(task.form.fragments[key])
        }
      }

      for (let i = 0; i < allHighlights.length; i++) {
        // Check the highlight wasn't already drawn (e.g. on initial load)
        let taskIndex = this.tasks.indexOf(task)
        let highlightId = `related-t${taskIndex}-h${i}`
        let selector = `[data-id="${highlightId}"]`
        if (
          typeof document !== 'undefined' &&
          document.querySelector(selector)
        ) {
          continue
        }

        this.drawHighlight(
          allHighlights[i],
          highlightId,
          'related',
          () => { this.setCurrentTask(task) }
        )
      }
    },

    /**
     * Set the current task.
     * @param {Task} task.
     *   The task.
     */
    setCurrentTask (task) {
      this.$emit('taskchange', this.currentTask, task)
      this.currentTask = task
      if (this.showRelatedTasks) {
        this.showAllRelatedTasks(task)
      }
    },

    /**
     * Show all related tasks.
     * @param {Task} task.
     *   The task.
     */
    showAllRelatedTasks (task) {
      const relatedTasks = this.getRelatedTasks(task)
      for (let relatedTask of relatedTasks) {
        this.drawRelatedTaskHighlights(relatedTask)
      }
    },

    /**
     * Get all related tasks.
     * @param {Task} task.
     *   The task.
     */
    getRelatedTasks (task) {
      return this.tasks.filter((otherTask) => {
        const otherSourceStr = JSON.stringify(otherTask.tileSource)
        return (
          otherTask !== task &&
          typeof otherTask !== 'undefined' &&
          otherSourceStr === JSON.stringify(task.tileSource)
        )
      })
    },

    /**
     * Go to the previous task.
     */
    previousTask () {
      const index = this.tasks.indexOf(this.currentTask)
      if (index > 0) {
        this.setCurrentTask(this.tasks[index - 1])
      }
    },

    /**
     * Go to the next task.
     */
    nextTask () {
      const index = this.tasks.indexOf(this.currentTask)
      if (index + 1 < this.tasks.length) {
        this.setCurrentTask(this.tasks[index + 1])
      }
    },

    /**
     * Update the note and emit the relevant event.
     * @param {Task} task.
     *   The task.
     * @param {String} text.
     *   The text.
     */
    updateNote (task, text) {
      const now = new Date().toISOString()
      const anno = this.annotator.storeCommentAnnotation(
        task,
        text
      )
      if (anno.created > now) {
        this.$emit('create', task, anno)
      } else {
        this.$emit('update', task, anno)
      }
      return anno
    },

    /**
     * Update a form and associated annotations.
     * @param {Task} task.
     *   The task.
     * @param {Object} form
     *   The updated form.
     */
    updateForm (task, form) {
      task.updateForm(form)
      for (let key in form.model) {
        let anno = this.annotator.getTranscribeAnnotation(task, key)
        if (form.model[key]) {
          const now = new Date().toISOString()
          const anno = this.annotator.storeTranscriptionAnnotation(
            task,
            key,
            form.model[key],
            form.fragments[key]
          )
          if (anno.created > now) {
            this.$emit('create', task, anno)
          } else {
            this.$emit('update', task, anno)
          }
        } else if (anno) {
          this.annotator.deleteAnnotation(task, anno.id)
          this.$emit('delete', task, anno)
        }
      }
    },

    /**
     * Emit submit event with a task object.
     * @param {Task} task.
     *   The task.
     */
    submitTask (task) {
      // Show form errors if enabled and in transcribe mode
      if (
        this.showFormErrors &&
        this.currentTask.mode === 'transcribe' &&
        typeof document !== 'undefined'
      ) {
        const formGroups = document.querySelector('.form-group')
        if (formGroups.length) {
          formGroups.classList.add('show-errors')
        }
      }

      /* eslint-disable handle-callback-err */
      this.beforeSubmit(task).then(() => {
        task.complete = true
        if (this.nextOnSubmit) {
          this.nextTask()
        }
        this.$emit('submit', task)
      }).catch(err => {
        console.debug(err)
      })
    },

    /**
     * Remove a tag and enable the selector in the same location.
     * @param {Task} task
     *   The task that the tag belongs to.
     * @param {String} id
     *   The tag ID.
     */
    editTag (task, id) {
      const vp = this.viewer.viewport
      const anno = this.annotator.getAnnotation(task, id)
      const imgRect = getRectFromFragment(anno.target.selector.value)
      const vpRect = vp.imageToViewportRectangle(imgRect)
      const rect = new OpenSeadragon.Rect(
        vpRect.x,
        vpRect.y,
        vpRect.width,
        vpRect.height
      )
      this.selectionRect = rect
      this.deleteTag(task, id)
    },

    /**
     * Delete a tag.
     * @param {Task} task
     *   The task that the tag belongs to.
     * @param {String} id
     *   The tag ID.
     */
    deleteTag (task, id) {
      const anno = this.annotator.getAnnotation(task, id)
      this.annotator.deleteAnnotation(task, id)
      deleteOverlay(this.viewer, id)
      this.$emit('delete', task, anno)
    },

    /**
     * Highlight any fragments when transcribe form input focused.
     * @param {Task} task
     *   The task that the tag belongs to.
     * @param {String} modelKey
     *   The form model key.
     */
    onTranscribeInputFocus (task, modelKey) {
      if (modelKey in task.form.fragments) {
        const rect = task.form.fragments[modelKey]
        this.drawHighlight(rect, `highlight-${modelKey}`)
      }
    },

    /**
     * Remove all form fragment highlights when transcribe form input blured.
     * @param {Task} task
     *   The task that the tag belongs to.
     * @param {String} modelKey
     *   The form model key.
     */
    onTranscribeInputBlur (task, modelKey) {
      for (modelKey in task.form.fragments) {
        deleteOverlay(this.viewer, `highlight-${modelKey}`)
      }
    },

    /**
     * Mode specific configuration for select tasks.
     * @param {Task} task
     *   The task.
     */
    configureSelectMode (task) {
      if (task.mode === 'select') {
        // Draw all selection overlays
        const annos = this.annotator.getSelectAnnotations(task)
        for (let anno of annos) {
          this.drawSelectionOverlay(task, anno)
        }
      }
    },

    /**
     * Generate tasks from task options.
     */
    generateTasks () {
      this.tasks = this.taskOpts.map(opts => {
        return new Task(opts)
      })
    },

    /**
     * Load a task and configure the view accordingly.
     * @param {Task} task
     *   The task.
     */
    loadTask (task) {
      if (task.mode === 'select') {
        this.configureSelectMode(task)
      }
      this.drawHighlights(task)
      if (this.showRelatedTasks) {
        this.showAllRelatedTasks(task)
      }
    },

    /**
     * Check for any unsaved annotations.
     * @param {Object} evt
     *   The event.
     */
    onBeforeUnload (evt) {
      const msg = 'Unsaved changes will be lost.'
      let unsavedTasks = this.tasks.filter((task) => {
        return task.annotations.length > 0 && !task.complete
      })
      if (this.confirmBeforeUnload && unsavedTasks.length > 0) {
        evt.returnValue = msg
        return msg
      }
    },

    /**
     * Fit to bounds for the task.
     * @param {Task} task
     *   The task.
     */
    fitToBounds (task) {
      if (task.bounds) {
        const imgRect = new OpenSeadragon.Rect(
          task.bounds.x,
          task.bounds.y,
          task.bounds.width,
          task.bounds.height
        )
        const vpRect = this.viewer.viewport.imageToViewportRectangle(imgRect)
        this.viewer.viewport.fitBounds(vpRect)
      }
    }
  },

  watch: {
    currentTask: function (newTask, oldTask) {
      this.taskLoaded = false
      if (
        oldTask &&
        JSON.stringify(oldTask.tileSource) ===
        JSON.stringify(newTask.tileSource)
      ) {
        this.viewer.clearOverlays()
        this.loadTask(newTask)
        this.fitToBounds(newTask)
        this.taskLoaded = true
      } else {
        this.viewer.close()
        this.viewer.open({
          tileSource: newTask.tileSource,
          success: (evt) => {
            this.loadTask(newTask)
            // Set the task loaded flag after the tile has been fully loaded.
            // This helps with things such as the selection preview canvases
            // being drawn using the correct image (rather than the one that has
            // just been removed).
            evt.item.addHandler('fully-loaded-change', () => {
              this.taskLoaded = true
            })
          }
        })
        this.viewer.addHandler('open', () => {
          this.fitToBounds(newTask)
        })
      }
    },
    taskOpts: {
      handler () {
        this.generateTasks()
        if (this.tasks.length) {
          this.setCurrentTask(this.tasks[0])
        }
      },
      deep: true
    }
  },

  mounted () {
    this.viewer = new OpenSeadragon.Viewer(this.viewerOpts)
    this.generateTasks()
    if (this.tasks.length) {
      this.setCurrentTask(this.tasks[0])
    }

    if (typeof window !== 'undefined') {
      window.addEventListener('beforeunload', this.onBeforeUnload)
    }
  },

  beforeDestroy () {
    if (typeof window !== 'undefined') {
      window.removeEventListener('beforeunload', this.onBeforeUnload)
    }
  }
}
</script>
