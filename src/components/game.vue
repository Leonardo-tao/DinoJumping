<template>
  <div class="game">
    <canvas id="canvas" ref="canvas" :width="canvasWidth" :height="canvasHeight" />
    <audio ref="jumpSound" crossorigin="anonymous" src="https://circumvent-cors.herokuapp.com/https://leonardotao.github.io/DinoJumping/jump.mp3" preload="auto"></audio>
    <audio ref="defeatSound" crossorigin="anonymous" src="https://circumvent-cors.herokuapp.com/https://leonardotao.github.io/DinoJumping/defeat.mp3" preload="auto"></audio>
  </div>
</template>

<script>
export default {
  name: 'playGame',
  data() {
    return {
      ctx: null,
      canvasWidth: 1200,
      canvasHeight: 170,
      dinoX: 10,
      dinoY: 0,
      isClicked: false,
      clickEvent: null,
      keydownEvent: null,
      speed: 0,                   // 画面移动速度
      groundX: 0,                 // 地面坐标
      stage: 0,                   // 游戏阶段
      isRunning: false,           // 地面是否开始移动
      isDied: false,
      currentLeg: 'left',         // 当前小恐龙腿的状态
      currentSwing: 'bird_up',    // 当前家雀翅膀的状态
      legTimer: null,             // 控制左右脚切换的定时器
      swingTimer: null,           // 控制翅膀切换的定时器
      scoreTimer: null,           // 控制增加分数的定时器
      pace: 100,                  // 步频
      swing: 200,                 // 扇翅膀
      isJumping: false,           // 跳跃状态
      isHanging: false,           // 悬空状态
      jumpHeight: 0,              // 当前跳跃高度
      maxHeight: 100,             // 最大跳跃高度
      gravity: 5,                 // 重力加速度
      minGravity: 1,              // 最小重力加速度
      gravityIndex: 1,            // 重力增长系数
      barriers: [],
      clouds: [],
      birds: [],
      barriers_list: [
        {
          name: 'tree1',
          width: 16,
          height: 34
        },
        {
          name: 'tree2',
          width: 49,
          height: 49
        },
        {
          name: 'tree3',
          width: 24,
          height: 47
        },
        {
          name: 'tree4',
          width: 47,
          height: 46
        },
        {
          name: 'tree5',
          width: 23,
          height: 45
        },
        {
          name: 'tree6',
          width: 70,
          height: 48
        }
      ],
      lastBarrierX: 1200,
      lastCloudX: 1200,
      scores: [1, 0, 0, 0, 0, 0],
      score: 0,
      loadedImages: {},           // 存储加载后的图片
      images: {
        initial: "initial",
        start: "start",
        die: "die",
        ground: "ground",
        left: "runl",
        right: "runr",
        jump: "start",
        bird_up: "bird_up", 
        bird_down: "bird_down",
        // bird_up: "111", 
        // bird_down: "222",
        tree1: "tree1",
        tree2: "tree2",
        tree3: "tree3",
        tree4: "tree4",
        tree5: "tree5",
        tree6: "tree6",
        over: "endText",
        restart: "restart",
        cloud: "cloud",
        "0":"0",
        "1":"1",
        "2":"2",
        "3":"3",
        "4":"4",
        "5":"5",
        "6":"6",
        "7":"7",
        "8":"8",
        "9":"9"
      },
      JUMP_STATE_NONE: 0,
      JUMP_STATE_ASCENDING: 1,
      JUMP_STATE_HANGING: 2,
      JUMP_STATE_DESCENDING: 3,
      jumpState: this.JUMP_STATE_NONE,
      deltaTime: 0,
      lastUpdateTime: 0,
      jumpStartTime: 0,
    }
  },
  computed: {
    groundW: function() {return this.loadedImages['ground'].width},
    groundH: function() {return this.loadedImages['ground'].height},
    dinoW: function() {return this.loadedImages['initial'].width},
    dinoH: function() {return this.loadedImages['initial'].height},
    groundY: function() {return this.canvasHeight - this.loadedImages['ground'].height - this.dinoH},
    treeY: function() {
      return this.canvasHeight - this.loadedImages['ground'].height
    }
  },
  methods: {
    // 绘制图片
    drawPic(pic, x, y, w, h) {
      let img = this.loadedImages[pic] // 使用已加载的图片

      if (!img) return // 如果图片尚未加载，直接返回

      if (w === undefined) w = img.width
      if (h === undefined) h = img.height
      if (y === undefined) y = this.groundY
      if (x === undefined) x = 0

      this.ctx.drawImage(img, x, y, w, h)
      // 绘制边框，测试碰撞边界
      // this.drawBorder(x, y, w, h)
    },
    clickHandle(e) {
      this.isClicked = true
      this.handleKeyDown(e)
      if(this.isDied) {
        this.resetGame()
        this.isClicked = false
      }
    },
    /*  测试碰撞边界
    drawBorder(x, y, w, h, borderColor = 'black', borderSize = 2) {
      // 保存当前的绘图状态
      this.ctx.save();

      // 设置边框颜色和宽度
      this.ctx.strokeStyle = borderColor;
      this.ctx.lineWidth = borderSize;

      // 绘制边框矩形
      this.ctx.strokeRect(x, y, w, h);

      // 恢复之前的绘图状态
      this.ctx.restore();
    },
    */
    // 更新地面位置
    updateGroundPosition() {
      this.groundX -= this.speed

      if (this.groundX <= -this.canvasWidth) {
        this.groundX = 0
      }
    },
    // 处理键盘函数
    handleKeyDown(e) {
      if (!this.isRunning && (e.key === 'Enter' || this.isClicked)) {
        this.isRunning = true
        this.speed = 2
        this.play() // 启动动画
        this.scrollTimer = setInterval(this.updateScore, 8)
        this.isClicked = false
      }

      if (this.isRunning && !this.isJumping && (e.key === ' ' || this.isClicked)) { // 按下空格键
        this.isJumping = true // 标记为正在跳跃
        this.$refs.jumpSound.play()
        this.jumpHeight = 0 // 重置跳跃高度
        this.jumpState = this.JUMP_STATE_ASCENDING
        this.jumpStartTime = Date.now() // 记录跳跃开始时间
        this.lastUpdateTime = Date.now() // 初始化最后更新时间
        this.isClicked = false
      }
    },
    // 加速函数
    accelerate() {
      this.speed += 0.001
      this.pace -= 1
    },
    updateJump() {
      if (this.isJumping) {
        if (this.jumpState === this.JUMP_STATE_ASCENDING) {
          if(this.gravity >= this.minGravity){
            this.gravityIndex += ( this.jumpHeight + 1) * 0.0001
            this.gravity -= ( this.gravity - this.minGravity ) * this.gravityIndex / (( this.maxHeight / this.gravity ))
          }
          this.jumpHeight += this.gravity;
          if (this.jumpHeight >= this.maxHeight) {
            this.jumpState = this.JUMP_STATE_HANGING;
            this.jumpHeight = this.maxHeight;
            this.hangingStartTime = Date.now(); // 记录滞空开始时间
          }
        } else if (this.jumpState === this.JUMP_STATE_HANGING) {
          const hangingDuration = Date.now() - this.hangingStartTime;
          if (hangingDuration > 110) { // 假设滞空时间为110毫秒
            this.jumpState = this.JUMP_STATE_DESCENDING;
            this.gravity = -3; // 设置重力加速度为负值，表示向上
          }
        } else if (this.jumpState === this.JUMP_STATE_DESCENDING) {
          this.jumpHeight += this.gravity
          if (this.jumpHeight <= 0) {
            this.jumpHeight = 0;
            this.isJumping = false;
            this.jumpState = this.JUMP_STATE_NONE;
            this.gravityIndex = 1.5
            this.gravity = 5; // 重置重力加速度为正值，表示向下
          }
        }
      } else {
        // 如果不在跳跃状态，重置跳跃状态
        this.jumpState = this.JUMP_STATE_NONE;
      }
    },
    updateAnimations() {
      // 绘制地面和小恐龙
      if(!this.isDied){
        this.drawPic('ground', this.groundX, this.canvasHeight - this.groundH - 10)
        this.drawPic('start', 10, this.groundY)  // 根据跳跃状态调整y坐标
        // 开始切换左右脚
        this.legTimer = setInterval(() => {
          this.currentLeg = this.currentLeg === 'left' ? 'right' : 'left'
        }, this.pace)
        this.swingTimer = setInterval(() => {
          this.currentSwing = this.currentSwing === 'bird_up' ? 'bird_down' : 'bird_up'
        }, this.swing)
      }
    },
    // 设置障碍物
    setBarrier() {
      const minGap = 300 // 障碍物的间隔
      const maxGap = 800 // 障碍物的间隔
      const x = this.lastBarrierX + Math.random() * (maxGap - minGap) + minGap // 随机生成 x 坐标
      const bar = this.barriers_list[Math.floor(Math.random() * this.barriers_list.length)]
      const n = bar.name
      const w = bar.width
      const h = bar.height
      const y = this.treeY - h
      this.barriers.push({ n, x, y, w, h }) // 将障碍物添加到数组中
      this.lastBarrierX = x
    },
    // 绘制障碍物
    drawBarriers() {
      this.barriers.forEach(barrier => {
        this.drawPic(barrier.n, barrier.x, barrier.y, barrier.w, barrier.h)
        barrier.x -= this.speed // 更新障碍物位置
      })
      // 移除超出画布的障碍物
      this.barriers = this.barriers.filter(barrier => barrier.x + 50 > 0)
    },
    // 设置家雀
    setBird() {
      const minGap = 300 // 家雀的间隔
      const maxGap = 800 // 家雀的间隔
      const x = this.lastBarrierX + Math.random() * (maxGap - minGap) + minGap // 随机生成 x 坐标
      const n = 'bird_up'
      const w = this.loadedImages['bird_up'].width
      const h = this.loadedImages['bird_up'].height
      const y = 50
      this.birds.push({ n, x, y, w, h }) // 将家雀添加到数组中
      this.lastBarrierX = x
    },
    // 绘制家雀
    drawBirds() {
      this.birds.forEach(bird => {
        this.drawPic(this.currentSwing, bird.x, bird.y, bird.w, bird.h)
        bird.x -= this.speed // 更新家雀位置
      })
      // 移除超出画布的家雀
      this.birds = this.birds.filter(bird => bird.x + 50 > 0)
    },
    // 设置云
    setCloud() {
      const minGap = 100 
      const maxGap = 500 
      const x = this.lastCloudX + Math.random() * (maxGap - minGap) + minGap // 随机生成 x 坐标
      const height = Math.random() * 70 + 10
      this.clouds.push({ x, height })
      this.lastCloudX = x
    },
    // 绘制云
    drawClouds() {
      this.clouds.forEach(cloud => {
        this.drawPic('cloud', cloud.x, cloud.height)
        cloud.x -= this.speed * 3 / 2
      })
      // 移除超出画布的障碍物
      this.clouds = this.clouds.filter(cloud => cloud.x + 50 > 0)
    },
    // 更新分数
    updateScore() {
      if(!this.isDied){
        this.score += 1
        this.scores[0] = Math.floor(this.score % 10) // 个位
        this.scores[1] = Math.floor((this.score % 100) / 10) // 十位
        this.scores[2] = Math.floor((this.score % 1000) / 100) // 百位
        this.scores[3] = Math.floor((this.score % 10000) / 1000) // 千位
        this.scores[4] = Math.floor((this.score % 100000) / 10000) // 万位
        this.scores[5] = Math.floor(this.score / 100000) // 十万位 
      }
    },
    // 开始跑步，地面移动 + 小恐龙切换左右脚
    play() {
      this.ctx.clearRect(0, 0, this.canvasWidth, this.canvasHeight)
      
      this.drawBarriers()             // 绘制障碍物
      this.drawClouds()               // 绘制云
      this.drawBirds()                // 绘制家雀
      this.drawPic('ground', this.groundX, this.canvasHeight - this.groundH - 10)         // 绘制地面
      this.drawPic('ground', this.groundX + this.canvasWidth, this.canvasHeight - this.groundH - 10)    // 绘制拼接地面
      this.updateGroundPosition()     // 地面移动
      this.accelerate()               // 设置加速
      this.updateJump()               // 更新跳跃
      this.checkCrash()               // 检测碰撞
      this.stage += 0.01              // 控制家雀在某一阶段才开始出现

      const dinoY = this.isJumping ? (this.groundY - this.jumpHeight) : this.groundY
      this.dinoY = dinoY
      const dinoP = this.isDied ? 'die' : this.isJumping ? 'start' : this.currentLeg
      this.drawPic(dinoP, this.dinoX, dinoY)  // 根据状态调整恐龙形态
      
      this.deltaTime = Date.now() - this.lastUpdateTime
      this.lastUpdateTime = Date.now()


      // 每隔一定时间生成家雀
      if (this.stage >= 1 && this.birds.length === 0 || Math.random() < 0.01) { // 设定生成概率
        this.setBird()
      }
      // 每隔一定时间生成障碍物
      if (this.barriers.length === 0 || Math.random() < 0.02) { // 设定生成概率
        this.setBarrier()
      }
      // 每隔一定时间生成云
      if (this.clouds.length === 0 || Math.random() < 2) { // 设定生成概率
        this.setCloud()
      }

      
      for(let i = this.scores.length ; i > 0 ; i--){
        this.drawPic(`${this.scores[i]}`,1000 + (5 - i) * 15,20)
      }
      
      if(this.isDied){
        this.drawPic('over', 470, 65)
        this.drawPic('restart',545, 90)
        return
      }

      // 请求下一帧动画
      requestAnimationFrame(this.play.bind(this))
    },
    // 检测碰撞
    checkCrash() {
      if(!this.isDied){
        this.barriers.forEach(barrier => {
          // 恐龙的右边界超过障碍物的左边界 且 下边界未超过障碍物的上边界
          if (this.dinoX + this.dinoW -5>= barrier.x && 
              this.dinoY + this.dinoH -20>= barrier.y ) {
            this.gameOver()
          }
          // else this.score += 15
        })
        this.birds.forEach(bird => {
          // 恐龙的右边界超过障碍物的左边界 且 下边界未超过障碍物的上边界
          if (this.dinoX + this.dinoW >= bird.x && 
              this.dinoY <= bird.y + bird.h) {
            this.gameOver()
          }
          // else this.score += 15
        })
      }
    },
    gameOver() {
      // 发生碰撞，执行相应的操作
      console.log("碰撞发生！")
      this.$refs.defeatSound.play()
      this.isDied = true
      this.isRunning = false
      this.speed = 0
    },
    resetGame() {
      window.location.reload()
    }
  },
  mounted() {
    this.ctx = this.$refs.canvas.getContext("2d")
    // 预加载图片资源
    const imagePromises = Object.entries(this.images).map(([key, imageSrc]) => {
      return new Promise((resolve, reject) => {
        const img = new Image()
        img.src = require(`/src/assets/images/${imageSrc}.png`)

        img.onload = () => {
          this.loadedImages[key] = img // 存储已加载的图片
          resolve()
        }

        img.onerror = () => {
          console.error(`加载图片失败: ${img.src}`)
          reject()
        }
      })
    })
    
    // 确保所有图片资源加载完成后再启动动画
    Promise.all(imagePromises)
    .then(() => {
        this.updateAnimations()       // 初始化
        this.keydownEvent = window.addEventListener('keydown', this.handleKeyDown)  // 监听键盘事件
        // this.$refs.canvas.addEventListener('click', this.resetGame)
        this.clickEvent = this.$refs.canvas.addEventListener('click', this.clickHandle)
      })
      .catch(error => {
        console.error("图片加载失败：", error)
      })

  },
  unmounted() {
    if (this.legTimer) {
      clearInterval(this.legTimer)
    }
    if (this.swingTimer) {
      clearInterval(this.swingTimer)
    }
    if (this.scoreTimer) {
      clearInterval(this.scoreTimer)
    }
    this.$refs.canvas.removeEventListener(this.clickEvent)
    window.removeEventListener(this.keydownEvent)
  }

}
</script>

<style scoped>
  #canvas {
    background-color: #fff;
  }
</style>
