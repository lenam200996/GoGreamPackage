<template>
  <div class="drr" :style="style"
      :class="classObject"
      @dblclick="dblclick($event)"
      @mousedown="bodyMouseDown($event)"
      @touchstart.stop.prevent="bodyMouseDown($event)"
      @mousemove="hoverDrr"
      @mouseout="outDrr"
       >
    <slot></slot>
    <!-- <span>{{poXY}}</span> -->
    <div
      v-for="(stick,index) in sticks"
      class="drr-stick"
      :class="['drr-stick-' + stick, resizable ? '' : 'not-resizable']"
      @mousedown.stop.prevent="stickDown(stick, $event)"
      @touchstart.stop.prevent="stickDown(stick, $event)"
      :style="drrStick(stick)"
      :key="index"
      >
    </div>
    <div class="ro-stick-handle" v-if="rotatable"></div>
    <span v-if="isAlignGrid&&active&&draggable&&resizable" class="vertical-line vertical-line-left" :style="style_line_left"></span>
    <span v-if="isAlignGrid&&active&&draggable&&resizable" class="vertical-line vertical-line-right" :style="style_line_right"></span>
    <span v-if="isAlignGrid&&active&&draggable&&resizable" class="horizontal-line horizontal-line-top" :style="style_line_top"></span>
    <span v-if="isAlignGrid&&active&&draggable&&resizable" class="horizontal-line horizontal-line-bottom" :style="style_line_bottom" ></span>
    <span v-if="isAlignGrid&&active&&draggable&&resizable" class="vertical-line" :style="style_line_center_v" ></span>
    <span v-if="isAlignGrid&&active&&draggable&&resizable" class="horizontal-line" :style="style_line_center_h" ></span>    
    <span v-if="!active&&isHover&&draggable&&resizable" class="type-name">{{typeName}}</span>
    <div v-if="isAlignGrid&&active&&draggable&&resizable" class="rect-display">
      <div class="rect-display-i">X: {{parseFloat(style.left.replace('px','')).toFixed(1)}}</div><div class="rect-display-i">Y: {{parseFloat(style.top.replace('px','')).toFixed(1)}}</div><br><div class="rect-display-i">W: {{parseFloat(style.width.replace('px','')).toFixed(1)}}</div><div class="rect-display-i">H: {{parseFloat(style.height.replace('px','')).toFixed(1)}}</div>
    </div>
  </div>
</template>

<script>
  import Vector from '@minogin/vector'

  const _ = require('lodash');

  const stickSize = 8;
  const roStickSize = 20;
  const styleMapping = {
    y: {
      t: 'top',
      m: 'marginTop',
      b: 'bottom',
    },
    x: {
      l: 'left',
      m: 'marginLeft',
      r: 'right',
    }
  };

  export default {
    name: 'drr',
    props: {
      typeName:{
        type:String,
        default:"type"
      },
      parentId :{
        type :Number,
      },
      column :{
        type :Number,
      },
      id: {
        type : Number,
        required : true,
      },
      x: {
        type: Number,
        required: false,
        default: 0,
        validator: function (val) {
          return typeof val === 'number'
        }
      },
      y: {
        type: Number,
         default: 0,
        required: false,
        validator: function (val) {
          return typeof val === 'number'
        }
      },
      w: {
        type: Number,
        required: false,
        validator: function (val) {
          return val > 0
        }
      },
      h: {
        type: Number,
        required: false,
        validator: function (val) {
          return val > 0
        }
      },
      angle: {
        type: Number,
        default: 0,
        validator: function (val) {
          return typeof val === 'number'
        }
      },
      selected: {
        type: Boolean,
        default: false
      },
      selectable: {
        type: Boolean,
        default: true
      },
      draggable: {
        type: Boolean,
        default: true
      },
      resizable: {
        type: Boolean,
        default: true
      },
      rotatable: {
        type: Boolean,
        default: true
      },
      hasActiveContent: {
        type: Boolean,
        default: false
      },
      aspectRatio: {
        type: Boolean,
        default: false
      },
      dragHandle: {
        type: String,
        default: null
      },
      dragCancel: {
        type: String,
        default: null
      },
      outerBound: {
        type: Object
      },
      innerBound: {
        type: Object
      }
    },

    data: function () {
      return {
        active: this.selected,
        contentActive: false,
        cx: this.x,
        cy: this.y,
        width: this.w,
        height: this.h,
        rotation: this.angle,
        bodyDrag: false,
        dragged: false,
        resized: false,
        rotated: false,
        change: false,
        arrayX_left : [],
        arrayY_top : [],
        arrayX_right :[],
        arrayY_bottom : [],
        arrayX_center : [],
        arrayY_center : [],
        arrayIdItem : [],
        isAlignGrid: false,
        isHover:false,
        bg: null
      }
    },

    computed: {
      getPositionGrid:function(){
        return this.$store.getters.getArrayGridElement({id: this.id,parentId: this.parentId,column:this.column})
      },
      style_line_center_v:function(){
        var display  = 'none'
        if(this.arrayX_center.includes(parseInt(this.style.left.replace('px','')) + this.width/2)){
          display = 'block'
        }else{
          display = 'none'
        }
        return {
          height :  window.innerHeight +'px',
          top : ('-'+this.style.top),
          left : parseInt(this.style.width.replace('px',''))/2 + 'px',
          display
        }
      },
      style_line_center_h:function(){
        return{
          width : window.innerWidth +'px',
          left : '-'+this.style.left,
          top : parseInt(this.style.height.replace('px',''))/2 + 'px',
          display: this.arrayY_center.includes(parseInt(this.style.top.replace('px','')) +this.height/2) ? 'block' :'none'
        }
      },
      style_line_left:function(){
        return {
          height : window.innerHeight +'px',
          top : ('-'+this.style.top),
          display : 
          this.arrayX_left.includes((typeof this.style.left)=='string'?parseInt(this.style.left.replace('px','')):this.style.left)?'block':'none'
        }
      },
      style_line_top:function(){
        return{
          width : window.innerWidth +'px',
          left : '-'+this.style.left,
          display : this.arrayY_top.includes((typeof this.style.top)=='string'?parseInt(this.style.top.replace('px','')):this.style.top )?'block':'none'
        }
      },
      style_line_right:function(){
        return {
        height : window.innerHeight +'px',
        top : ('-'+this.style.top),
        display : this.arrayX_right.includes(((typeof this.style.left)=='string'?parseInt(this.style.left.replace('px','')):this.style.left) +((typeof this.style.width)=='string'?parseInt(this.style.width.replace('px','')):this.style.width))?'block':'none'
        }
      },
      style_line_bottom:function(){
        return{
          width : window.innerWidth +'px',
          left : '-'+this.style.left,
          display: this.arrayY_bottom.includes(((typeof this.style.top)=='string'?parseInt(this.style.top.replace('px','')):this.style.top )+((typeof this.style.height)=='string'?parseInt(this.style.height.replace('px','')):this.style.height))?'block':'none'
        }
      },
      sticks() {
        let sticks = []
        if (this.resizable)
          sticks.push('tl', 'tr', 'br', 'bl')
        if (this.rotatable)
          sticks.push('ro')
        return sticks
      },

      classObject() {
        return {
          active: this.active,
          inactive: !this.active,
          dragging: this.bodyDrag,
          'content-active': this.contentActive
        }
      },

      style() {
            return {
              left: (this.cx - this.width / 2) + 'px',
              top: (this.cy - this.height / 2) + 'px',
              width: this.width + 'px',
              height: this.height + 'px',
              transform: 'rotate(' + this.rotation + 'deg)',
              rotation : this.rotation,
              backgroundColor: this.bg
            }
      },

      drrStick() {
        return (stick) => {
          const stickStyle = {
            width: `${stickSize}px`,
            height: `${stickSize}px`,
          };
          if (stick == 'ro') {
            stickStyle['top'] = `${-stickSize / 2 - roStickSize}px`;
            stickStyle['marginLeft'] = `${-stickSize / 2 + 1}px`;
          }
          else {
            stickStyle[styleMapping.y[stick[0]]] = `${-stickSize / 2}px`;
            stickStyle[styleMapping.x[stick[1]]] = `${-stickSize / 2}px`;
          }
          return stickStyle;
        }
      },

      rect() {
        return {
          x: this.cx,
          y: this.cy,
          w: this.width,
          h: this.height,
          angle: this.rotation
        }
      },
      poXY(){
        return {
          minx: this.cx - this.x ,
          miny: this.cy -this.y,
          maxx: this.cx - this.x+ this.width,
          maxy: this.cy - this.y+ this.height
        }
      }
    },

    watch: {
      isHover:function(val){
        if(val && !this.active){
          this.bg = '#3899ec14'
        }else{
          this.bg = null
        }
      },
      getPositionGrid:function(val){
        this.arrayX_left = []
        this.arrayY_top = []
        this.arrayX_right = []
        this.arrayY_bottom = []
        this.arrayX_center = []
        this.arrayY_center = []
        this.arrayIdItem = []
        val.forEach(item => {
          this.arrayX_left.push(item.x_left)
          this.arrayY_top.push(item.y_top)
          this.arrayX_right.push(item.x_right)
          this.arrayY_bottom.push(item.y_bottom)
          this.arrayX_center.push(item.x_left + item.x_center)
          this.arrayY_center.push(item.y_top + item.y_center)
          this.arrayIdItem.push(item.id)
        });
      },
      style:function( val,old){
          this.$store.commit('updatePositionElement',{id : this.id ,val : val})
      }, 
      rect:function(val,old){
        if(val.x != old.x || val.y != old.y || val.h != old.h || val.w != old.w || val.angle != old.angle ){
          this.change = true
        }
      },
      active(val) {
        if (val)
          this.$emit('select',this.id);
        else
          this.$emit('deselect');
      },

      selected(val) {
        this.active = val
      },

      hasActiveContent: {
        handler: function(val) {
          if (val) {
            this.$on('content-active', this.onContentActive)
            this.$on('content-inactive', this.onContentInactive)
          }
          else {
            this.$off('content-active')
            this.$off('content-inactive')
            if (this.contentActive)
              this.onContentInactive()
          }
        },
        immediate: true
      },

      x() {
        if (this.stickDrag || this.bodyDrag)
          return
        this.cx = this.x;
      },

      y() {
        if (this.stickDrag || this.bodyDrag)
          return
        
        this.cy = this.y;
      },

      w() {
        if (this.stickDrag || this.bodyDrag)
          return

        this.currentStick = ['m', 'r'];

        this.width = this.w
      },

      h() {
        if (this.stickDrag || this.bodyDrag)
          return

        this.currentStick = ['b', 'm'];

        this.height = this.h
      },

      angle() {
        if (this.stickDrag || this.bodyDrag)
          return

        this.rotation = this.angle
      }
    },

    created: function () {
      this.stickDrag = false;
      this.bodyDrag = false;
      this.stickStartPos = { mouseX: 0, mouseY: 0, x: 0, y: 0, w: 0, h: 0 };

      this.currentStick = [];
    },

    mounted: function () {
      this.parentElement = this.$el.parentNode;
      // this.parentWidth = this.parentW ? this.parentW : this.parentElement.clientWidth;
      // this.parentHeight = this.parentH ? this.parentH : this.parentElement.clientHeight;

      document.documentElement.addEventListener('mousemove', this.move);
      document.documentElement.addEventListener('mouseup', this.up);
      document.documentElement.addEventListener('mouseleave', this.up);

      document.documentElement.addEventListener('mousedown', this.deselect);

      document.documentElement.addEventListener('touchmove', this.move, true);
      document.documentElement.addEventListener('touchend touchcancel', this.up, true);
      document.documentElement.addEventListener('touchstart', this.up, true);

      if (this.dragHandle) {
        let dragHandles = Array.prototype.slice.call(this.$el.querySelectorAll(this.dragHandle));
        for (let i in dragHandles) {
          dragHandles[i].setAttribute('data-drag-handle', this._uid);
        }
      }

      if (this.dragCancel) {
        let cancelHandles = Array.prototype.slice.call(this.$el.querySelectorAll(this.dragCancel));
        for (let i in cancelHandles) {
          cancelHandles[i].setAttribute('data-drag-cancel', this._uid);
        }
      }

     
    },

    beforeDestroy: function () {
      document.documentElement.removeEventListener('mousemove', this.move);
      document.documentElement.removeEventListener('mouseup', this.up);
      document.documentElement.removeEventListener('mouseleave', this.up);

      document.documentElement.removeEventListener('mousedown', this.deselect);

      document.documentElement.removeEventListener('touchmove', this.move, true);
      document.documentElement.removeEventListener('touchend touchcancel', this.up, true);
      document.documentElement.removeEventListener('touchstart', this.up, true);
    },

    methods: {
      hoverDrr(){
        this.isHover = true
      },
      outDrr(){
        this.isHover = false
      },
      // bodyMouseEnter:function(ev){
      //   
        
      // },
      dblclick(e) {
        this.$emit('content-active')
      },

      onContentActive() {
        this.contentActive = true
        this.active = false
        for (const child of this.$children) {
          child.$emit('active')
        }
      },

      onContentInactive() {
        this.contentActive = false
        this.active = true
        for (const child of this.$children) {
          child.$emit('inactive')
        }
      },

      deselect() {
        this.$emit('deselect')
        this.active = false
      },
      move(ev) {
        if (!this.stickDrag && !this.bodyDrag) {
          return
        }
        ev.stopPropagation();

        if (this.stickDrag) {
          this.stickMove(ev);
        }
        if (this.bodyDrag) {
          this.bodyMove(ev)
        }
        var xLeft = (typeof this.style.left)=='string'?(parseInt(this.style.left.replace('px',''))):(this.style.left) 
        var xRight =  ((typeof this.style.left)=='string'?parseInt(this.style.left.replace('px','')):this.style.left) +((typeof this.style.width)=='string'?parseInt(this.style.width.replace('px','')):(this.style.width))
        var yTop = (typeof this.style.top)=='string'?(parseInt(this.style.top.replace('px',''))):(this.style.top) 
        var yBottom = ((typeof this.style.top)=='string'?parseInt(this.style.top.replace('px','')):this.style.top )+((typeof this.style.height)=='string'?parseInt(this.style.height.replace('px','')):this.style.height)
        var xCenter = parseInt(this.style.left.replace('px','')) + this.width/2
        var yCenter = parseInt(this.style.top.replace('px','')) + this.height/2
        var incX = 0
        var incY = 0
        var idAuto = null
        this.arrayX_center.forEach((itemXCenter,index) =>{
          if(Math.abs(itemXCenter - xCenter) < 5){
            incX= (itemXCenter - xCenter)
            idAuto = this.arrayIdItem[index]
          }
        })

        this.arrayY_center.forEach((itemYCenter,index) => {
          if(Math.abs(itemYCenter - yCenter) < 5){
            incY= (itemYCenter - yCenter)
            idAuto = this.arrayIdItem[index]
          }
        })

        this.arrayX_left.forEach((itemXL,index) => {
          if( Math.abs(itemXL - xLeft) < 5 ){
            incX= (itemXL - xLeft)
            idAuto = this.arrayIdItem[index]
          }
         
        })
        this.arrayX_right.forEach((itemXR,index)=>{
          if(Math.abs(itemXR - xRight) < 5){
            incX= (itemXR - xRight)
            idAuto = this.arrayIdItem[index]
          }
          
        })
        this.arrayY_top.forEach((itemYT,index) =>{
          if(Math.abs(itemYT - yTop) <5){
            incY= (itemYT - yTop)
            idAuto = this.arrayIdItem[index]
          }
          
        })
        this.arrayY_bottom.forEach((itemYB,index)=>{
          if(Math.abs(itemYB - yBottom) < 5){
            incY= (itemYB - yBottom)
            idAuto = this.arrayIdItem[index]
          }
          
        })

        this.cx += incX
        this.cy += incY
        this.$store.commit('setAutoAlignBlockId',idAuto)


      },

      up(ev) {
       
        if (this.stickDrag) {
          this.stickUp(ev);
        }
        if (this.bodyDrag) {
          this.bodyUp(ev)
        }
        if(this.change){
           this.$store.commit('bindingPosition',{id : this.id ,val : this.rect})
        }
        this.isAlignGrid = false
        
      },

      bodyMouseDown: function (e) {
        this.isAlignGrid = true
        this.$store.commit('enableUndo')
        this.change = false
        if (this.contentActive || !this.selectable) {
          return
        }
        else {
          e.stopPropagation()
          e.preventDefault()
        }

        let target = e.target || e.srcElement;

        this.active = true;

        if (e.button && e.button !== 0) {
          return
        }

        this.$emit('clicked', e);

        if (!this.draggable || !this.active) {
          return
        }

        if (this.dragHandle && target.getAttribute('data-drag-handle') !== this._uid.toString()) {
          return
        }

        if (this.dragCancel && target.getAttribute('data-drag-cancel') === this._uid.toString()) {
          return
        }

        this.bodyDrag = true
        this.dragged = false

        this.dragStartEmitted = false
        this.startRect = _.cloneDeep(this.rect)

        this.stickStartPos.mouseX = e.pageX || e.touches[0].pageX
        this.stickStartPos.mouseY = e.pageY || e.touches[0].pageY

        this.stickStartPos.cx = this.cx
        this.stickStartPos.cy = this.cy
      },

      bodyMove(ev) {
        

        const stickStartPos = this.stickStartPos;

        const newPos = {
          mouseX: ev.pageX || ev.touches[0].pageX,
          mouseY: ev.pageY || ev.touches[0].pageY
        }
        const delta = {
          x: newPos.mouseX - stickStartPos.mouseX,
          y: newPos.mouseY - stickStartPos.mouseY
        }

        let newcx = stickStartPos.cx + delta.x
        let newcy = stickStartPos.cy + delta.y
        let x1 = newcx - this.width / 2
        let y1 = newcy - this.height / 2
        let x2 = newcx + this.width / 2
        let y2 = newcy + this.height / 2

        if (this.outerBound && this.rotation == 0) {
          let bx1 = this.outerBound.x - this.outerBound.w / 2
          let by1 = this.outerBound.y - this.outerBound.h / 2
          let bx2 = this.outerBound.x + this.outerBound.w / 2
          let by2 = this.outerBound.y + this.outerBound.h / 2
          if (x1 < bx1)
            delta.x -= x1 - bx1
          if (x2 > bx2)
            delta.x -= x2 - bx2
          if (y1 < by1)
            delta.y -= y1 - by1
          if (y2 > by2)
            delta.y -= y2 - by2
        }

        if (this.innerBound && this.rotation == 0) {
          let bx1 = this.innerBound.x - this.innerBound.w / 2
          let by1 = this.innerBound.y - this.innerBound.h / 2
          let bx2 = this.innerBound.x + this.innerBound.w / 2
          let by2 = this.innerBound.y + this.innerBound.h / 2
          if (x1 > bx1)
            delta.x -= x1 - bx1
          if (x2 < bx2)
            delta.x -= x2 - bx2
          if (y1 > by1)
            delta.y -= y1 - by1
          if (y2 < by2)
            delta.y -= y2 - by2
        }

        this.cx = stickStartPos.cx + delta.x
        this.cy = stickStartPos.cy + delta.y

        if (!this.dragStartEmitted) {
          this.$emit('dragstart', this.startRect);
          this.dragStartEmitted = true
        }

        this.dragged = true
        this.$emit('drag', this.rect);
      },

      bodyUp() {
        this.bodyDrag = false;
        if (this.dragged) {
          this.$emit('dragstop', this.rect, this.startRect);
          this.$emit('change', this.rect);
        }

        this.stickStartPos = { mouseX: 0, mouseY: 0, x: 0, y: 0, w: 0, h: 0 };
  
      },

      stickDown: function (stick, ev) {
        this.isAlignGrid = true
         this.$store.commit('enableUndo')
         this.change = false
        if (!this.resizable || !this.active)
          return

        this.resizeStartEmitted = false
        this.rotateStartEmitted = false
        this.startRect = _.cloneDeep(this.rect)

        this.stickDrag = true;
        this.resized = false
        this.rotated = false
        this.stickStartPos.mouseX = ev.pageX || ev.touches[0].pageX;
        this.stickStartPos.mouseY = ev.pageY || ev.touches[0].pageY;
        this.stickStartPos.cx = this.cx;
        this.stickStartPos.cy = this.cy;
        this.stickStartPos.width = this.width;
        this.stickStartPos.height = this.height;
        this.stickStartPos.rotation = this.rotation;
        this.currentStick = stick
      },

      stickMove(ev) {
        this.autoAlignBlock

        const stickStartPos = this.stickStartPos;

        let delta = new Vector(
          (ev.pageX || ev.touches[0].pageX) - stickStartPos.mouseX,
          (ev.pageY || ev.touches[0].pageY) - stickStartPos.mouseY
        )


        if (this.currentStick == 'ro') {
          let up = new Vector(0, -(this.height) / 2 - roStickSize)
          let rotationRad = Vector.rad(stickStartPos.rotation);
          up = up.rotate(rotationRad)
          let v = up.add(delta)

          if (!this.rotateStartEmitted) {
            this.$emit('rotatestart', this.startRect);
            this.rotateStartEmitted = true
          }

          this.rotation = Vector.deg(v.angle()) + 90
          this.rotated = true
          this.$emit('rotate', this.rect);
        }
        else {
          let dirX = this.currentStick[1] == 'r' ? 1 : -1
          let dirY = this.currentStick[0] == 'b' ? 1 : -1

          let phi = Vector.rad(stickStartPos.rotation);
          let p
          if (this.aspectRatio) {
            let axis = new Vector(dirX * stickStartPos.width / 2, dirY * stickStartPos.height / 2)
            axis = axis.rotate(phi).unit()
            p = axis.mul(axis.mul(delta))
          }
          else {
            p = delta
          }

          let pn = p.rotate(-phi)

          let newcx = stickStartPos.cx + p.x / 2
          let newcy = stickStartPos.cy + p.y / 2
          let newwidth = stickStartPos.width + dirX * pn.x
          let newheight = stickStartPos.height + dirY * pn.y
          let x1 = newcx - newwidth / 2
          let y1 = newcy - newheight / 2
          let x2 = newcx + newwidth / 2
          let y2 = newcy + newheight / 2

          if (this.outerBound && this.rotation == 0) {
            let bx1 = this.outerBound.x - this.outerBound.w / 2
            let by1 = this.outerBound.y - this.outerBound.h / 2
            let bx2 = this.outerBound.x + this.outerBound.w / 2
            let by2 = this.outerBound.y + this.outerBound.h / 2
            let dx = 0
            let dy = 0
            if (x1 < bx1)
              dx = bx1 - x1
            if (x2 > bx2)
              dx = bx2 - x2
            if (y1 < by1)
              dy = by1 - y1
            if (y2 > by2)
              dy = by2 - y2

            if (dx != 0 || dy != 0) {
              if (this.aspectRatio) {
                if (dx / p.x < dy / p.y) {
                  p.y += dx * p.y / p.x
                  p.x += dx
                }
                else {
                  p.x += dy * p.x / p.y
                  p.y += dy
                }
              }
              else {
                p.x += dx
                p.y += dy
              }
            }
          }

          if (this.innerBound && this.rotation == 0) {
            let bx1 = this.innerBound.x - this.innerBound.w / 2
            let by1 = this.innerBound.y - this.innerBound.h / 2
            let bx2 = this.innerBound.x + this.innerBound.w / 2
            let by2 = this.innerBound.y + this.innerBound.h / 2
            let dx = 0
            let dy = 0
            if (x1 > bx1)
              dx = bx1 - x1
            if (x2 < bx2)
              dx = bx2 - x2
            if (y1 > by1)
              dy = by1 - y1
            if (y2 < by2)
              dy = by2 - y2

            if (dx != 0 || dy != 0) {
              if (this.aspectRatio) {
                if (dx / p.x < dy / p.y) {
                  p.y += dx * p.y / p.x
                  p.x += dx
                }
                else {
                  p.x += dy * p.x / p.y
                  p.y += dy
                }
              }
              else {
                p.x += dx
                p.y += dy
              }
            }
          }

          this.cx = stickStartPos.cx + p.x / 2
          this.cy = stickStartPos.cy + p.y / 2
          pn = p.rotate(-phi)
          this.width = stickStartPos.width + dirX * pn.x
          this.height = stickStartPos.height + dirY * pn.y

          if (!this.resizeStartEmitted) {
            this.$emit('resizestart', this.startRect);
            this.resizeStartEmitted = true
          }

          this.resized = true
          this.$emit('resize', this.rect);
        }
      },

      stickUp() {
        this.stickDrag = false;
        this.stickStartPos = {
          mouseX: 0,
          mouseY: 0,
          x: 0,
          y: 0,
          w: 0,
          h: 0
        };

        if (this.resized) {
          this.$emit('resizestop', this.rect, this.startRect);  // TODO
          this.$emit('change', this.rect);
        }

        if (this.rotated) {
          this.$emit('rotatestop', this.rect, this.startRect);
          this.$emit('change', this.rect);
        }

       
        },  
    },
  }

</script>

<style scoped>
  /*TODO less */

  .drr {
    position: absolute;
    box-sizing: border-box;
    cursor: pointer;
  }

  .drr:hover:before {
    content: '';
    width: 100%;
    height: 100%;
    position: absolute;
    top: 0;
    left: 0;
    box-sizing: border-box;
    outline: 2px solid #ffffff;

  }

  .drr.active:before {
    content: '';
    width: 100%;
    height: 100%;
    position: absolute;
    top: 0;
    left: 0;
    box-sizing: border-box;
    outline: 2px dashed lightskyblue;
  }

  .drr-stick {
    box-sizing: border-box;
    position: absolute;
    font-size: 1px;
    background: #ffffff;
    border: 1px solid #6c6c6c;
    box-shadow: 0 0 2px #bbb;
  }

  .drr-stick:hover {
    border-color: lightskyblue;
  }

  .inactive > .drr-stick {
    display: none;
  }

  .drr-stick-tl, .drr-stick-br {
    cursor: nwse-resize;
  }

  .drr-stick-tm, .drr-stick-bm {
    left: 50%;
    cursor: ns-resize;
  }

  .drr-stick-tr, .drr-stick-bl {
    cursor: nesw-resize;
  }

  .drr-stick-ml, .drr-stick-mr {
    top: 50%;
    cursor: ew-resize;
  }

  .drr-stick-ro {
    left: 50%;
    cursor: ew-resize;
    border-radius: 4px;
    width: 10px;
    height: 10px;
  }

  .ro-stick-handle {
    left: 50%;
    top: -16px;
    box-sizing: border-box;
    position: absolute;
    font-size: 1px;
    background: #ffffff;
    border: 1px solid #6c6c6c;
    box-shadow: 0 0 2px #bbb;
    width: 0px;
    height: 16px;
  }

  .inactive > .ro-stick-handle {
    display: none;
  }

  .drr-stick.not-resizable {
    display: none;
  }

  .content-active {
    border: 2px solid lightskyblue; /*TODO*/
  }
  .vertical-line{
    width: 1px;
    background: rgb(255, 0, 170);
    position: absolute;
    z-index: 999999;
  }
  .vertical-line-left{
    left: -2px;
  }
  .vertical-line-right{
    right: -2px;
  }
  .horizontal-line{
    height: 1px;
    background: rgb(255, 0, 170);
    position: absolute;
    z-index: 999999;
  }
  .horizontal-line-top{
    top: -2px;
  }
  .horizontal-line-bottom{
    bottom: -2px;
  }

  .rect-display{
    position: absolute;
    top: -48px;
    left:-60px;
    color: #ffffff;
    border-radius: 10px;
    width: 140px;
  }

  .rect-display-i{
    width: 60px;
    font-size: 10px;
    border-radius: 7px;
    background-color: #585858;
    float: left;
    margin: 1px 1px;
  }

  .type-name{
    position: absolute;
    font-size: 14px;
    background-color: #EB6641;
    color: #ffffff;
    top: -20px;
    padding: 0 5px;
    left: 0;
    z-index: 9999999;
  }
</style>
