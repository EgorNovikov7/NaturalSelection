<template>
  <div>
    <canvas ref="canvas" width="700" height="700"></canvas>
    <div>Хищников: {{ predatorsCount }}, Жертв: {{ preysCount }}</div>
  </div>
</template>

<script>
class Creature {
  constructor(x, y, size, speed, color, type) {
    this.x = x
    this.y = y
    this.size = size
    this.speed = speed
    this.color = color
    this.type = type
    this.dx = Math.random() * 2 - 1
    this.dy = Math.random() * 2 - 1
  }

  move() {
    this.x += this.dx * this.speed
    this.y += this.dy * this.speed
    
    if (this.x < 0 || this.x > 700) this.dx *= -1
    if (this.y < 0 || this.y > 700) this.dy *= -1
  }
}

class Predator extends Creature {
  constructor(x, y) {
    super(x, y, 20, 1.5, 'red', 'predator')
  }
}

class Prey extends Creature {
  constructor(x, y) {
    super(x, y, 15, 1, 'green', 'prey')
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
  computed: {
    predatorsCount() {
      return this.creatures.filter(c => c.type === 'predator').length
    },
    preysCount() {
      return this.creatures.filter(c => c.type === 'prey').length
    }
  },
  mounted() {
    const canvas = this.$refs.canvas
    this.ctx = canvas.getContext('2d')
    
    // 5 жертв и 2 хищника
    for (let i = 0; i < 5; i++) {
      this.creatures.push(new Prey(
        Math.random() * 700,
        Math.random() * 700
      ))
    }
    for (let i = 0; i < 2; i++) {
      this.creatures.push(new Predator(
        Math.random() * 700,
        Math.random() * 700
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