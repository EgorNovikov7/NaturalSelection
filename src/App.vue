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
  }
}

export default {
  data() {
    return {
      ctx: null,
      creatures: []
    }
  },
  mounted() {
    const canvas = this.$refs.canvas
    this.ctx = canvas.getContext('2d')
    
    // Создаем тестовых существ
    for (let i = 0; i < 10; i++) {
      this.creatures.push(new Creature(
        Math.random() * 700,
        Math.random() * 700,
        15 + Math.random() * 10,
        `hsl(${Math.random() * 360}, 70%, 50%)`
      ))
    }
    
    this.draw()
  },
  methods: {
    draw() {
      this.ctx.clearRect(0, 0, 700, 700)
      
      this.creatures.forEach(creature => {
        this.ctx.fillStyle = creature.color
        this.ctx.beginPath()
        this.ctx.arc(creature.x, creature.y, creature.size, 0, Math.PI * 2)
        this.ctx.fill()
      })
    }
  }
}
</script>