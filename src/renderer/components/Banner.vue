<template lang="pug">
  footer
    .copyright ⓒ 2020-2021. #[a(href='https://github.com/pronist' target='_blank') 정상우]. all rights reserved
</template>

<script>
export default {
  name: 'banner'
}
</script>

<style lang="stylus">
  $Color = alpha(black, .5)

  footer
    position fixed
    bottom 25px
    right 0
    width 100%
    color $Color
    text-align center
    font-size .82rem
    a
      text-decoration underline !important
      color $Color
      &:hover
        color black
</style>
