<template>
    <div class="container">
        <div class="left-panel">
            <n-card title="学生课表" class="full-height-card"
                content-style="display: flex; flex-direction: column; padding: 0; overflow: hidden;"
                header-style="padding: 10px 16px; flex-shrink: 0;">
                <template #header-extra>
                    <n-space>
                        <n-button size="small" secondary type="primary" @click="handleExport">
                            导出配置
                        </n-button>
                        <n-button size="small" secondary type="info" @click="triggerImport">
                            导入配置
                        </n-button>
                    </n-space>
                </template>

                <div class="total-credits">
                    <n-statistic label="" :value="totalCredits">
                        <template #prefix>总学分:</template>
                    </n-statistic>
                </div>

                <div class="schedule-container">
                    <div class="header-row">
                        <div class="time-col-header"></div>
                        <div v-for="day in days" :key="day" class="day-header">{{ day }}</div>
                    </div>

                    <div class="schedule-body">
                        <div v-for="(slot, index) in timeSlots" :key="index" class="time-row">
                            <div class="time-col">
                                <div class="slot-idx">{{ index + 1 }}</div>
                                <div class="slot-time">{{ slot[0] }}</div>
                                <div class="slot-time">{{ slot[1] }}</div>
                            </div>
                            <div v-for="(day, dayIdx) in days" :key="dayIdx" class="course-cell">
                                <div v-for="course in getCoursesForCell(day, index + 1)" :key="course.id"
                                    class="course-block" :style="{ backgroundColor: course.color }">
                                    <div class="course-name">{{ course.name }}</div>
                                    <div class="course-info">{{ course.weeks }} @ {{ course.location }}</div>
                                    <div class="course-teacher">{{ course.teacher }}</div>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
            </n-card>
        </div>

        <div class="right-panel">
            <n-card title="选课中心" class="full-height-card"
                content-style="display: flex; flex-direction: column; padding: 10px; overflow: hidden;"
                header-style="padding: 10px 16px; flex-shrink: 0;">
                <div class="search-bar">
                    <n-input v-model:value="searchText" placeholder="搜索课程名称 / 教师姓名..." clearable />
                </div>

                <div class="table-wrapper">
                    <n-data-table :columns="columns" :data="filteredCourses" :pagination="pagination" flex-height
                        class="fill-height-table" />
                </div>
            </n-card>
        </div>

        <n-modal v-model:show="showDetailModal" preset="card" style="width: 600px;" title="课程详细信息">
            <n-descriptions bordered :column="2">
                <n-descriptions-item v-for="(value, key) in currentDetailRow" :key="key" :label="key">
                    {{ value || '-' }}
                </n-descriptions-item>
            </n-descriptions>
        </n-modal>

        <input type="file" ref="fileInput" style="display: none" accept=".json" @change="handleFileChange">
    </div>
</template>

<script setup>
import { ref, computed, onMounted, h } from 'vue'
import { NCard, NStatistic, NInput, NDataTable, NButton, useMessage, NModal, NDescriptions, NDescriptionsItem, NSpace } from 'naive-ui'
import Papa from 'papaparse'

const days = ['周一', '周二', '周三', '周四', '周五', '周六', '周日']
const timeSlots = [
    ["8:00", "8:40"], ["8:45", "9:25"], ["9:30", "10:10"], ["10:30", "11:10"],
    ["11:15", "11:55"], ["14:30", "15:10"], ["15:15", "15:55"], ["16:05", "16:45"],
    ["16:50", "17:30"], ["18:40", "19:20"], ["19:25", "20:05"], ["20:10", "20:50"],
    ["20:55", "21:35"]
]

const rawCourses = ref([])
const searchText = ref('')
const selectedCourseIds = ref(new Set())
const message = useMessage()
const fileInput = ref(null)

const showDetailModal = ref(false)
const currentDetailRow = ref({})

const parseRange = (str) => {
    const res = []
    const parts = str.split(',')
    parts.forEach(p => {
        if (p.includes('-')) {
            const [start, end] = p.split('-').map(Number)
            for (let i = start; i <= end; i++) res.push(i)
        } else {
            res.push(Number(p))
        }
    })
    return res
}

const parseTimeStr = (str) => {
    if (!str) return []
    const regex = /((?:[\d,-]+周,?)+)([\d,-]+)节([^(]*)/g
    const results = []
    let match
    while ((match = regex.exec(str)) !== null) {
        results.push({
            weeks: match[1].replace(/周/g, ''),
            sessions: parseRange(match[2]),
            sessionStr: match[2],
            location: match[3] || '未知地点'
        })
    }
    return results
}

const summarizeTimeLocation = (row) => {
    const summaries = []
    days.forEach(day => {
        if (row[day]) {
            const parsed = parseTimeStr(row[day])
            parsed.forEach(p => {
                summaries.push(`${day}${p.sessionStr}节[${p.location}]`)
            })
        }
    })
    return summaries.join('; ')
}

onMounted(() => {
    Papa.parse('/courses.csv', {
        download: true,
        header: true,
        complete: (results) => {
            rawCourses.value = results.data
                .filter(row => row['课程名称'])
                .map((row, index) => ({
                    id: index, // 使用索引作为ID，需注意CSV文件变动会导致ID失效
                    name: row['课程名称'],
                    teacher: row['成绩录入老师'] || row['其他教师'] || '',
                    credit: parseFloat(row['学分']) || 0,
                    rawRow: row,
                    timeLocDisplay: summarizeTimeLocation(row),
                    note: row['选课备注'] || ''
                }))
        }
    })
})


const handleExport = () => {
    if (selectedCourseIds.value.size === 0) {
        message.warning('当前没有选择任何课程')
        return
    }
    // 将 Set 转换为 Array 导出
    const dataToSave = Array.from(selectedCourseIds.value)
    const blob = new Blob([JSON.stringify(dataToSave)], { type: 'application/json' })
    const url = URL.createObjectURL(blob)

    const a = document.createElement('a')
    a.href = url
    a.download = `schedule-config-${new Date().toISOString().slice(0, 10)}.json`
    document.body.appendChild(a)
    a.click()
    document.body.removeChild(a)
    URL.revokeObjectURL(url)
    message.success('配置已导出')
}

const triggerImport = () => {
    fileInput.value.click()
}

const handleFileChange = (event) => {
    const file = event.target.files[0]
    if (!file) return

    const reader = new FileReader()
    reader.onload = (e) => {
        try {
            const importedIds = JSON.parse(e.target.result)
            if (Array.isArray(importedIds)) {
                const validIds = importedIds.filter(id => id >= 0 && id < rawCourses.value.length)
                selectedCourseIds.value = new Set(validIds)
                message.success(`成功导入 ${validIds.length} 门课程`)
            } else {
                message.error('文件格式错误')
            }
        } catch (err) {
            console.error(err)
            message.error('无法解析文件')
        }
        event.target.value = ''
    }
    reader.readAsText(file)
}

// --- 计算属性 ---
const filteredCourses = computed(() => {
    //   if (!searchText.value) return rawCourses.value.slice(0, 100)
    const lowerText = searchText.value.toLowerCase()
    return rawCourses.value.filter(c =>
        (c.name && c.name.toLowerCase().includes(lowerText)) ||
        (c.teacher && c.teacher.toLowerCase().includes(lowerText))
    )
})

const selectedCoursesData = computed(() => {
    return rawCourses.value.filter(c => selectedCourseIds.value.has(c.id))
})

const totalCredits = computed(() => {
    return selectedCoursesData.value.reduce((sum, c) => sum + c.credit, 0)
})

const getCoursesForCell = (day, sessionNum) => {
    const cellCourses = []
    selectedCoursesData.value.forEach(course => {
        const timeStr = course.rawRow[day]
        if (!timeStr) return
        const timeBlocks = parseTimeStr(timeStr)
        timeBlocks.forEach(block => {
            if (block.sessions.includes(sessionNum)) {
                cellCourses.push({
                    id: course.id,
                    name: course.name,
                    teacher: course.teacher,
                    weeks: block.weeks + '周',
                    location: block.location,
                    color: stringToColor(course.name)
                })
            }
        })
    })
    return cellCourses
}

const stringToColor = (str) => {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
        hash = str.charCodeAt(i) + ((hash << 5) - hash);
    }
    const c = (hash & 0x00FFFFFF).toString(16).toUpperCase();
    return '#' + "00000".substring(0, 6 - c.length) + c + '80';
}

const toggleCourse = (row) => {
    if (selectedCourseIds.value.has(row.id)) {
        selectedCourseIds.value.delete(row.id)
        message.warning(`已退选: ${row.name}`)
    } else {
        selectedCourseIds.value.add(row.id)
        message.success(`已选择: ${row.name}`)
    }
    selectedCourseIds.value = new Set(selectedCourseIds.value)
}

const openDetail = (row) => {
    currentDetailRow.value = row.rawRow
    showDetailModal.value = true
}

const columns = [
    {
        title: '课名',
        key: 'name',
        width: 150,
        fixed: 'left',
        render(row) {
            return h(
                NButton,
                {
                    text: true,
                    type: 'info',
                    style: { fontWeight: 'bold' },
                    onClick: () => openDetail(row)
                },
                { default: () => row.name }
            )
        }
    },
    { title: '教师', key: 'teacher', width: 80, ellipsis: true },
    { title: '学分', key: 'credit', width: 60, align: 'center' },
    {
        title: '时间/地点',
        key: 'timeLocDisplay',
        width: 200,
        ellipsis: { tooltip: true }
    },
    { title: '备注', key: 'note', width: 100, ellipsis: { tooltip: true } },
    {
        title: '操作',
        key: 'actions',
        width: 80,
        fixed: 'right',
        render(row) {
            const isSelected = selectedCourseIds.value.has(row.id)
            return h(
                NButton,
                {
                    size: 'tiny',
                    type: isSelected ? 'error' : 'primary',
                    onClick: () => toggleCourse(row)
                },
                { default: () => isSelected ? '退选' : '选课' }
            )
        }
    }
]

const pagination = {
    pageSize: 20,
    showSizePicker: true,
    pageSizes: [3, 5, 7],
    onChange: (page) => {
        pagination.page = page
    },
    onUpdatePageSize: (pageSize) => {
        pagination.pageSize = pageSize
        pagination.page = 1
    }
}
</script>

<style scoped>
.container {
    display: flex;
    height: 100vh;
    padding: 10px;
    box-sizing: border-box;
    gap: 10px;
    background-color: #f0f2f5;
    overflow: hidden;
}

.full-height-card {
    height: 100%;
    display: flex;
    flex-direction: column;
}

.left-panel {
    width: 30%;
    height: 100%;
    flex-shrink: 0;
}

.total-credits {
    padding: 8px;
    background: #e6fffb;
    border-bottom: 1px solid #d9d9d9;
    text-align: center;
    flex-shrink: 0;
}

.schedule-container {
    flex: 1;
    overflow-y: auto;
    border: 1px solid #eee;
    font-size: 12px;
    display: flex;
    flex-direction: column;
}

.header-row {
    display: flex;
    background: #fafafa;
    position: sticky;
    top: 0;
    z-index: 10;
    border-bottom: 1px solid #ddd;
}

.time-col-header {
    width: 40px;
    flex-shrink: 0;
    border-right: 1px solid #eee;
}

.day-header {
    flex: 1;
    text-align: center;
    padding: 5px 0;
    font-weight: bold;
    border-right: 1px solid #eee;
    min-width: 0;
}

.time-row {
    display: flex;
    border-bottom: 1px solid #eee;
    min-height: 55px;
}

.time-col {
    width: 40px;
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    background: #fafafa;
    border-right: 1px solid #eee;
    font-size: 10px;
    color: #666;
}

.course-cell {
    flex: 1;
    border-right: 1px solid #eee;
    padding: 1px;
    min-width: 0;
}

.course-block {
    font-size: 10px;
    padding: 1px 2px;
    border-radius: 3px;
    margin-bottom: 1px;
    color: #fff;
    line-height: 1.1;
    cursor: default;
}

.course-name {
    font-weight: bold;
    transform: scale(0.95);
    transform-origin: left top;
}

.course-info {
    font-size: 9px;
    opacity: 0.9;
    transform: scale(0.9);
    transform-origin: left top;
}

.course-teacher {
    font-size: 9px;
    opacity: 0.8;
    transform: scale(0.9);
    transform-origin: left top;
}

.right-panel {
    flex: 1;
    height: 100%;
    display: flex;
    flex-direction: column;
    min-width: 0;
}

.search-bar {
    margin-bottom: 10px;
    flex-shrink: 0;
}

.table-wrapper {
    flex: 1;
    overflow: hidden;
    display: flex;
    flex-direction: column;
}

.fill-height-table {
    height: 100%;
}
</style>