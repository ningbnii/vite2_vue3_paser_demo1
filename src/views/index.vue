<template>
  <div class="score" @touchmove.prevent>Score:{{ score }}</div>
  <div class="control" @touchmove.prevent>
    <van-button
      type="primary"
      @touchstart="left"
      @touchend="handleTouchEnd('left')"
      >向左</van-button
    >
    <van-button
      type="primary"
      @touchstart="jump"
      @touchend="handleTouchEnd('jump')"
      >跳</van-button
    >
    <van-button
      type="primary"
      @touchstart="right"
      @touchend="handleTouchEnd('right')"
      >向右</van-button
    >
  </div>
  <div class="canvas_box" ref="canvasBox">
    <div ref="myCanvas"></div>
  </div>
</template>
<script setup>
import { ref, onMounted } from "vue";
import * as Phaser from "Phaser";
import { Toast } from "vant";
import sky from "../assets/sky.png";
import ground from "../assets/platform.png";
import star from "../assets/star.png";
import bomb from "../assets/bomb.png";
import dude from "../assets/dude.png";

let myCanvas = ref(null);
let canvasBox = ref(null);
let platforms;
let player;
let cursors;
let stars;
let controlStatus = [];
let score = ref(0);
let bombs;
let game;
let gameOverStatus = false;

onMounted(() => {
  let config = {
    type: Phaser.AUTO,
    width: canvasBox.value.clientWidth,
    height: canvasBox.value.clientHeight,
    parent: myCanvas.value,
    physics: {
      default: "arcade",
      arcade: {
        gravity: { y: 300 },
        debug: false,
      },
    },
    scene: {
      preload: preload,
      create: create,
      update: update,
    },
  };
  game = new Phaser.Game(config);

  /**
   * 加载资源
   */
  function preload() {
    this.load.image("sky", sky);
    this.load.image("ground", ground);
    this.load.image("star", star);
    this.load.image("bomb", bomb);
    this.load.spritesheet("dude", dude, {
      frameWidth: 32,
      frameHeight: 48,
    });
  }

  /**
   * 创建游戏对象
   */
  function create() {
    // setOrigin设置中心点为00
    this.add.image(0, 0, "sky").setOrigin(0, 0);
    // 生成一个静态物理组，在arcade物理系统中，有动态的和静态的两类物体，动态物体可以通过外力，比如
    // 速度，加速度，得以四处移动，它可以跟其他对象发生反弹，碰撞，此类碰撞受物体质量和其他因素影响，
    // 静态物体只有位置和尺寸，重力对它没有影响，你不能给它设置速度，有东西跟它碰撞时，它一点不动，所以，用作地面和平台很完美，
    // 我们打算让玩家在上面来回跑动
    platforms = this.physics.add.staticGroup();
    platforms.create(-100, 250, "ground").setOrigin(0, 0).refreshBody();
    platforms.create(300, 400, "ground").setOrigin(0, 0).refreshBody();
    // setScale缩放，放大2倍
    platforms
      .create(0, 568, "ground")
      .setOrigin(0, 0)
      .setScale(2)
      .refreshBody();

    player = this.physics.add.sprite(100, 450, "dude");
    player.setBounce(0.2);

    // 画面边界碰撞检测
    player.setCollideWorldBounds(true);
    // player.body.setGravityY(300)

    // 向左移动动画，使用0,1,2,3帧，跑动时每秒10帧，repeat:-1，动画要循环播放
    this.anims.create({
      key: "left",
      frames: this.anims.generateFrameNumbers("dude", { start: 0, end: 3 }),
      frameRate: 10,
      repeat: -1,
    });

    this.anims.create({
      key: "right",
      frames: this.anims.generateFrameNumbers("dude", { start: 5, end: 8 }),
      frameRate: 10,
      repeat: -1,
    });

    this.anims.create({
      key: "turn",
      frames: [{ key: "dude", frame: 4 }],
      frameRate: 10,
    });

    // 监听键盘
    cursors = this.input.keyboard.createCursorKeys();
    // player和platforms添加碰撞检测
    this.physics.add.collider(player, platforms);

    // 添加星星
    stars = this.physics.add.group({
      key: "star",
      repeat: 5,
      setXY: {
        x: 6,
        y: 0,
        stepX: 70,
      },
    });

    stars.children.iterate(function (child) {
      child.setBounceY(Phaser.Math.FloatBetween(0.4, 0.8));
    });

    // 碰撞器，检测星星和platforms
    this.physics.add.collider(stars, platforms);
    // 检测玩家是否与星星重叠
    this.physics.add.overlap(player, stars, collectStar, null, this);

    // 添加敌人
    bombs = this.physics.add.group();
    // 碰撞器
    this.physics.add.collider(bombs, platforms);
    this.physics.add.collider(player, bombs, hitBomb, null, this);
  }

  /**
   * 帧刷新
   */
  function update() {
    if (gameOverStatus) {
      return;
    }
    if (cursors.left.isDown || controlStatus.includes("left")) {
      player.setVelocityX(-160);
      player.anims.play("left", true);
    } else if (cursors.right.isDown || controlStatus.includes("right")) {
      player.setVelocityX(160);
      player.anims.play("right", true);
    } else {
      player.setVelocityX(0);
      player.anims.play("turn");
    }
    if (
      (cursors.up.isDown || controlStatus.includes("jump")) &&
      player.body.touching.down
    ) {
      player.setVelocityY(-360);
    }
  }

  /**
   * 收集星星
   * @param player
   * @param star
   */
  function collectStar(player, star) {
    star.disableBody(true, true);
    score.value += 10;
    if (score.value == 100) {
      Toast.success("恭喜您通关了");
      gameOverStatus = true;
      this.physics.pause();
      player.anims.play("turn");
      return;
    }

    // 看看还有多少星星活着

    if (stars.countActive(true) === 0) {
      stars.children.iterate(function (child) {
        // 重新激活所有星星，重置它们的位置为0，这将使所有星星再次从画面顶部落下来
        child.enableBody(true, child.x, 0, true, true);
      });
      let x = Phaser.Math.Between(0, this.game.config.width);
      let bomb = bombs.create(x, 16, "bomb");
      bomb.setBounce(1);
      bomb.setCollideWorldBounds(true);
      bomb.setVelocityY(Phaser.Math.Between(-200, 200));
    }
  }

  function hitBomb(player, bomb) {
    // 游戏结束
    gameOverStatus = true;
    this.physics.pause();
    player.setTint(0xff00000);
    player.anims.play("turn");
  }
});

/**
 * 移动端控制
 */
function left() {
  controlStatus.push("left");
}

function right() {
  controlStatus.push("right");
}

function jump() {
  controlStatus.push("jump");
}

function handleTouchEnd(controlName) {
  let key = controlStatus.indexOf(controlName);
  if (key > -1) {
    controlStatus.splice(key, 1);
  }
}
</script>
<style lang="less" scoped>
.control {
  width: 100%;
  position: absolute;
  bottom: 0;
  display: flex;
  justify-content: space-around;
}
.score {
  position: absolute;
  top: 10px;
  left: 10px;
}
.canvas_box {
  width: 100vw;
  height: 100vh;
}
</style>
