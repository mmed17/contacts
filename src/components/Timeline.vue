<template>
  <div ref="visualization" class="vis-modern-container"></div>
</template>

<script>
import { DataSet } from "vis-data";
import { Timeline } from "vis-timeline/peer";
import "vis-timeline/styles/vis-timeline-graph2d.css";
import moment from "moment";

export default {
  name: "Timeline",
  data() {
    return {
      timeline: null,
    };
  },
  mounted() {
    // 1. Define "Lanes" (Groups) to allow vertical movement
    // We create 5 generic lanes so users can organize tasks vertically
    const groups = new DataSet([
      { id: 1, content: "Strategy", className: "vis-group-modern" },
      { id: 2, content: "Design",   className: "vis-group-modern" },
      { id: 3, content: "Dev",      className: "vis-group-modern" },
      { id: 4, content: "QA",       className: "vis-group-modern" },
      { id: 5, content: "Release",  className: "vis-group-modern" }
    ]);

    // 2. Define Initial Tasks
    const now = moment().startOf('day');
    const items = new DataSet([
      {
        id: 1,
        group: 1, // Put in 'Strategy' lane
        content: "Project Kickoff",
        start: now.clone().toDate(),
        end: now.clone().add(3, "days").toDate(),
        type: "range",
        className: "blue-task"
      },
      {
        id: 2,
        group: 2, // Put in 'Design' lane
        content: "UI Mockups",
        start: now.clone().add(2, "days").toDate(),
        end: now.clone().add(6, "days").toDate(),
        type: "range",
        className: "purple-task"
      }
    ]);

    // 3. Configuration
    const options = {
      height: "500px",
      start: now.clone().subtract(2, 'days').toDate(),
      end: now.clone().add(14, 'days').toDate(),
      
      // Interaction Settings
      editable: {
        add: true,         // Double-click empty space
        updateTime: true,  // Drag and stretch
        updateGroup: true, // Drag vertically between lanes
        remove: true       // Delete key
      },
      
      // Make it a "Range" by default when adding
      type: "range",
      stack: false,        // Allow overlapping in the same lane? False = stack them.
      
      // UX Snapping (Snap to nearest hour)
      snap: function (date, scale, step) {
        const hour = 60 * 60 * 1000;
        return Math.round(date / hour) * hour;
      },

      // --- HANDLERS ---
      
      // 1. ADD NEW TASK
      onAdd: (item, callback) => {
        const title = prompt("Enter Task Title:", "New Item");
        if (title) {
          item.content = title;
          // Force it to be a box (Range) of 2 days, not a single point
          item.end = moment(item.start).add(2, 'days').toDate(); 
          item.className = "blue-task"; // Default color
          callback(item);
        } else {
          callback(null); // Cancel
        }
      },

      // 2. EDIT TASK (Double Click)
      onUpdate: (item, callback) => {
        // Simple Logic: Double click to rename
        const title = prompt("Edit Task Title:", item.content);
        if (title) {
          item.content = title;
          callback(item);
        } else {
          callback(null);
        }
      },

      // 3. MOVE / STRETCH
      onMove: (item, callback) => {
        // You can save to server here
        console.log(`Updated '${item.content}':`, item.start, item.end);
        callback(item);
      },

      // 4. REMOVE
      onRemove: (item, callback) => {
        if (confirm(`Delete '${item.content}'?`)) {
          callback(item);
        } else {
          callback(null);
        }
      },
    };

    // 4. Render
    this.timeline = new Timeline(this.$refs.visualization, items, groups, options);
  }
};
</script>

<style scoped>
/* --- VIS JS MODERN OVERRIDES --- */

/* Container */
:deep(.vis-modern-container) {
  border: 1px solid #e5e7eb;
  border-radius: 8px;
}

/* Timeline Background */
:deep(.vis-timeline) {
  border: none;
}

/* Group Labels (Left Column) */
:deep(.vis-labelset .vis-label) {
  border-bottom: 1px solid #f3f4f6;
  color: #374151;
  font-weight: 600;
  padding: 10px;
  display: flex;
  align-items: center;
}

/* Grid Lines */
:deep(.vis-time-axis .vis-grid.vis-minor) { border-color: #f3f4f6; }
:deep(.vis-time-axis .vis-grid.vis-major) { border-color: #e5e7eb; }

/* TASK STYLES - Common */
:deep(.vis-item) {
  border-width: 0;
  border-radius: 6px;
  font-size: 13px;
  font-weight: 500;
  box-shadow: 0 1px 2px rgba(0,0,0,0.1);
  transition: transform 0.1s, box-shadow 0.1s;
}

:deep(.vis-item:hover) {
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
  transform: translateY(-1px);
  z-index: 100;
}

:deep(.vis-item.vis-selected) {
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.4); /* Indigo ring */
  background-color: inherit;
  border-color: inherit;
}

/* Content Padding inside task */
:deep(.vis-item-content) {
  padding: 8px 12px;
}

/* --- COLOR VARIANTS --- */
/* You can randomly assign these classes or let user pick */

/* Blue Task */
:deep(.blue-task) {
  background: #3b82f6; /* Tailwind Blue 500 */
  color: white;
}
:deep(.blue-task.vis-selected) { background: #2563eb; }

/* Purple Task */
:deep(.purple-task) {
  background: #8b5cf6; /* Tailwind Purple 500 */
  color: white;
}
:deep(.purple-task.vis-selected) { background: #7c3aed; }

/* Drag Handles (The edges you pull to stretch) */
:deep(.vis-drag-left), :deep(.vis-drag-right) {
  background: rgba(255,255,255,0.4);
  width: 5px;
  border-radius: 10px;
}
</style>