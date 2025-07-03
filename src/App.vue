<template>
  <div class="simulation-container">
    <div class="simulation-wrapper">
      <canvas ref="canvas" width="700" height="700"></canvas>
      <div class="statistics-panel">
        <h2>Статистика</h2>
        <div class="stats">
          <p>Умерли от голода: {{ stats.energyDeaths }}</p>
          <p>Съедено существ: {{ stats.predationDeaths }}</p>
          <p>Новых существ: {{ stats.newCreatures }}</p>
          <p>Всего существ: {{ creatures.length }}</p>
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
            <button @click="spawnRandomCreature">Добавить существо</button>
            <button @click="spawnFood(10)">Добавить еду</button>
          </div>
          
          <div class="settings">
            <div>
              <label>Интервал еды (сек):</label>
              <input type="number" v-model.number="foodSpawnInterval" min="1" @change="restartFoodSpawner">
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
    this.isAggressive = false
  }

  getRandomColor() {
    return `hsl(${Math.random() * 360}, 70%, 50%)`
  }

  update(deltaTime) {
    this.energy -= 1.5 * deltaTime * (this.size / 20)
    return this.energy <= 0
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
    
    this.x = Math.max(50, Math.min(650 - this.size, this.x))
    this.y = Math.max(50, Math.min(650 - this.size, this.y))
  }
}

class Food {
  constructor(x, y) {
    this.x = x
    this.y = y
    this.energy = 45
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
      for (let i = 0; i < 15; i++) this.spawnRandomCreature()
      this.spawnFood(15)
      this.updateStats()
    },
    
    spawnRandomCreature() {
      const x = 50 + Math.random() * 600
      const y = 50 + Math.random() * 600
      const size = 10 + Math.random() * 30
      const speed = 0.5 + Math.random() * 1.5
      const vision = 50 + Math.random() * 150
      
      this.creatures.push(new Creature(x, y, size, speed, vision, 'creature'))
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
    
    restartFoodSpawner() {
      this.startFoodSpawner()
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
      
      this.creatures.forEach((creature, index) => {
        const isDead = creature.update(deltaTime)
        
        if (isDead) {
          creaturesToRemove.push(index)
          this.stats.energyDeaths++
          this.foods.push(new Food(creature.x, creature.y))
          return
        }
        
        // Находим ближайшую еду или существо меньшего размера
        let target = null
        let minDist = Infinity
        
        // Сначала проверяем другие существа
        this.creatures.forEach(other => {
          if (other === creature || other.size >= creature.size) return
          
          const dist = creature.distanceTo(other)
          if (dist < creature.vision && dist < minDist) {
            minDist = dist
            target = other
            creature.isAggressive = true
          }
        })
        
        // Если не нашли подходящее существо, ищем еду
        if (!target) {
          creature.isAggressive = false
          this.foods.forEach(food => {
            const dist = creature.distanceTo(food)
            if (dist < creature.vision && dist < minDist) {
              minDist = dist
              target = food
            }
          })
        }
        
        // Двигаемся к цели или случайным образом
        if (target) {
          creature.moveTowards(target, deltaTime)
        } else {
          creature.moveRandomly(deltaTime)
        }
      })
      
      // Удаляем умерших существ
      creaturesToRemove.sort((a, b) => b - a).forEach(index => {
        this.creatures.splice(index, 1)
      })
      
      this.updateStats()
    },
    
    checkCollisions() {
      // Проверка столкновений с едой
      for (let i = 0; i < this.creatures.length; i++) {
        const creature = this.creatures[i]
        
        for (let j = 0; j < this.foods.length; j++) {
          const food = this.foods[j]
          
          if (this.distance(creature, food) < creature.size / 2) {
            creature.energy += food.energy
            creature.foodEaten++
            this.foods.splice(j, 1)
            j--
            
            if (creature.foodEaten >= 5) {
              creature.foodEaten = 0
              this.creatures.push(this.reproduceCreature(creature))
              this.stats.newCreatures++
            }
          }
        }
      }
      
      // Проверка столкновений между существами
      for (let i = 0; i < this.creatures.length; i++) {
        for (let j = i + 1; j < this.creatures.length; j++) {
          const c1 = this.creatures[i]
          const c2 = this.creatures[j]
          
          if (this.distance(c1, c2) < (c1.size + c2.size) / 2) {
            if (c1.size > c2.size) {
              this.creatureEatsCreature(c1, c2)
              j-- // Уменьшаем j, так как массив изменился
            } 
            else if (c2.size > c1.size) {
              this.creatureEatsCreature(c2, c1)
              i-- // Уменьшаем i, так как массив изменился
              break // Выходим из внутреннего цикла, так как c1 был удален
            }
            else {
              // Отталкивание при одинаковом размере
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
          }
        }
      }
    },
    
    creatureEatsCreature(predator, prey) {
      predator.energy += prey.energy * 0.8
      predator.foodEaten++
      
      const preyIndex = this.creatures.indexOf(prey)
      if (preyIndex !== -1) {
        this.creatures.splice(preyIndex, 1)
        this.stats.predationDeaths++
      }
      
      if (predator.foodEaten >= 5) {
        predator.foodEaten = 0
        this.creatures.push(this.reproduceCreature(predator))
        this.stats.newCreatures++
      }
    },
    
    reproduceCreature(parent) {
      const size = parent.size * (0.8 + Math.random() * 0.4)
      const speed = parent.speed * (0.8 + Math.random() * 0.4)
      const vision = parent.vision * (0.8 + Math.random() * 0.4)
      
      return new Creature(
        parent.x + (Math.random() * 40 - 20),
        parent.y + (Math.random() * 40 - 20),
        Math.max(5, Math.min(50, size)),
        Math.max(0.2, Math.min(2.5, speed)),
        Math.max(30, Math.min(250, vision)),
        'creature'
      )
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
        this.ctx.fillStyle = creature.color
        this.ctx.beginPath()
        this.ctx.arc(creature.x, creature.y, creature.size / 2, 0, Math.PI * 2)
        this.ctx.fill()
        
        // Глаза
        this.ctx.fillStyle = creature.isAggressive ? '#ff0000' : '#ffffff'
        this.ctx.beginPath()
        this.ctx.arc(
          creature.x + creature.dx * creature.size * 0.3,
          creature.y + creature.dy * creature.size * 0.3,
          creature.size * 0.15,
          0,
          Math.PI * 2
        )
        this.ctx.fill()
        
        // Обводка для крупных существ
        if (creature.size > 20) {
          this.ctx.strokeStyle = creature.isAggressive ? '#ff0000' : '#00ff00'
          this.ctx.lineWidth = 2
          this.ctx.beginPath()
          this.ctx.arc(creature.x, creature.y, creature.size / 2 + 2, 0, Math.PI * 2)
          this.ctx.stroke()
        }
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