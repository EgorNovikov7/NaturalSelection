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
  constructor(x, y, size, speed, vision) {
    this.x = x
    this.y = y
    this.baseSize = size // Базовый размер
    this.size = size // Текущий размер (может увеличиваться)
    this.speed = speed
    this.vision = vision
    this.energy = 100
    this.dx = Math.random() * 2 - 1
    this.dy = Math.random() * 2 - 1
    this.color = `hsl(${Math.random() * 360}, 70%, 50%)`
    this.foodEaten = 0
    this.creaturesEaten = 0 // Количество съеденных существ
    this.isHunting = false
  }

  update(deltaTime) {
    // Расход энергии зависит от размера
    this.energy -= 3 * deltaTime * (this.size / 15)
    return this.energy <= 0
  }

  distanceTo(target) {
    return Math.sqrt(Math.pow(this.x - target.x, 2) + Math.pow(this.y - target.y, 2))
  }

  findTarget(foods, creatures) {
    // Ищем ближайшую еду
    let nearestFood = null
    let minFoodDist = Infinity
    
    foods.forEach(food => {
      // Проверяем, чтобы еда не была слишком близко к стене
      if (food.x <= 50 + this.size/2 || food.x >= 650 - this.size/2 ||
          food.y <= 50 + this.size/2 || food.y >= 650 - this.size/2) {
        return
      }
      
      const dist = this.distanceTo(food)
      if (dist < this.vision && dist < minFoodDist) {
        minFoodDist = dist
        nearestFood = food
      }
    })

    // Ищем существ на 30% меньше себя
    let nearestPrey = null
    let minPreyDist = Infinity
    
    creatures.forEach(other => {
      if (other === this || other.size > this.size * 0.7) return
      
      const dist = this.distanceTo(other)
      if (dist < this.vision && dist < minPreyDist) {
        minPreyDist = dist
        nearestPrey = other
      }
    })

    // Если нашли подходящую цель и она ближе чем еда
    if (nearestPrey && (!nearestFood || minPreyDist < minFoodDist)) {
      this.isHunting = true
      return nearestPrey
    }

    this.isHunting = false
    return nearestFood
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

  moveRandomly(deltaTime) {
    if (this.x <= 50 + this.size/2 || this.x >= 650 - this.size/2 ||
        this.y <= 50 + this.size/2 || this.y >= 650 - this.size/2) {
      // Отталкиваемся от стен
      this.dx = this.x <= 50 + this.size/2 ? 1 : 
               this.x >= 650 - this.size/2 ? -1 : 
               Math.random() * 2 - 1
      this.dy = this.y <= 50 + this.size/2 ? 1 : 
               this.y >= 650 - this.size/2 ? -1 : 
               Math.random() * 2 - 1
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
    
    // Убедимся, что существо не выходит за границы
    this.x = Math.max(50 + this.size/2, Math.min(650 - this.size/2, this.x))
    this.y = Math.max(50 + this.size/2, Math.min(650 - this.size/2, this.y))
  }

  // Увеличение размера после поедания другого существа
  grow() {
    this.creaturesEaten++
    // Размер увеличивается на 10% от базового размера за каждое съеденное существо
    this.size = this.baseSize * (1 + this.creaturesEaten * 0.1)
  }
}

class Food {
  constructor(x, y) {
    // Гарантируем, что еда не появляется слишком близко к стенам
    this.x = Math.max(60, Math.min(640, x))
    this.y = Math.max(60, Math.min(640, y))
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
      for (let i = 0; i < 10; i++) this.spawnRandomCreature(true)
      this.spawnFood(10)
    },
    
    spawnRandomCreature(minimal = false) {
      const x = 50 + Math.random() * 600
      const y = 50 + Math.random() * 600
      
      if (minimal) {
        // Увеличенные начальные параметры
        this.creatures.push(new Creature(
          x, y,
          20 + Math.random() * 10,   // размер 20-30
          0.7 + Math.random() * 0.3, // скорость 0.7-1.0
          70 + Math.random() * 30     // зрение 70-100
        ))
      } else {
        // Обычные параметры (при добавлении вручную)
        this.creatures.push(new Creature(
          x, y,
          20 + Math.random() * 15,
          0.7 + Math.random() * 0.6,
          70 + Math.random() * 50
        ))
      }
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
      this.updateStats()
      this.draw()
      
      this.animationFrameId = requestAnimationFrame(this.update)
    },
    
    updateCreatures(deltaTime) {
      const creaturesToRemove = []
      
      this.creatures.forEach((creature, index) => {
        if (creature.update(deltaTime)) {
          creaturesToRemove.push(index)
          this.stats.energyDeaths++
          this.foods.push(new Food(creature.x, creature.y))
          return
        }
        
        const target = creature.findTarget(this.foods, this.creatures)
        if (target) {
          creature.moveTowards(target, deltaTime)
        } else {
          creature.moveRandomly(deltaTime)
        }
      })
      
      creaturesToRemove.sort((a, b) => b - a).forEach(index => {
        this.creatures.splice(index, 1)
      })
    },
    
    checkCollisions() {
      for (let i = 0; i < this.creatures.length; i++) {
        // Столкновение с едой
        for (let j = 0; j < this.foods.length; j++) {
          const creature = this.creatures[i]
          const food = this.foods[j]
          
          if (this.distance(creature, food) < creature.size / 2) {
            creature.energy += food.energy
            creature.foodEaten++
            this.foods.splice(j, 1)
            j--
            
            if (creature.foodEaten >= 3) {
              creature.foodEaten = 0
              this.reproduceCreature(creature)
            }
          }
        }
        
        // Столкновение между существами
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
      // Одно существо может съесть другое, если оно больше на 30%
      if (c1.size > c2.size * 1.3) {
        this.creatureEatsCreature(c1, c2)
      } else if (c2.size > c1.size * 1.3) {
        this.creatureEatsCreature(c2, c1)
      } else {
        // Отталкивание одинаковых по размеру
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
    
    creatureEatsCreature(predator, prey) {
      predator.energy += prey.energy * 0.7
      predator.foodEaten++
      predator.grow() // Увеличиваем размер хищника
      
      const index = this.creatures.indexOf(prey)
      if (index !== -1) {
        this.creatures.splice(index, 1)
        this.stats.predationDeaths++
      }
      
      if (predator.foodEaten >= 3) {
        predator.foodEaten = 0
        this.reproduceCreature(predator)
      }
    },
    
    reproduceCreature(parent) {
      const size = 20 + Math.random() * 5 // Потомки тоже крупнее
      const speed = 0.7 + Math.random() * 0.3
      const vision = 70 + Math.random() * 30
      
      this.creatures.push(new Creature(
        parent.x + (Math.random() * 40 - 20),
        parent.y + (Math.random() * 40 - 20),
        size,
        speed,
        vision
      ))
      this.stats.newCreatures++
    },
    
    distance(a, b) {
      return Math.sqrt(Math.pow(a.x - b.x, 2) + Math.pow(a.y - b.y, 2))
    },
    
    updateStats() {
      if (this.creatures.length === 0) return
      
      this.stats.maxSize = Math.max(...this.creatures.map(c => c.size))
      this.stats.maxSpeed = Math.max(...this.creatures.map(c => c.speed))
      this.stats.maxVision = Math.max(...this.creatures.map(c => c.vision))
    },
    
    draw() {
      this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height)
      
      // Границы
      this.ctx.strokeStyle = '#ffffff'
      this.ctx.lineWidth = 2
      this.ctx.strokeRect(50, 50, 600, 600)
      
      // Еда
      this.foods.forEach(food => {
        this.ctx.fillStyle = food.color
        this.ctx.beginPath()
        this.ctx.arc(food.x, food.y, food.size, 0, Math.PI * 2)
        this.ctx.fill()
      })
      
      // Существа
      this.creatures.forEach(creature => {
        // Тело
        this.ctx.fillStyle = creature.color
        this.ctx.beginPath()
        this.ctx.arc(creature.x, creature.y, creature.size / 2, 0, Math.PI * 2)
        this.ctx.fill()
        
        // Глаза (красные если в режиме охоты)
        this.ctx.fillStyle = creature.isHunting ? '#ff0000' : '#ffffff'
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
/* Стили остаются без изменений */
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