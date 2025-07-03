<template>
  <div>
    <canvas ref="canvas" width="700" height="700"></canvas>
    <div>
      Хищников: {{ predatorsCount }}, Жертв: {{ preysCount }}
      <br>Съедено: {{ eatenCount }}, Родилось: {{ bornCount }}
    </div>
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
    this.energy = 100
  }

  move() {
    this.x += this.dx * this.speed
    this.y += this.dy * this.speed
    
    if (this.x < 0 || this.x > 700) this.dx *= -1
    if (this.y < 0 || this.y > 700) this.dy *= -1
    
    this.energy -= 0.1
    return this.energy > 0
  }

  distanceTo(other) {
    return Math.sqrt((this.x - other.x) ** 2 + (this.y - other.y) ** 2)
  }
}

class Predator extends Creature {
  constructor(x, y) {
    super(x, y, 20, 1.5, 'red', 'predator')
    this.eaten = 0
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
      animationId: null,
      eatenCount: 0,
      bornCount: 0
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
    
    for (let i = 0; i < 10; i++) {
      this.creatures.push(new Prey(
        Math.random() * 700,
        Math.random() * 700
      ))
    }
    for (let i = 0; i < 3; i++) {
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
        this.updateCreatures()
        this.checkCollisions()
        this.draw()
        this.animationId = requestAnimationFrame(animate)
      }
      animate()
    },
    
    updateCreatures() {
      const newCreatures = []
      const deadCreatures = []
      
      this.creatures.forEach((creature, index) => {
        if (!creature.move()) {
          deadCreatures.push(index)
        }
        
        // Размножение при достаточной энергии
        if (creature.energy > 150) {
          creature.energy = 50
          newCreatures.push(this.reproduce(creature))
          this.bornCount++
        }
      })
      
      // Удаляем умерших
      deadCreatures.reverse().forEach(index => {
        this.creatures.splice(index, 1)
      })
      
      // Добавляем новых
      this.creatures.push(...newCreatures)
    },
    
    checkCollisions() {
      for (let i = 0; i < this.creatures.length; i++) {
        for (let j = i + 1; j < this.creatures.length; j++) {
          const a = this.creatures[i]
          const b = this.creatures[j]
          
          if (a.distanceTo(b) < a.size + b.size) {
            if (a.type === 'predator' && b.type === 'prey') {
              this.eat(a, b)
            } else if (b.type === 'predator' && a.type === 'prey') {
              this.eat(b, a)
            }
          }
        }
      }
    },
    
    eat(predator, prey) {
      predator.energy += 50
      predator.eaten++
      this.eatenCount++
      
      const index = this.creatures.indexOf(prey)
      if (index !== -1) {
        this.creatures.splice(index, 1)
      }
    },
    
    reproduce(parent) {
      return new parent.constructor(
        parent.x + (Math.random() * 40 - 20),
        parent.y + (Math.random() * 40 - 20)
      )
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