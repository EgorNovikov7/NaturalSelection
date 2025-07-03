<template>
  <div>
    <canvas ref="canvas" width="700" height="700"></canvas>
  </div>
</template>

<script>
class Creature {
  constructor(x, y, size, color) {
    this.x = x
    this.y = y
    this.size = size
    this.color = color
    this.dx = Math.random() * 2 - 1
    this.dy = Math.random() * 2 - 1
  }

  move() {
    this.x += this.dx * 2
    this.y += this.dy * 2
    
    // Отскок от границ
    if (this.x < 0 || this.x > 700) this.dx *= -1
    if (this.y < 0 || this.y > 700) this.dy *= -1
  }
}

export default {
  data() {
    return {
      ctx: null,
      creatures: [],
      animationId: null
    }
  },
  mounted() {
    const canvas = this.$refs.canvas
    this.ctx = canvas.getContext('2d')
    
    for (let i = 0; i < 10; i++) {
      this.creatures.push(new Creature(
        Math.random() * 700,
        Math.random() * 700,
        15,
        `hsl(${Math.random() * 360}, 70%, 50%)`
      ))
    }
    
    this.startAnimation()
  },
  methods: {
    startAnimation() {
      const animate = () => {
        this.creatures.forEach(c => c.move())
        this.draw()
        this.animationId = requestAnimationFrame(animate)
      }
      animate()
    },
    draw() {
      this.ctx.clearRect(0, 0, 700, 700)
      this.creatures.forEach(c => {
        this.ctx.fillStyle = c.color
        this.ctx.beginPath()
        this.ctx.arc(c.x, c.y, c.size, 0, Math.PI * 2)
        this.ctx.fill()
      })
    }
  },
  beforeUnmount() {
    cancelAnimationFrame(this.animationId)
  }
}
</script>