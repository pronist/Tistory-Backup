<template lang="pug">
  #extractor
    nav#nav(class='uk-navbar-container uk-navbar-transparent' uk-navbar)
      div(class='uk-navbar-left')
        ul(class='uk-navbar-nav')
          li: router-link(to='/' uk-icon='icon: arrow-left; ratio: 1.4')
      div(class='uk-navbar-right')
        ul(class='uk-navbar-nav')
          li: img(:src='userProfile')
    main#main
      h3.warning *글만 백업되며, 페이지와 공지사항은 미포함입니다. (티스토리 오픈 API 미지원)
      table(class='uk-table uk-table-middle uk-table-divider')
        thead
          tr
            th
            th 블로그 정보
            th 주소
            th 대표 블로그
        tbody
          tr(v-for="blog in blogs")
            td: checkbox(class='uk-checkbox' type='checkbox' :val='blog.name' v-model='checkedNames')
            td {{ blog.title }}
            td {{ blog.url }}
            td {{ blog.default }}
      button#extract(ref='extract' @click='zip(checkedNames)') 시작하기
      button#stop(ref='stop' @click='isRunning = false') 중지
      a#download(href='#' ref='download') 다운받기
      #metainfo
        .url {{ url }}
        .message {{ message }}
        .error {{ error }}
</template>

<script>

import tistory from 'tistory'
import JSZip from 'jszip'
import moment from 'moment'
import Swal from 'sweetalert2'

import Checkbox from '../Checkbox'

import path from 'path'

const electron = require('electron').remote

export default {
  name: 'extractor',
  async mounted () {
    try {
      const { data } = await tistory.blog.info(this.$store.state.tistory.accessToken)

      this.blogs = data.tistory.item.blogs
      this.userProfile = this.blogs.filter(blog => blog.default === 'Y')[0].profileImageUrl
    } catch (e) {
      Swal.fire({ icon: 'error', title: '에러', text: '블로그 정보를 불러올 수 없습니다.' })

      // Redirect '/' to retry login
      this.$router.push('/')
    }
  },
  data () {
    return {
      userProfile: '',
      blogs: [],
      checkedNames: [],
      url: '',
      message: '',
      error: '',
      isRunning: false
    }
  },
  methods: {
    /**
     * Create index.html, insert to folder
     *
     * @param {Document} doc
     * @param {JSZip} postFolder
     * @param {string} pathname
     */
    addHtml (doc, postFolder, pathname) {
      postFolder.file('index.html', new XMLSerializer().serializeToString(doc))
      this.message = path.join(pathname, 'index.html')
    },
    /**
     * Get images from Kakao CDN
     *
     * @param {Document} doc
     * @param {JSZip} postFolder
     * @param {string} pathname
     */
    async addImages (doc, postFolder, pathname) {
      const imageBlocks = doc.querySelectorAll('.imageblock')

      for (let bidx = 0; imageBlocks.length > bidx; bidx++) {
        const img = imageBlocks[bidx].getElementsByTagName('img')[0]

        const sources = img.getAttribute('src').split(',')
        const imgRegex = /kage@(.*)/

        for (let i = 0; i < sources.length; i++) {
          let imageSourceName

          if (imgRegex.test(sources[i])) {
            imageSourceName = `https://blog.kakaocdn.net/dn/${imgRegex.exec(sources[i])[1]}`
          } else {
            imageSourceName = sources[i]
          }
          const response = await fetch(imageSourceName)

          switch (response.status) {
            case 200:
              const imageFilename = bidx + '.' + response.headers.get('Content-Type').substring(6)
              postFolder.file(imageFilename, response.blob())

              if (i > 0) {
                const newImg = doc.createElement('img')
                newImg.setAttribute('src', './' + imageFilename)
                img.parentNode.appendChild(newImg)
              } else {
                img.setAttribute('src', './' + imageFilename)
              }

              this.message = path.join(pathname, imageFilename)
              break
            case 404:
              this.error = `${imageSourceName} 에 해당하는 이미지를 가져오는데 실패했습니다.`
          }
        }
      }
    },
    /**
     * Add Post in Zip
     *
     * @param {JSZip} blogFolder
     * @param {object} blogName
     */
    async addPost (blogFolder, blogName, post) {
      const postFolder = blogFolder.folder(`${post.id}. ` + post.title)

      try {
        const { data } = await tistory.post.read(this.$store.state.tistory.accessToken, { blogName, postId: post.id })

        this.message = path.join(blogName, post.title)
        this.url = post.postUrl

        const parser = new DOMParser()
        const doc = parser.parseFromString(data.tistory.item.content, 'text/html')
        const pathname = this.message

        await this.addImages(doc, postFolder, pathname)
        this.addHtml(doc, postFolder, pathname)
      } catch (e) {
        this.error = `${post.postUrl} 에 해당하는 포스트를 찾을 수 없습니다.`
      }
    },
    /**
     * Add blog in Zip
     *
     * @param {JSZip} zip
     * @param {string} blogName
     */
    async addBlog (zip, blogName) {
      const blogFolder = zip.folder(blogName)

      for (let page = 1; this.isRunning; page++) {
        try {
          const { data } = await tistory.post.list(this.$store.state.tistory.accessToken, { blogName, page })

          if (data.tistory.item.hasOwnProperty('posts')) {
            for (let post of data.tistory.item.posts) {
              if (this.isRunning) {
                await this.addPost(blogFolder, blogName, post)
              } else break
            }
          } else break
        } catch (e) {
          Swal.fire({ icon: 'error', title: '에러', text: `https://${blogName}.tistory.com/?page=${page} 에 해당하는 글 목록을 불러올 수 없습니다.` })
          this.error = `https://${blogName}.tistory.com/?page=${page} 에 해당하는 글 목록을 불러올 수 없습니다.`
          this.isRunning = false

          break
        }
      }
    },
    /**
     * build Zip
     *
     * @param {string[]} checkedNames
     *
     * @return {JSZip} zip
     */
    async buildZip (checkedNames) {
      const zip = new JSZip()

      this.isRunning = true

      for (const blog of checkedNames) {
        if (this.isRunning) {
          await this.addBlog(zip, blog)
        } else break
      }

      return zip
    },
    /**
     * Create Zip and Downloadable URL
     *
     * @param {JSZip} zip
     */
    createZip (zip) {
      zip.generateAsync({ type: 'blob' }).then(blob => {
        const file = new Blob([ blob ], { type: 'application/zip' })
        const fileURL = URL.createObjectURL(file)

        this.$refs.download.setAttribute('href', fileURL)
        this.$refs.download.setAttribute('download', moment().format('X_YYYY_MM_DD'))
        this.$refs.download.style.display = 'inline-block'

        Swal.fire({ icon: 'success', title: '성공', text: '티스토리 블로그의 백업이 완료되었습니다. 다운로드 버튼을 눌러 결과를 확인해보세요.' })
      })
    },
    /**
     * Create blogs post backup with Zip
     *
     * @param {string} checkedNames
     */
    async zip (checkedNames) {
      if (checkedNames.length <= 0) {
        Swal.fire({ icon: 'error', title: '에러', text: '티스토리 블로그를 백업하려면 블로그 선택해야합니다.' })
        this.error = '티스토리 블로그를 백업하려면 블로그 선택해야합니다.'

        return
      }

      const zip = await this.buildZip(checkedNames)
      this.isRunning ? this.createZip(zip) : Swal.fire({ icon: 'warning', title: '경고', text: '백업이 중지되었습니다.' })

      this.isRunning = false
    }
  },
  components: {
    Checkbox
  }
}
</script>

<style lang="stylus">
  #extractor
    #nav
      position fixed
      top 0
      right 0
      width 100%
      padding 10px 20px
      box-sizing border-box
      a
        padding 0
      .uk-navbar-left
        li
          margin-right 12px
      .uk-navbar-right
        img
          width 40px
          height 40px
          border-radius 50%
    #main
      width 820px
      .warning
        font-size .92rem
        font-weight 600
        color black
      #extract, #download, #stop
        color white
        padding 15px 30px
        border-radius 5px
        box-shadow 0 0 0 1px rgba(0, 0, 0, .1), 0 2px 5px rgba(0, 0, 0, .1)
        font-weight 500
        border none
        cursor pointer
        font-size 1rem
      #extract, #stop
        margin-right 12px
      #extract
        background-color #ed5207
        display inline-block
      #stop
        background-color red
      #download
        background-color black
        display none
      table
        td
          &:nth-child(2)
            font-weight 600
            color black
      #metainfo
        margin 15px 0
        *
          word-break break-all
        .url
          margin-bottom 3px
          font-size .9rem
        .message
          color black
          font-weight 500
        .error
          margin-top 10px
          color red
          margin 5px 0
</style>
