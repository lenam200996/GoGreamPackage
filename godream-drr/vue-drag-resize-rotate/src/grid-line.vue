<template>
    <div>
        <span v-if="active&&draggable&&resizable" class="vertical-line-left" :style="style_line_left"></span>
        <span v-if="active&&draggable&&resizable" class="vertical-line-right" :style="style_line_right"></span>
        <span v-if="active&&draggable&&resizable" class="vertical-line-top" :style="style_line_top"></span>
        <span v-if="active&&draggable&&resizable" class="vertical-line-bottom" :style="style_line_bottom" ></span>
        <span v-if="active&&draggable&&resizable" class="rect-display">x: {{style.left}} | y: {{style.top}} </span>
    </div>
</template>

<script>
    export default {
        
    }
</script>

<style lang="scss" scoped>

</style>