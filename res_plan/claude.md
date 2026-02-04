# ERP Hub Resource Planning Dashboard - Documentation

## Overview
This is a standalone HTML-based resource planning system for ERP projects. It matches demand (project requirements) with supply (available resources) to help with capacity planning and utilization forecasting.

**Key Feature**: All data is stored in the HTML file itself (localStorage), making it completely portable and self-contained.

---

## Current Structure

The application has **3 main tabs**:

### 1. Resources Tab (👥 Resources)
Manage your team members and their skills.

**Features**:
- Add/edit/delete resources
- Define primary and secondary skills
- Set resource levels (Junior, Mid, Senior, Lead, Principal)
- Track status (Active, Bench, Leave)
- Configure hours per week and cost rate
- Manage custom skills and levels

### 2. Projects Tab (📁 Projects)
Manage your project portfolio.

**Features**:
- Add/edit/delete projects
- Define project type (Active or Planned)
- Set probability percentage (for forecasting)
- Configure priority (High, Medium, Low)
- Define start/end dates and required FTE
- **Positions Management**: Each project can have multiple positions based on required FTE
- Expandable rows to view and edit positions
- Assign resources to positions directly

**Position Properties**:
- Position title (e.g., "Lead Developer", "Backend Developer")
- Resource level requirement
- Capacity percentage (0-100%)
- Expected start/end dates
- Resource allocation (assign specific team members)

### 3. Dashboard Tab (📊 Dashboard)
Real-time analytics and visualizations.

**Features**:
- 5 key metrics cards
- 4 interactive charts
- Resource utilization heatmap
- Filter by skill and project type
- PDF export capability

---

## How Allocations Work

### ⚠️ **IMPORTANT CHANGE**: Allocations Are Now Derived

**Previous Structure**:
- Had a separate "Current Allocation" tab where allocations were manually created and managed
- Required manual synchronization between Projects and Allocations

**Current Structure**:
- **No separate Allocation tab** - allocations are automatically derived from project positions
- When you assign a resource to a project position, the system automatically creates monthly allocations
- Allocation months are calculated from the position's expected start/end dates
- Allocation percentage comes from the position's capacity setting

### How It Works (Technical):

1. **Data Storage**: Project positions are stored within each project object:
   ```javascript
   project.positions = [
     {
       posId: "POS001",
       position: "Lead Developer",
       resourceLevel: "Senior",
       capacity: 100,
       expectedStart: "2025-01-15",
       expectedEnd: "2025-06-30",
       resourceId: "R001"  // Assigned resource
     }
   ]
   ```

2. **Allocation Derivation**: The `getDerivedAllocations()` function (line 1582):
   - Iterates through all projects
   - For each position with an assigned resource and dates
   - Generates monthly allocations between start and end dates
   - Creates allocation objects with: resourceId, projectId, month, percent, role

3. **Usage**: All dashboard calculations use `getDerivedAllocations()` through the `getFilteredData()` function

**Benefits**:
- ✅ Single source of truth (project positions)
- ✅ No manual synchronization needed
- ✅ Automatically updates when positions change
- ✅ Simpler data model

---

## Dashboard Metrics

### Metric Cards (Top Row)

| Metric | Calculation | Source |
|--------|-------------|--------|
| **Active Projects** | Count of projects with type="Active" | `fd.projects` |
| **Planned Projects** | Count of projects with type="Planned" | `fd.projects` |
| **Weighted FTE** | Sum of planned project FTE × probability% | `fd.projects` |
| **Total Resources** | Count of resources | `fd.resources` |
| **Available** | Resources with avg utilization < 50% | Derived allocations |
| **Overloaded** | Resources with any month > 100% utilization | Derived allocations |

**Average Utilization Calculation**:
- For each resource, collect all allocation percentages
- Calculate average of non-zero months
- Sum all averages and divide by resource count

---

## Chart Data Sources & Calculations

### 1. Project Pipeline (Horizontal Bar Chart)
**Location**: Dashboard, top-left
**Purpose**: Show FTE demand per project

**Data Source**: `fd.projects` (filtered projects)
**Calculation**:
- X-axis: Required FTE (from project.requiredFTE)
- Y-axis: Project names (truncated to 18 chars)
- Color: Green (Active) or Blue (Planned)
- Sorted by: Type (Active first), then probability (descending)

**Reference**: Lines 1698-1726

---

### 2. Aggregate Utilization (Line Chart)
**Location**: Dashboard, top-right
**Purpose**: Show team capacity utilization over time

**Data Source**: `fd.allocations` (derived from project positions)
**Calculation**:
1. Extract unique months from all allocations
2. For each month, sum: allocation.percent × project.probability / 100
3. Calculate utilization %: (total allocation / team capacity) × 100
   - Team capacity = number of resources × 100
4. Compare against 100% target line

**Key Feature**: Probability-weighted (planned projects count fractionally based on likelihood)

**Reference**: Lines 1728-1760

---

### 3. Projected Revenue (Bar Chart)
**Location**: Dashboard, middle-left
**Purpose**: Forecast monthly revenue based on resource allocation and rates

**Data Source**:
- `fd.allocations` (derived)
- `data.resources` (for cost rates)
- `data.projects` (for probability)

**Calculation**:
For each allocation:
```
monthly_revenue = resource.costRate × (allocation.percent / 100) ×
                  resource.hoursPerWeek × 4.33 × (project.probability / 100)
```

Where:
- `costRate`: $/hour for the resource
- `allocation.percent`: Capacity allocated (0-100%)
- `hoursPerWeek`: Typically 40
- `4.33`: Average weeks per month (52/12)
- `probability/100`: Weight for planned projects

**Display**: Revenue in thousands ($K)

**Reference**: Lines 1762-1791

---

### 4. Supply vs Demand (Grouped Bar Chart)
**Location**: Dashboard, middle-right
**Purpose**: Show capacity gap by skill area

**Data Source**:
- Supply: `fd.resources` (count by primary skill)
- Demand: `fd.allocations` (derived, weighted by probability)

**Calculation**:

**Supply**:
- Count resources grouped by primarySkill
- Shows available headcount per skill

**Demand**:
1. For each allocation, get resource's primary skill
2. Calculate weighted demand: (allocation.percent / 100) × (project.probability / 100)
3. Sum by skill
4. Divide by number of months (to get average monthly demand)

**Interpretation**:
- Supply > Demand: Excess capacity in that skill
- Demand > Supply: Shortage/overallocation in that skill

**Reference**: Lines 1793-1828

---

### 5. Resource Utilization Heatmap (Table)
**Location**: Dashboard, bottom (full width)
**Purpose**: Show utilization per resource per month

**Data Source**: Calls `getDerivedAllocations()` directly (line 1833)

**Calculation**:
1. Extract all unique months from allocations
2. For each resource and month:
   - Sum all allocation percentages for that resource in that month
   - A resource can have multiple allocations in one month (different projects)
3. Calculate row average (non-zero months only)

**Color Coding**:
| Color | Range | Class |
|-------|-------|-------|
| Gray | 0% | Available |
| Red (light) | <50% | Low utilization |
| Yellow | 50-70% | Medium |
| Green | 70-90% | Optimal |
| Blue | 90-100% | High |
| Red (dark) | >100% | Overloaded |

**Reference**: Lines 1830-1859

---

## How to Use the System

### Adding a New Project

1. Go to **Projects** tab
2. Click **"+ Add Project"**
3. Fill in project details:
   - Name, Client, Type (Active/Planned)
   - Probability % (100 for Active, 0-100 for Planned)
   - Priority, Start/End dates
   - Required FTE (number of positions needed)
4. **Configure Positions** (auto-generated based on FTE):
   - Edit position titles (e.g., "Lead Developer", "QA Engineer")
   - Set required resource level
   - Set capacity % (usually 100% for full-time)
   - Adjust start/end dates if needed
   - Assign resources (or leave unassigned)
5. Click **"Save Project"**

**Result**: Allocations are automatically created for all assigned positions

### Managing Resource Allocations

**Option 1: From Projects Tab**
1. Find the project in the table
2. Click the **expand button (▶)** to show positions
3. Edit position details inline:
   - Change assigned resource
   - Adjust capacity %
   - Modify dates
4. Changes auto-save

**Option 2: Using Edit Modal**
1. Click the **edit button (✎)** on a project row
2. Update positions in the modal
3. Click **"Update Project"**

**Note**: When you change a position's dates or capacity, the dashboard automatically recalculates allocations

### Viewing Utilization

1. Go to **Dashboard** tab
2. Check the **Overloaded** metric card (shows resources >100%)
3. Scroll to **Resource Utilization Heatmap**
4. Look for red cells (>100%) or dark red (overloaded)
5. Identify which months and resources are overallocated

**To resolve overload**:
- Go back to Projects tab
- Reduce capacity % on positions for overloaded resources
- Or reassign positions to other resources
- Or adjust project dates to spread load

### Filters

**Dashboard Filters**:
- **Skill Filter**: Show only projects using specific skill
- **Type Filter**: Active only, Planned only, or All
- **Reset**: Clear all filters

**Note**: Filters affect all charts and metrics simultaneously

### Export & Import

**Save File**:
- Click **"💾 Save File"** in header
- Downloads a standalone HTML file with current data
- File is fully portable and can be opened in any browser

**Import Data**:
- Click **"📥 Import"** in header
- Select a JSON file with resource/project data
- Old allocation format is automatically migrated to positions

**Export PDF**:
- Go to Dashboard tab
- Click **"📄 Export PDF"**
- Generates a snapshot of current dashboard

---

## Data Model

### Resources
```javascript
{
  id: "R001",
  name: "Alice Johnson",
  primarySkill: "Frontend",
  secondarySkill: "UI/UX",
  resourceLevel: "Senior",
  status: "Active",          // Active, Bench, or Leave
  hoursPerWeek: 40,
  costRate: 150              // $/hour
}
```

### Projects
```javascript
{
  id: "P001",
  name: "E-Commerce Platform",
  client: "RetailCorp",
  type: "Active",            // Active or Planned
  probability: 100,          // 0-100%
  priority: "High",          // High, Medium, Low
  startDate: "2025-01-15",
  endDate: "2025-06-30",
  requiredFTE: 3,
  positions: [...]           // Array of position objects
}
```

### Positions (within Projects)
```javascript
{
  posId: "POS001",
  position: "Lead Developer",
  resourceLevel: "Lead",
  capacity: 100,                    // 0-100%
  expectedStart: "2025-01-15",
  expectedEnd: "2025-06-30",
  resourceId: "R001"                // Assigned resource (or "" if unassigned)
}
```

### Derived Allocations (Auto-generated)
```javascript
{
  id: "A001",
  resourceId: "R001",
  projectId: "P001",
  month: "Jan-2025",               // Format: "MMM-YYYY"
  percent: 100,                    // From position.capacity
  role: "Lead Developer"           // From position.position
}
```

---

## Technical Details

### Key Functions

| Function | Purpose | Location |
|----------|---------|----------|
| `getDerivedAllocations()` | Converts project positions to monthly allocations | Line 1582 |
| `getFilteredData()` | Applies dashboard filters and returns filtered datasets | Line 1640 |
| `renderDashboard()` | Orchestrates all dashboard updates | Line 1660 |
| `updateMetrics()` | Calculates and displays metric cards | Line 1670 |
| `renderHeatmap()` | Generates utilization heatmap table | Line 1830 |
| `getMonthsBetween()` | Generates month array from date range | Line 1614 |

### Data Persistence
- Uses `localStorage.setItem('erpPlanningData', JSON.stringify(data))`
- Auto-saves on every change
- Loaded on page initialization
- Can be exported to standalone HTML file

### Month Format
- Format: `"MMM-YYYY"` (e.g., "Jan-2025", "Feb-2025")
- Sorted chronologically in charts and heatmap
- Generated from position start/end dates

---

## Migration Notes

### Changes from Previous Version

**✅ Completed**:
1. ❌ Removed "Current Allocation" tab from navigation
2. ✅ Allocations now derived from project positions
3. ✅ Updated import function to not require allocations
4. ✅ All dashboard calculations use derived allocations
5. ✅ Comments added explaining the new structure

**📋 What Was Kept**:
- All dashboard charts and metrics
- PDF export functionality
- Skills and resource levels management
- Project management with positions
- Filter functionality

**🔄 Backward Compatibility**:
- Import function handles old data format with separate allocations
- Automatically migrates projects to include positions if missing
- Cleans up old allocation data on import

---

## Validation Checks

### Dashboard Calculations Verified ✅

| Component | Data Source | Status |
|-----------|-------------|--------|
| Active Projects Metric | fd.projects | ✅ Correct |
| Planned Projects Metric | fd.projects | ✅ Correct |
| Weighted FTE Metric | fd.projects | ✅ Correct |
| Available Metric | Derived allocations | ✅ Correct |
| Overloaded Metric | Derived allocations | ✅ Correct |
| Pipeline Chart | fd.projects | ✅ Correct |
| Utilization Chart | Derived allocations | ✅ Correct (probability-weighted) |
| Revenue Chart | Derived allocations + cost rates | ✅ Correct (probability-weighted) |
| Supply vs Demand Chart | fd.resources + derived allocations | ✅ Correct |
| Heatmap | getDerivedAllocations() | ✅ Correct |

**All calculations are using the correct data sources and formulas.**

---

## Common Tasks

### Check Team Capacity
1. Dashboard → Total Resources metric
2. Multiply by 100 to get total capacity percentage points
3. Example: 10 resources = 1000% total capacity

### Find Available Resources
1. Dashboard → Heatmap
2. Look for gray/light-red rows (low utilization)
3. Or check "Available" metric card for count

### Identify Resource Gaps
1. Dashboard → Supply vs Demand chart
2. Look for skills where Demand (blue) > Supply (green)
3. Consider hiring or training in those skills

### Forecast Revenue
1. Dashboard → Projected Revenue chart
2. Shows probability-weighted monthly revenue
3. Planned projects count fractionally based on probability

### Plan Project Staffing
1. Projects tab → Add or edit project
2. Set Required FTE based on project needs
3. System creates that many positions
4. Assign resources to positions
5. Check Dashboard to see impact on utilization

---

## Troubleshooting

### Resources Show >100% Utilization
**Cause**: Resource assigned to multiple positions overlapping in time
**Solution**:
- Reduce capacity % on one or more positions
- Or reassign some positions to other resources
- Or adjust project timelines to reduce overlap

### Revenue Seems Low for Planned Projects
**Cause**: Revenue is probability-weighted
**Solution**: This is expected - planned projects with 50% probability count as 50% of full revenue

### Heatmap Shows No Data
**Cause**: No resources assigned to project positions
**Solution**: Go to Projects tab, expand projects, assign resources to positions

### Dashboard Shows All Zeros
**Cause**: No projects with assigned resources
**Solution**: Add projects and assign resources to positions

---

## Best Practices

1. **Always assign resources to positions** - Unassigned positions don't show in dashboard
2. **Use realistic probability %** - Be honest about project likelihood for accurate forecasting
3. **Set appropriate capacity %** - Use <100% for part-time or shared resources
4. **Review heatmap regularly** - Catch overallocation early
5. **Update project dates** - Keep timelines current for accurate planning
6. **Check supply vs demand** - Identify skill gaps before they become problems
7. **Export PDF regularly** - Create snapshots for stakeholder meetings
8. **Save file periodically** - Create backups of your data

---

## Summary

This ERP Resource Planning Dashboard provides a complete solution for matching project demand with team supply. The key innovation is **automatic allocation derivation from project positions**, eliminating manual allocation management and ensuring data consistency.

**Three-Tab Structure**:
- **Resources**: Manage your team
- **Projects**: Define projects and positions
- **Dashboard**: Analyze and forecast

**All dashboard metrics and charts correctly use derived allocations from project positions, with probability-weighting for planned projects to provide realistic forecasts.**
