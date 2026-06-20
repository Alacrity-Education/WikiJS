<template lang='pug'>
  v-dialog(
    v-model='isShown'
    max-width='550'
    overlay-color='blue darken-4'
    overlay-opacity='.7'
    )
    v-card
      .dialog-header.is-blue
        v-icon.mr-3(color='white') mdi-folder-plus-outline
        .body-1 {{$t('common:folder.create')}}
      v-card-text.pt-5
        v-text-field(
          ref='folderNameInput'
          v-model='folderName'
          :label='$t(`common:folder.name`)'
          outlined
          autofocus
          :hint='$t(`common:folder.nameHint`)'
          persistent-hint
          @input='generatePath'
          @keyup.enter='createFolder'
        )
        v-text-field.mt-4(
          v-model='folderSlug'
          :label='$t(`common:folder.slug`)'
          outlined
          :prefix='pathPrefix'
          :hint='$t(`common:folder.slugHint`)'
          persistent-hint
          :rules='[slugRule]'
        )
        v-select.mt-4(
          v-if='namespaces.length > 1'
          v-model='currentLocale'
          :items='namespaces'
          :label='$t(`common:field.locale`)'
          outlined
          hide-details
        )
      v-card-chin
        v-spacer
        v-btn(text, @click='close') {{$t('common:actions.cancel')}}
        v-btn.px-4(color='primary', @click='createFolder', :disabled='!isValid', :loading='loading')
          v-icon(left) mdi-check
          span {{$t('common:actions.create')}}
</template>

<script>
import _ from 'lodash'
import gql from 'graphql-tag'
import { get } from 'vuex-pathify'

/* global siteLangs, siteConfig */

export default {
  props: {
    value: {
      type: Boolean,
      default: false
    }
  },
  data () {
    return {
      folderName: '',
      folderSlug: '',
      currentLocale: siteConfig.lang,
      loading: false,
      namespaces: siteLangs.length ? siteLangs.map(ns => ns.code) : [siteConfig.lang]
    }
  },
  computed: {
    isShown: {
      get () { return this.value },
      set (val) { this.$emit('input', val) }
    },
    path: get('page/path'),
    locale: get('page/locale'),
    pathPrefix () {
      const parentPath = this.parentPath
      return parentPath ? `/${parentPath}/` : '/'
    },
    parentPath () {
      if (!this.path || this.path === 'home') return ''
      return this.path
    },
    folderPath () {
      const parent = this.parentPath
      return parent ? `${parent}/${this.folderSlug}` : this.folderSlug
    },
    fullPath () {
      return `${this.folderPath}/index`
    },
    isValid () {
      return this.folderName.trim().length > 0 &&
        this.folderSlug.length > 0 &&
        !this.folderSlug.includes(' ') &&
        !this.folderSlug.includes('.') &&
        !this.folderSlug.includes('/') &&
        !this.folderSlug.includes('\\')
    },
    slugRule () {
      return v => {
        if (!v || v.length < 1) return this.$t('common:folder.slugRequired')
        if (/[ .\\/]/.test(v)) return this.$t('common:folder.slugInvalid')
        return true
      }
    }
  },
  watch: {
    isShown (newValue) {
      if (newValue) {
        this.folderName = ''
        this.folderSlug = ''
        this.currentLocale = this.locale || siteConfig.lang
        this.$nextTick(() => {
          if (this.$refs.folderNameInput) {
            this.$refs.folderNameInput.focus()
          }
        })
      }
    }
  },
  methods: {
    generatePath () {
      this.folderSlug = this.folderName
        .replace(/\s+(.)/g, (match, chr) => chr.toUpperCase())
        .replace(/\s+/g, '')
        .replace(/[^\w-]/g, '')
    },
    close () {
      this.isShown = false
    },
    async createFolder () {
      if (!this.isValid || this.loading) return
      this.loading = true
      this.$store.commit('loadingStart', 'folder-create')
      try {
        const resp = await this.$apollo.mutate({
          mutation: gql`
            mutation (
              $content: String!
              $description: String!
              $editor: String!
              $isPublished: Boolean!
              $isPrivate: Boolean!
              $locale: String!
              $path: String!
              $tags: [String]!
              $title: String!
            ) {
              pages {
                create(
                  content: $content
                  description: $description
                  editor: $editor
                  isPublished: $isPublished
                  isPrivate: $isPrivate
                  locale: $locale
                  path: $path
                  tags: $tags
                  title: $title
                ) {
                  responseResult {
                    succeeded
                    errorCode
                    message
                  }
                  page {
                    id
                    path
                  }
                }
              }
            }
          `,
          variables: {
            content: '<div class="wikijs-folder-marker"></div>',
            description: '',
            editor: 'code',
            isPublished: true,
            isPrivate: false,
            locale: this.currentLocale,
            path: this.fullPath,
            tags: [],
            title: this.folderName.trim()
          }
        })
        const result = _.get(resp, 'data.pages.create.responseResult', {})
        if (result.succeeded) {
          this.close()
          window.location.assign(`/${this.currentLocale}/${this.fullPath}`)
        } else {
          throw new Error(result.message || this.$t('common:error.unexpected'))
        }
      } catch (err) {
        this.$store.commit('pushGraphError', err)
      }
      this.$store.commit('loadingStop', 'folder-create')
      this.loading = false
    }
  }
}
</script>
