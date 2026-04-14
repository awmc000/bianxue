<script setup lang="ts">
import { computed, reactive, ref } from 'vue'

type Screen = 'dashboard' | 'workspace' | 'results' | 'leaderboard'
type ErrorType =
  | 'Wrong Character'
  | 'Wrong Word Choice'
  | 'Word Order Error'
  | 'Missing Word'
  | 'Extra Word'
  | 'Grammar Pattern Error'
  | 'Measure Word Error'

interface JobError {
  id: string
  label: string
  type: ErrorType
  incorrect: string
  correction: string
  explanationEn: string
  explanationZh: string
}

interface Job {
  id: string
  title: string
  department: string
  level: string
  payout: number
  difficulty: string
  scenario: string[]
  memo: string
  text: string
  selectableSpans: string[]
  errors: JobError[]
}

interface AnnotationDraft {
  span: string
  errorType: ErrorType
  correction: string
}

interface AnnotationRecord extends AnnotationDraft {
  id: number
}

interface PracticeItem {
  id: string
  prompt: string
  answer: string
  refund: number
}

const jobs: Job[] = [
  {
    id: 'daily-204',
    title: 'Daily Contract 204',
    department: 'Local Correspondence Office',
    level: 'HSK 2',
    payout: 145,
    difficulty: 'Moderate',
    scenario: [
      'A deputy clerk submitted a neighborhood notice after a heroic battle with the office mimeograph.',
      'Your task is to clear it for distribution before the district lunch gong sounds.',
    ],
    memo: 'Flag one error per annotation. Broad highlights invite suspicion.',
    text: '昨天我们公司开了一个重要会意，经理说大家要早点回去准备明天的活动。因为时间很紧，所以每个人都要把自己的工作先完成好。',
    selectableSpans: ['会意', '早点回去', '完成好', '活动'],
    errors: [
      {
        id: 'e1',
        label: 'Wrong character in meeting noun',
        type: 'Wrong Character',
        incorrect: '会意',
        correction: '会议',
        explanationEn: '会意 means to grasp the meaning; 会议 is the noun for a meeting.',
        explanationZh: '“会意”表示领会意思，这里应该用表示会议的“会议”。',
      },
      {
        id: 'e2',
        label: 'Awkward word order in verb phrase',
        type: 'Word Order Error',
        incorrect: '完成好',
        correction: '好好完成',
        explanationEn: 'The adverbial emphasis belongs before the verb in this sentence.',
        explanationZh: '这里强调方式时，修饰成分应放在动词前面，所以更自然的是“好好完成”。',
      },
    ],
  },
  {
    id: 'queue-119',
    title: 'Queue Job 119',
    department: 'News Bureau',
    level: 'HSK 3',
    payout: 210,
    difficulty: 'High',
    scenario: [
      'A local bulletin was rushed to print after a rainstorm damaged the previous draft.',
      'Editorial command insists the tone remain calm, factual, and only mildly alarming.',
    ],
    memo: 'Expect denser mistakes and tighter score windows.',
    text: '今天早上很多市民都去公园参加植树活动，记者表示大家的热情非常高，而且现场准备了很多个工具给志愿者使用。',
    selectableSpans: ['很多个工具', '给志愿者使用', '非常高'],
    errors: [
      {
        id: 'e3',
        label: 'Measure word mismatch',
        type: 'Measure Word Error',
        incorrect: '很多个工具',
        correction: '很多工具',
        explanationEn: 'Tools usually do not take 个 here; omitting it is more natural.',
        explanationZh: '这里“工具”前一般不需要“个”，直接说“很多工具”更自然。',
      },
      {
        id: 'e4',
        label: 'Word choice in enthusiasm phrase',
        type: 'Wrong Word Choice',
        incorrect: '非常高',
        correction: '很高',
        explanationEn: 'The phrase 热情很高 is standard; 非常高 is understandable but less idiomatic here.',
        explanationZh: '“热情很高”是更常见的搭配，这里用“非常高”不够自然。',
      },
    ],
  },
]

const practiceItems: PracticeItem[] = [
  {
    id: 'p1',
    prompt: 'Fill in the blank: 今天我们开了一个重要____。',
    answer: '会议',
    refund: 18,
  },
  {
    id: 'p2',
    prompt: 'Choose the more natural phrase: A) 完成好工作 B) 好好完成工作',
    answer: 'B',
    refund: 12,
  },
]

const initialJob = jobs[0]!
const dailyJob = jobs[0]!

const screen = ref<Screen>('dashboard')
const currentJobId = ref(initialJob.id)
const selectedSpan = ref(initialJob.selectableSpans[0]!)
const selectedAnnotationId = ref<number | null>(null)
const practiceCompleted = ref<string[]>([])
const exerciseResult = ref('')
const showReviewLanguage = ref<'en' | 'zh'>('en')

const draft = reactive<AnnotationDraft>({
  span: initialJob.selectableSpans[0]!,
  errorType: 'Wrong Character',
  correction: '会议',
})

const annotations = ref<AnnotationRecord[]>([
  {
    id: 1,
    span: '会意',
    errorType: 'Wrong Character',
    correction: '会议',
  },
])

const currentJob = computed<Job>(() => jobs.find((job) => job.id === currentJobId.value) ?? initialJob)

const matchedErrors = computed(() =>
  annotations.value
    .map((annotation) => {
      const matched = currentJob.value.errors.find(
        (error) =>
          error.incorrect === annotation.span &&
          error.type === annotation.errorType &&
          error.correction === annotation.correction.trim(),
      )
      return matched ? { annotation, error: matched } : null
    })
    .filter((item): item is { annotation: AnnotationRecord; error: JobError } => item !== null),
)

const missedErrors = computed(() =>
  currentJob.value.errors.filter(
    (error) => !matchedErrors.value.some((match) => match.error.id === error.id),
  ),
)

const falsePositives = computed(() =>
  annotations.value.filter(
    (annotation) =>
      !matchedErrors.value.some((match) => match.annotation.id === annotation.id),
  ),
)

const accuracyScore = computed(() => {
  const base = matchedErrors.value.length * 42
  const penalty = missedErrors.value.length * 18 + falsePositives.value.length * 12
  return Math.max(52, Math.min(98, base + 32 - penalty))
})

const payoutScore = computed(() => {
  const refund = practiceItems
    .filter((item) => practiceCompleted.value.includes(item.id))
    .reduce((sum, item) => sum + item.refund, 0)
  return Math.round((currentJob.value.payout * accuracyScore.value) / 100) + refund
})

const rankProgress = computed(() => Math.min(96, 38 + matchedErrors.value.length * 18))

const leaderboard = [
  { name: 'Archivist Maple', earnings: '¥2,480', accuracy: '96%' },
  { name: 'You', earnings: '¥1,920', accuracy: '89%' },
  { name: 'Deputy Lin', earnings: '¥1,870', accuracy: '87%' },
  { name: 'Clerk No. 7', earnings: '¥1,715', accuracy: '84%' },
]

function resetDraft(span?: string) {
  const nextSpan = span ?? currentJob.value.selectableSpans[0]!
  selectedSpan.value = nextSpan
  draft.span = nextSpan
  draft.errorType = 'Wrong Character'
  draft.correction = ''
  selectedAnnotationId.value = null
}

function startJob(jobId: string) {
  currentJobId.value = jobId
  annotations.value = [
    {
      id: 1,
      span: jobs.find((job) => job.id === jobId)?.selectableSpans[0] ?? '会意',
      errorType: 'Wrong Character',
      correction: jobId === 'daily-204' ? '会议' : '',
    },
  ]
  practiceCompleted.value = []
  exerciseResult.value = ''
  showReviewLanguage.value = 'en'
  resetDraft(jobs.find((job) => job.id === jobId)?.selectableSpans[0] ?? '会意')
  screen.value = 'workspace'
}

function chooseSpan(span: string) {
  selectedSpan.value = span
  draft.span = span
}

function saveAnnotation() {
  if (!draft.correction.trim()) {
    return
  }

  if (selectedAnnotationId.value !== null) {
    annotations.value = annotations.value.map((annotation) =>
      annotation.id === selectedAnnotationId.value ? { ...annotation, ...draft } : annotation,
    )
  } else {
    annotations.value = [
      ...annotations.value,
      {
        id: Date.now(),
        span: draft.span,
        errorType: draft.errorType,
        correction: draft.correction.trim(),
      },
    ]
  }

  resetDraft(selectedSpan.value)
}

function editAnnotation(annotation: AnnotationRecord) {
  selectedAnnotationId.value = annotation.id
  selectedSpan.value = annotation.span
  draft.span = annotation.span
  draft.errorType = annotation.errorType
  draft.correction = annotation.correction
}

function deleteAnnotation(id: number) {
  annotations.value = annotations.value.filter((annotation) => annotation.id !== id)
  if (selectedAnnotationId.value === id) {
    resetDraft(selectedSpan.value)
  }
}

function submitJob() {
  screen.value = 'results'
}

function completePractice(item: PracticeItem) {
  if (practiceCompleted.value.includes(item.id)) {
    return
  }
  practiceCompleted.value = [...practiceCompleted.value, item.id]
  exerciseResult.value = `Refund processed: +¥${item.refund} for ${item.answer}`
}
</script>

<template>
  <div class="app-shell">
    <header class="topbar">
      <div>
        <p class="eyebrow">Bianxue Bureau of Text Rectification</p>
        <h1>Clerk Workstation</h1>
      </div>
      <div class="status-strip">
        <div class="status-card">
          <span>Rank</span>
          <strong>Assistant Editor II</strong>
        </div>
        <div class="status-card">
          <span>Funds</span>
          <strong>¥4,230</strong>
        </div>
        <div class="status-card">
          <span>Daily Streak</span>
          <strong>6 days</strong>
        </div>
      </div>
    </header>

    <nav class="nav-tabs">
      <button :class="{ active: screen === 'dashboard' }" @click="screen = 'dashboard'">Dispatch</button>
      <button :class="{ active: screen === 'workspace' }" @click="screen = 'workspace'">Active Job</button>
      <button :class="{ active: screen === 'results' }" @click="screen = 'results'">Review</button>
      <button :class="{ active: screen === 'leaderboard' }" @click="screen = 'leaderboard'">Leaderboard</button>
    </nav>

    <main v-if="screen === 'dashboard'" class="dashboard">
      <section class="hero-card paper">
        <div>
          <p class="eyebrow">Daily Contract</p>
          <h2>{{ dailyJob.title }}</h2>
          <p>
            Short-form bureaucratic triage with clear scoring, English-first feedback, and a bigger daily payout.
          </p>
          <div class="hero-meta">
            <span>{{ dailyJob.department }}</span>
            <span>{{ dailyJob.level }}</span>
            <span>Base ¥{{ dailyJob.payout }}</span>
          </div>
        </div>
        <button class="cta" @click="startJob(dailyJob.id)">Start Daily Contract</button>
      </section>

      <section class="dashboard-grid">
        <article class="panel">
          <div class="panel-head">
            <h3>Job Queue</h3>
            <span>3-5 min each</span>
          </div>
          <div class="job-list">
            <button
              v-for="job in jobs"
              :key="job.id"
              class="job-card"
              @click="startJob(job.id)"
            >
              <strong>{{ job.title }}</strong>
              <span>{{ job.department }}</span>
              <small>{{ job.difficulty }} · Base ¥{{ job.payout }}</small>
            </button>
          </div>
        </article>

        <article class="panel">
          <div class="panel-head">
            <h3>Department Ladder</h3>
            <span>Career progression</span>
          </div>
          <ul class="department-list">
            <li class="current">Elementary Assignment Review Dept.</li>
            <li class="current">Local Correspondence Office</li>
            <li>News Bureau</li>
            <li>Civil Documentation Dept.</li>
            <li>Academic Review Board</li>
          </ul>
          <div class="progress-block">
            <div class="progress-label">
              <span>Promotion meter</span>
              <strong>{{ rankProgress }}%</strong>
            </div>
            <div class="progress-bar">
              <div class="progress-fill" :style="{ width: `${rankProgress}%` }"></div>
            </div>
          </div>
        </article>

        <article class="panel">
          <div class="panel-head">
            <h3>Today’s Learning Angle</h3>
            <span>v1 feedback focus</span>
          </div>
          <div class="tag-row">
            <span>Wrong Character</span>
            <span>Word Order</span>
            <span>Measure Word</span>
          </div>
          <p class="panel-copy">
            This prototype emphasizes one-error-per-annotation, visible payouts, and quick review loops rather than backend scoring.
          </p>
        </article>
      </section>
    </main>

    <main v-else-if="screen === 'workspace'" class="workspace">
      <section class="panel scenario-panel">
        <p class="eyebrow">{{ currentJob.department }}</p>
        <h2>{{ currentJob.title }}</h2>
        <ul class="scenario-list">
          <li v-for="line in currentJob.scenario" :key="line">{{ line }}</li>
        </ul>
        <div class="memo">
          <strong>Clerk memo</strong>
          <p>{{ currentJob.memo }}</p>
        </div>
        <button class="ghost" @click="screen = 'dashboard'">Back to dispatch</button>
      </section>

      <section class="paper document-panel">
        <div class="panel-head">
          <h3>Document Passage</h3>
          <span>Click candidate spans to annotate</span>
        </div>
        <p class="document-text">{{ currentJob.text }}</p>
        <div class="span-picker">
          <button
            v-for="span in currentJob.selectableSpans"
            :key="span"
            :class="{ active: selectedSpan === span }"
            @click="chooseSpan(span)"
          >
            {{ span }}
          </button>
        </div>
      </section>

      <section class="panel editor-panel">
        <div class="panel-head">
          <h3>Annotation Editor</h3>
          <span>1 annotation = 1 mistake</span>
        </div>
        <label>
          Selected span
          <input :value="draft.span" readonly />
        </label>
        <label>
          Error type
          <select v-model="draft.errorType">
            <option>Wrong Character</option>
            <option>Wrong Word Choice</option>
            <option>Word Order Error</option>
            <option>Missing Word</option>
            <option>Extra Word</option>
            <option>Grammar Pattern Error</option>
            <option>Measure Word Error</option>
          </select>
        </label>
        <label>
          Correction
          <input v-model="draft.correction" placeholder="Enter the corrected text" />
        </label>
        <div class="editor-actions">
          <button class="ghost" @click="resetDraft(selectedSpan)">Reset</button>
          <button class="cta" @click="saveAnnotation">Save Annotation</button>
        </div>

        <div class="annotation-list">
          <div class="annotation-row" v-for="annotation in annotations" :key="annotation.id">
            <div>
              <strong>{{ annotation.span }}</strong>
              <p>{{ annotation.errorType }} → {{ annotation.correction }}</p>
            </div>
            <div class="row-actions">
              <button class="mini" @click="editAnnotation(annotation)">Edit</button>
              <button class="mini danger" @click="deleteAnnotation(annotation.id)">Delete</button>
            </div>
          </div>
        </div>

        <button class="submit" @click="submitJob">Submit For Grading</button>
      </section>
    </main>

    <main v-else-if="screen === 'results'" class="results">
      <section class="hero-card panel">
        <div>
          <p class="eyebrow">Moment of Truth</p>
          <h2>{{ currentJob.title }} graded</h2>
          <p>
            Your prototype score blends detection, type matching, correction accuracy, and practice refunds.
          </p>
        </div>
        <div class="score-badge">
          <strong>{{ accuracyScore }}%</strong>
          <span>Payout ¥{{ payoutScore }}</span>
        </div>
      </section>

      <section class="results-grid">
        <article class="panel">
          <div class="panel-head">
            <h3>Breakdown</h3>
            <span>Fair and readable</span>
          </div>
          <div class="metric-grid">
            <div class="metric-card">
              <span>Correctly found</span>
              <strong>{{ matchedErrors.length }}</strong>
            </div>
            <div class="metric-card">
              <span>Missed errors</span>
              <strong>{{ missedErrors.length }}</strong>
            </div>
            <div class="metric-card">
              <span>False positives</span>
              <strong>{{ falsePositives.length }}</strong>
            </div>
            <div class="metric-card">
              <span>Base fee</span>
              <strong>¥{{ currentJob.payout }}</strong>
            </div>
          </div>
        </article>

        <article class="panel">
          <div class="panel-head">
            <h3>Explanation Language</h3>
            <span>Player override</span>
          </div>
          <div class="toggle-row">
            <button :class="{ active: showReviewLanguage === 'en' }" @click="showReviewLanguage = 'en'">English</button>
            <button :class="{ active: showReviewLanguage === 'zh' }" @click="showReviewLanguage = 'zh'">中文</button>
          </div>
          <div class="explanation-list">
            <div class="explanation-card" v-for="error in currentJob.errors" :key="error.id">
              <strong>{{ error.incorrect }} → {{ error.correction }}</strong>
              <p>{{ showReviewLanguage === 'en' ? error.explanationEn : error.explanationZh }}</p>
            </div>
          </div>
        </article>
      </section>

      <section class="results-grid">
        <article class="panel">
          <div class="panel-head">
            <h3>Missed / incorrect</h3>
            <span>Coaching targets</span>
          </div>
          <div class="review-column">
            <div v-if="missedErrors.length === 0" class="empty-state">No missed errors in this mock run.</div>
            <div v-for="error in missedErrors" :key="error.id" class="review-card">
              <strong>{{ error.label }}</strong>
              <p>Expected {{ error.type }} on “{{ error.incorrect }}”.</p>
            </div>
            <div v-for="annotation in falsePositives" :key="annotation.id" class="review-card warning">
              <strong>False positive</strong>
              <p>{{ annotation.span }} was not accepted as submitted.</p>
            </div>
          </div>
        </article>

        <article class="panel">
          <div class="panel-head">
            <h3>Practice Refunds</h3>
            <span>Optional reinforcement</span>
          </div>
          <div class="practice-list">
            <div class="practice-card" v-for="item in practiceItems" :key="item.id">
              <div>
                <strong>{{ item.prompt }}</strong>
                <p>Correct answer: {{ item.answer }}</p>
              </div>
              <button class="mini" @click="completePractice(item)">
                {{ practiceCompleted.includes(item.id) ? 'Completed' : `Earn ¥${item.refund}` }}
              </button>
            </div>
          </div>
          <p class="refund-note">{{ exerciseResult || 'Complete practice to recover part of your payout loss.' }}</p>
          <div class="editor-actions">
            <button class="ghost" @click="screen = 'workspace'">Revise annotations</button>
            <button class="cta" @click="screen = 'leaderboard'">View standings</button>
          </div>
        </article>
      </section>
    </main>

    <main v-else class="leaderboard">
      <section class="hero-card panel">
        <div>
          <p class="eyebrow">Daily and Weekly Standings</p>
          <h2>Clerk leaderboard prototype</h2>
          <p>Compare payout totals and accuracy rates, then jump back into the queue for another short contract.</p>
        </div>
        <button class="cta" @click="screen = 'dashboard'">Take another job</button>
      </section>

      <section class="dashboard-grid">
        <article class="panel">
          <div class="panel-head">
            <h3>Daily Earnings</h3>
            <span>Today</span>
          </div>
          <div class="leaderboard-list">
            <div class="leader-row" v-for="entry in leaderboard" :key="entry.name">
              <strong>{{ entry.name }}</strong>
              <span>{{ entry.earnings }}</span>
              <small>{{ entry.accuracy }} accuracy</small>
            </div>
          </div>
        </article>

        <article class="panel">
          <div class="panel-head">
            <h3>Career Snapshot</h3>
            <span>Persistent progression</span>
          </div>
          <div class="snapshot-grid">
            <div class="metric-card">
              <span>Jobs completed</span>
              <strong>42</strong>
            </div>
            <div class="metric-card">
              <span>Average accuracy</span>
              <strong>88%</strong>
            </div>
            <div class="metric-card">
              <span>Department unlocked</span>
              <strong>2 / 5</strong>
            </div>
            <div class="metric-card">
              <span>Best daily payout</span>
              <strong>¥278</strong>
            </div>
          </div>
        </article>
      </section>
    </main>
  </div>
</template>

<style scoped>
:global(body) {
  margin: 0;
  font-family: Georgia, 'Times New Roman', serif;
  background:
    radial-gradient(circle at top left, rgba(145, 45, 35, 0.18), transparent 28%),
    linear-gradient(180deg, #d8c6a4 0%, #c4ae87 100%);
  color: #2f2418;
}

:global(*) {
  box-sizing: border-box;
}

.app-shell {
  min-height: 100vh;
  padding: 24px;
  background:
    linear-gradient(135deg, rgba(74, 52, 31, 0.18), transparent 35%),
    linear-gradient(0deg, rgba(255, 248, 232, 0.4), rgba(255, 248, 232, 0.4));
}

.topbar,
.hero-card,
.panel,
.paper {
  border: 1px solid rgba(72, 48, 29, 0.18);
  box-shadow: 0 18px 40px rgba(76, 47, 29, 0.12);
}

.topbar {
  display: flex;
  justify-content: space-between;
  gap: 16px;
  align-items: center;
  padding: 20px 24px;
  border-radius: 20px;
  background: rgba(248, 241, 228, 0.92);
}

.eyebrow {
  margin: 0 0 6px;
  text-transform: uppercase;
  letter-spacing: 0.12em;
  font-size: 0.72rem;
  color: #8a3d2f;
}

h1,
h2,
h3,
p {
  margin-top: 0;
}

h1 {
  margin-bottom: 0;
  font-size: clamp(2rem, 4vw, 3rem);
}

.status-strip {
  display: grid;
  grid-template-columns: repeat(3, minmax(120px, 1fr));
  gap: 12px;
  width: min(100%, 460px);
}

.status-card,
.metric-card {
  padding: 14px;
  border-radius: 14px;
  background: #f3ebdc;
}

.status-card span,
.metric-card span,
.panel-head span,
.hero-meta span,
.leader-row small {
  display: block;
  font-size: 0.82rem;
  color: #76563f;
}

.status-card strong,
.metric-card strong {
  display: block;
  margin-top: 4px;
  font-size: 1.05rem;
}

.nav-tabs {
  display: flex;
  gap: 10px;
  margin: 20px 0;
}

button {
  border: 0;
  border-radius: 999px;
  padding: 12px 18px;
  font: inherit;
  cursor: pointer;
  transition:
    transform 140ms ease,
    background 140ms ease,
    color 140ms ease;
}

button:hover {
  transform: translateY(-1px);
}

.nav-tabs button,
.ghost,
.mini {
  background: rgba(248, 241, 228, 0.88);
  color: #4d3323;
}

.nav-tabs .active,
.toggle-row .active,
.span-picker .active {
  background: #8f3327;
  color: #fff7ed;
}

.cta,
.submit {
  background: #8f3327;
  color: #fff7ed;
}

.submit {
  width: 100%;
  margin-top: 16px;
}

.danger {
  color: #8f3327;
}

.dashboard,
.results,
.leaderboard {
  display: grid;
  gap: 20px;
}

.workspace {
  display: grid;
  grid-template-columns: 0.9fr 1.3fr 1fr;
  gap: 20px;
}

.hero-card,
.panel,
.paper {
  border-radius: 22px;
  padding: 22px;
  background: rgba(249, 244, 234, 0.92);
}

.hero-card {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
}

.hero-meta,
.tag-row,
.toggle-row,
.editor-actions,
.row-actions,
.span-picker {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.hero-meta span,
.tag-row span {
  padding: 8px 12px;
  border-radius: 999px;
  background: #efe3cf;
}

.dashboard-grid,
.results-grid {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 20px;
}

.results-grid {
  grid-template-columns: repeat(2, minmax(0, 1fr));
}

.panel-head {
  display: flex;
  justify-content: space-between;
  gap: 12px;
  align-items: baseline;
  margin-bottom: 16px;
}

.job-list,
.annotation-list,
.practice-list,
.leaderboard-list,
.review-column,
.explanation-list {
  display: grid;
  gap: 12px;
}

.job-card,
.annotation-row,
.practice-card,
.leader-row,
.review-card,
.explanation-card {
  width: 100%;
  text-align: left;
  border-radius: 16px;
  padding: 16px;
  background: #f5ede0;
}

.annotation-row,
.practice-card,
.leader-row {
  display: flex;
  justify-content: space-between;
  gap: 12px;
  align-items: center;
}

.department-list,
.scenario-list {
  margin: 0;
  padding-left: 18px;
  display: grid;
  gap: 8px;
}

.department-list .current {
  color: #8f3327;
  font-weight: 700;
}

.progress-block,
.memo {
  margin-top: 18px;
}

.progress-label {
  display: flex;
  justify-content: space-between;
  margin-bottom: 8px;
}

.progress-bar {
  height: 14px;
  border-radius: 999px;
  background: #e5d5bc;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  border-radius: inherit;
  background: linear-gradient(90deg, #8f3327, #be6d34);
}

.document-text {
  padding: 22px;
  min-height: 220px;
  border-radius: 18px;
  line-height: 1.9;
  font-size: 1.15rem;
  background:
    linear-gradient(transparent 95%, rgba(102, 68, 41, 0.18) 95%),
    #fff8ec;
  background-size: 100% 2.1rem;
}

label {
  display: grid;
  gap: 8px;
  margin-bottom: 14px;
  font-size: 0.95rem;
}

input,
select {
  width: 100%;
  padding: 12px 14px;
  border-radius: 14px;
  border: 1px solid rgba(86, 60, 39, 0.2);
  background: #fff8ec;
  font: inherit;
}

.score-badge {
  min-width: 180px;
  padding: 20px;
  border-radius: 18px;
  text-align: center;
  background: #8f3327;
  color: #fff7ed;
}

.score-badge strong {
  display: block;
  font-size: 2.2rem;
}

.metric-grid,
.snapshot-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
}

.review-card.warning {
  border: 1px dashed #8f3327;
}

.empty-state,
.refund-note,
.panel-copy {
  color: #654a37;
}

@media (max-width: 1100px) {
  .workspace,
  .dashboard-grid,
  .results-grid {
    grid-template-columns: 1fr;
  }

  .topbar,
  .hero-card,
  .annotation-row,
  .practice-card,
  .leader-row {
    align-items: flex-start;
    flex-direction: column;
  }

  .status-strip {
    width: 100%;
  }
}
</style>
