<template>
  <div class="wheel-container">
    <div class="relative-wrapper">
      <svg
        :width="size"
        :height="size"
        viewBox="0 0 100 100"
        class="progress-ring"
      >
        <circle
          class="track"
          cx="50"
          cy="50"
          :r="radius"
          fill="transparent"
          stroke="#e5e7eb"
          :stroke-width="strokeWidth"
        />

        <circle
          class="progress"
          cx="50"
          cy="50"
          :r="radius"
          fill="transparent"
          :stroke="color"
          :stroke-width="strokeWidth"
          stroke-linecap="round"
          :stroke-dasharray="circumference + ' ' + circumference"
          :style="{ strokeDashoffset: strokeDashoffset }"
        />
      </svg>

      <div class="center-content">
        <span class="percentage" :style="{ color: color }">{{ percentage }}%</span>
      </div>
    </div>
    
    <p class="label">{{ label }}</p>
  </div>
</template>

<script>
export default {
  name: "ProgressWheel",
  props: {
    percentage: {
      type: Number,
      required: true,
      default: 0
    },
    label: {
      type: String,
      default: ""
    },
    color: {
      type: String,
      default: "#3b82f6" // Default Blue
    },
    size: {
      type: Number,
      default: 150 // Pixel width/height
    },
    strokeWidth: {
      type: Number,
      default: 10
    }
  },
  computed: {
    radius() {
      return 50 - this.strokeWidth / 2;
    },
    circumference() {
      return 2 * Math.PI * this.radius;
    },
    strokeDashoffset() {
      // Logic: Offset = Circumference - (Percent / 100 * Circumference)
      return this.circumference - (this.percentage / 100) * this.circumference;
    }
  }
};
</script>

<style scoped>
.wheel-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  font-family: -apple-system, sans-serif;
}

.relative-wrapper {
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
}

.progress-ring {
  transform: rotate(-90deg); /* Start from top */
  transform-origin: 50% 50%;
}

.track {
  opacity: 0.3;
}

.progress {
  transition: stroke-dashoffset 0.8s ease-in-out; /* Smooth animation */
}

.center-content {
  position: absolute;
  text-align: center;
}

.percentage {
  font-size: 2rem; /* Big bold text */
  font-weight: 800;
}

.label {
  margin-top: 15px;
  font-size: 0.95rem;
  color: #4b5563;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}
</style>