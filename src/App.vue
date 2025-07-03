<template>
  <div class="simulation-container">
    <div class="simulation-wrapper">
      <canvas ref="canvas" width="700" height="700"></canvas>
      <div class="statistics-panel">
        <h2>Статистика</h2>
        <div class="stats">
          <p>Умерли от голода: {{ stats.energyDeaths }}</p>
          <p>Съедено хищниками: {{ stats.predationDeaths }}</p>
          <p>Новых существ: {{ stats.newCreatures }}</p>
          <p>Хищников: {{ predatorCount }}</p>
          <p>Мирных: {{ preyCount }}</p>
          <p>Еды: {{ foods.length }}</p>
          
          <h3>Характеристики</h3>
          <p>Макс. размер: {{ stats.maxSize.toFixed(2) }}</p>
          <p>Макс. скорость: {{ stats.maxSpeed.toFixed(2) }}</p>
          <p>Макс. зрение: {{ stats.maxVision.toFixed(2) }}</p>
        </div>

        <div class="controls">
          <h3>Управление</h3>
          <button @click="togglePause">
            {{ isPaused ? 'Продолжить' : 'Пауза' }}
          </button>
          
          <div class="spawn-controls">
            <button @click="spawnRandomPredator">Добавить хищника</button>
            <button @click="spawnRandomPrey">Добавить мирного</button>
            <button @click="spawnFood(10)">Добавить еду</button>
          </div>
          
          <div class="settings">
            <div>
              <label>Интервал еды (сек):</label>
              <input type="number" v-model.number="foodSpawnInterval" min="1">
            </div>
            <div>
              <label>Количество еды:</label>
              <input type="number" v-model.number="foodSpawnAmount" min="1">
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
// Базовый класс существа
class Creature {
  constructor(x, y, size, speed, vision, type) {
    this.x = x
    this.y = y
    this.size = size
    this.speed = speed
    this.vision = vision
    this.type = type
    this.energy = 100
    this.dx = Math.random() * 2 - 1
    this.dy = Math.random() * 2 - 1
    this.color = this.getRandomColor()
    this.foodEaten = 0
  }

  getRandomColor() {
    return `hsl(${Math.random() * 360}, 70%, 50%)`
  }

  // Добавлен метод update()
  update(deltaTime) {
    this.energy -= 0.04 * deltaTime
    return this.energy <= 0 // Возвращает true, если существо умерло
  }

  distanceTo(target) {
    return Math.sqrt(Math.pow(this.x - target.x, 2) + Math.pow(this.y - target.y, 2))
  }

  moveTowards(target, deltaTime) {
    const dx = target.x - this.x
    const dy = target.y - this.y
    const dist = Math.sqrt(dx * dx + dy * dy)
    
    if (dist > 1) {
      this.dx = dx / dist
      this.dy = dy / dist
    }
    
    this.move(deltaTime)
  }

  moveAwayFrom(threat, deltaTime) {
    const dx = this.x - threat.x
    const dy = this.y - threat.y
    const dist = Math.sqrt(dx * dx + dy * dy)
    
    if (dist > 1) {
      this.dx = dx / dist
      this.dy = dy / dist
    }
    
    this.move(deltaTime)
  }

  moveRandomly(deltaTime) {
    if (this.x <= 50 || this.x >= 650 - this.size ||
        this.y <= 50 || this.y >= 650 - this.size) {
      this.dx = Math.random() * 2 - 1
      this.dy = Math.random() * 2 - 1
    }
    
    if (Math.random() < 0.01) {
      this.dx = Math.random() * 2 - 1
      this.dy = Math.random() * 2 - 1
    }
    
    this.move(deltaTime)
  }

  move(deltaTime) {
    this.x += this.dx * this.speed * 30 * deltaTime
    this.y += this.dy * this.speed * 30 * deltaTime
    
    // Ограничение в пределах поля
    this.x = Math.max(50, Math.min(650 - this.size, this.x))
    this.y = Math.max(50, Math.min(650 - this.size, this.y))
  }
}

// Класс хищника
class Predator extends Creature {
  constructor(x, y, size, speed, vision) {
    super(x, y, size, speed, vision, 'predator')
    this.color = `hsl(${Math.random() * 60}, 70%, 50%)` // Красные оттенки
  }

  findTarget(creatures) {
    let nearest = null
    let minDist = Infinity
    
    creatures.forEach(other => {
      if (other === this || other.type !== 'prey') return
      
      const dist = this.distanceTo(other)
      if (dist < this.vision && dist < minDist) {
        minDist = dist
        nearest = other
      }
    })
    
    return nearest
  }
}

// Класс мирного существа
class Prey extends Creature {
  constructor(x, y, size, speed, vision) {
    super(x, y, size, speed, vision, 'prey')
    this.color = `hsl(${120 + Math.random() * 90}, 70%, 50%)` // Зеленые оттенки
  }

  findTarget(foods) {
    let nearest = null
    let minDist = Infinity
    
    foods.forEach(food => {
      const dist = this.distanceTo(food)
      if (dist < this.vision && dist < minDist) {
        minDist = dist
        nearest = food
      }
    })
    
    return nearest
  }
}

// Класс еды
class Food {
  constructor(x, y) {
    this.x = x
    this.y = y
    this.energy = 30
    this.color = '#2ecc71'
    this.size = 5
  }
}

export default {
  data() {
    return {
      canvas: null,
      ctx: null,
      creatures: [],
      foods: [],
      isPaused: false,
      lastTime: 0,
      animationFrameId: null,
      foodSpawnTimer: null,
      foodSpawnInterval: 15,
      foodSpawnAmount: 10,
      stats: {
        energyDeaths: 0,
        predationDeaths: 0,
        newCreatures: 0,
        maxSize: 0,
        maxSpeed: 0,
        maxVision: 0
      }
    }
  },
  computed: {
    predatorCount() {
      return this.creatures.filter(c => c.type === 'predator').length
    },
    preyCount() {
      return this.creatures.filter(c => c.type === 'prey').length
    }
  },
  mounted() {
    this.canvas = this.$refs.canvas
    this.ctx = this.canvas.getContext('2d')
    this.initSimulation()
    this.startAnimation()
    this.startFoodSpawner()
  },
  beforeUnmount() {
    this.stopSimulation()
  },
  methods: {
    initSimulation() {
      // Создаем начальных существ
      for (let i = 0; i < 5; i++) this.spawnRandomPrey()
      for (let i = 0; i < 2; i++) this.spawnRandomPredator()
      this.spawnFood(20)
      this.updateStats()
    },
    
    spawnRandomPredator() {
      const x = 50 + Math.random() * 600
      const y = 50 + Math.random() * 600
      this.creatures.push(new Predator(
        x, y,
        20 + Math.random() * 10,
        1 + Math.random(),
        100 + Math.random() * 50
      ))
    },
    
    spawnRandomPrey() {
      const x = 50 + Math.random() * 600
      const y = 50 + Math.random() * 600
      this.creatures.push(new Prey(
        x, y,
        15 + Math.random() * 10,
        0.5 + Math.random(),
        80 + Math.random() * 40
      ))
    },
    
    spawnFood(amount) {
      for (let i = 0; i < amount; i++) {
        this.foods.push(new Food(
          50 + Math.random() * 600,
          50 + Math.random() * 600
        ))
      }
    },
    
    startFoodSpawner() {
      if (this.foodSpawnTimer) clearInterval(this.foodSpawnTimer)
      this.foodSpawnTimer = setInterval(() => {
        if (!this.isPaused) this.spawnFood(this.foodSpawnAmount)
      }, this.foodSpawnInterval * 1000)
    },
    
    startAnimation() {
      this.lastTime = performance.now()
      this.animationFrameId = requestAnimationFrame(this.update)
    },
    
    stopSimulation() {
      cancelAnimationFrame(this.animationFrameId)
      clearInterval(this.foodSpawnTimer)
    },
    
    togglePause() {
      this.isPaused = !this.isPaused
      if (!this.isPaused) this.startAnimation()
    },
    
    update(currentTime) {
      if (this.isPaused) return
      
      const deltaTime = (currentTime - this.lastTime) / 1000
      this.lastTime = currentTime
      
      this.updateCreatures(deltaTime)
      this.checkCollisions()
      this.draw()
      
      this.animationFrameId = requestAnimationFrame(this.update)
    },
    
    updateCreatures(deltaTime) {
      const creaturesToRemove = []
      const newCreatures = []
      
      this.creatures.forEach((creature, index) => {
        // Потеря энергии
        const isDead = creature.update(deltaTime)
        
        if (isDead) {
          creaturesToRemove.push(index)
          this.stats.energyDeaths++
          this.foods.push(new Food(creature.x, creature.y))
          return
        }
        
        // Разное поведение для хищников и мирных
        if (creature.type === 'predator') {
          const target = creature.findTarget(this.creatures)
          if (target) {
            creature.moveTowards(target, deltaTime)
          } else {
            creature.moveRandomly(deltaTime)
          }
        } else { // Мирные существа
          const target = creature.findTarget(this.foods)
          if (target) {
            creature.moveTowards(target, deltaTime)
          } else {
            creature.moveRandomly(deltaTime)
          }
        }
      })
      
      // Удаление умерших существ
      creaturesToRemove.sort((a, b) => b - a).forEach(index => {
        this.creatures.splice(index, 1)
      })
      
      // Добавление новых существ
      this.creatures.push(...newCreatures)
      
      this.updateStats()
    },
    
    checkCollisions() {
      for (let i = 0; i < this.creatures.length; i++) {
        for (let j = 0; j < this.foods.length; j++) {
          const creature = this.creatures[i]
          const food = this.foods[j]
          
          if (creature.type === 'prey' && this.distance(creature, food) < creature.size / 2) {
            creature.energy += food.energy
            creature.foodEaten++
            this.foods.splice(j, 1)
            j--
            
            if (creature.foodEaten >= 4) {
              creature.foodEaten = 0
              this.creatures.push(this.reproduceCreature(creature))
              this.stats.newCreatures++
            }
          }
        }
        
        for (let j = i + 1; j < this.creatures.length; j++) {
          const c1 = this.creatures[i]
          const c2 = this.creatures[j]
          
          if (this.distance(c1, c2) < (c1.size + c2.size) / 2) {
            this.resolveCollision(c1, c2)
          }
        }
      }
    },
    
    resolveCollision(c1, c2) {
      // Хищник атакует мирное существо
      if (c1.type === 'predator' && c2.type === 'prey') {
        this.predatorEatsPrey(c1, c2)
      } 
      else if (c2.type === 'predator' && c1.type === 'prey') {
        this.predatorEatsPrey(c2, c1)
      }
      // Отталкивание одинаковых существ
      else {
        const dx = c2.x - c1.x
        const dy = c2.y - c1.y
        const dist = Math.sqrt(dx * dx + dy * dy)
        
        if (dist > 0) {
          const pushForce = 0.5
          c1.x -= dx / dist * pushForce
          c1.y -= dy / dist * pushForce
          c2.x += dx / dist * pushForce
          c2.y += dy / dist * pushForce
        }
      }
    },
    
    predatorEatsPrey(predator, prey) {
      predator.energy += prey.energy / 2
      predator.foodEaten++
      
      const index = this.creatures.indexOf(prey)
      if (index !== -1) {
        this.creatures.splice(index, 1)
        this.stats.predationDeaths++
      }
    },
    
    reproduceCreature(parent) {
      const size = parent.size * (0.8 + Math.random() * 0.4)
      const speed = parent.speed * (0.8 + Math.random() * 0.4)
      const vision = parent.vision * (0.8 + Math.random() * 0.4)
      
      if (parent.type === 'predator') {
        return new Predator(
          parent.x + (Math.random() * 40 - 20),
          parent.y + (Math.random() * 40 - 20),
          size, speed, vision
        )
      } else {
        return new Prey(
          parent.x + (Math.random() * 40 - 20),
          parent.y + (Math.random() * 40 - 20),
          size, speed, vision
        )
      }
    },
    
    distance(a, b) {
      return Math.sqrt(Math.pow(a.x - b.x, 2) + Math.pow(a.y - b.y, 2))
    },
    
    updateStats() {
      if (this.creatures.length === 0) {
        this.stats.maxSize = 0
        this.stats.maxSpeed = 0
        this.stats.maxVision = 0
        return
      }
      
      this.stats.maxSize = Math.max(...this.creatures.map(c => c.size))
      this.stats.maxSpeed = Math.max(...this.creatures.map(c => c.speed))
      this.stats.maxVision = Math.max(...this.creatures.map(c => c.vision))
    },
    
    draw() {
      if (!this.ctx) return
      
      this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height)
      
      // Рисуем границы
      this.ctx.strokeStyle = '#ffffff'
      this.ctx.lineWidth = 2
      this.ctx.strokeRect(50, 50, 600, 600)
      
      // Рисуем еду
      this.foods.forEach(food => {
        this.ctx.fillStyle = food.color
        this.ctx.beginPath()
        this.ctx.arc(food.x, food.y, food.size, 0, Math.PI * 2)
        this.ctx.fill()
      })
      
      // Рисуем существ
      this.creatures.forEach(creature => {
        // Тело
        this.ctx.fillStyle = creature.color
        this.ctx.beginPath()
        this.ctx.arc(creature.x, creature.y, creature.size / 2, 0, Math.PI * 2)
        this.ctx.fill()
        
        // Глаза (разные для хищников и мирных)
        this.ctx.fillStyle = creature.type === 'predator' ? '#ff0000' : '#ffffff'
        this.ctx.beginPath()
        this.ctx.arc(
          creature.x + creature.dx * creature.size * 0.3,
          creature.y + creature.dy * creature.size * 0.3,
          creature.size * 0.15,
          0,
          Math.PI * 2
        )
        this.ctx.fill()
      })
    }
  }
}
</script>

<style>
.simulation-container {
  display: flex;
  justify-content: center;
  padding: 20px;
}

.simulation-wrapper {
  display: flex;
  gap: 20px;
}

canvas {
  background-color: #000000;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.statistics-panel {
  width: 300px;
  background-color: #fff;
  border-radius: 8px;
  padding: 20px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.stats {
  margin-bottom: 20px;
}

.controls {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

button {
  padding: 10px 15px;
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
  transition: background-color 0.2s;
}

button:hover {
  background-color: #2980b9;
}

.spawn-controls {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.settings {
  display: flex;
  flex-direction: column;
  gap: 10px;
  padding: 15px;
  background-color: #f8f9fa;
  border-radius: 4px;
}

.settings div {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

input {
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  width: 60px;
}

h2, h3 {
  color: #2c3e50;
  margin-top: 0;
}

h3 {
  margin-bottom: 10px;
}
</style>