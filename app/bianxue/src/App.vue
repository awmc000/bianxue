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

const leaderboard = [
  { name: 'Archivist Maple', earnings: '¥2,480', accuracy: '96%' },
  { name: 'You', earnings: '¥1,920', accuracy: '89%' },
  { name: 'Deputy Lin', earnings: '¥1,870', accuracy: '87%' },
  { name: 'Clerk No. 7', earnings: '¥1,715', accuracy: '84%' },
]

const departments = [
  'Elementary Assignment Review Dept.',
  'Local Correspondence Office',
  'News Bureau',
  'Civil Documentation Dept.',
  'Academic Review Board',
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

function resetDraft(span?: string) {
  const nextSpan = span ?? currentJob.value.selectableSpans[0]!
  selectedSpan.value = nextSpan
  draft.span = nextSpan
  draft.errorType = 'Wrong Character'
  draft.correction = ''
  selectedAnnotationId.value = null
}

function startJob(jobId: string) {
  const nextJob = jobs.find((job) => job.id === jobId) ?? initialJob
  currentJobId.value = nextJob.id
  annotations.value = [
    {
      id: 1,
      span: nextJob.selectableSpans[0]!,
      errorType: 'Wrong Character',
      correction: nextJob.id === 'daily-204' ? '会议' : '',
    },
  ]
  practiceCompleted.value = []
  exerciseResult.value = ''
  showReviewLanguage.value = 'en'
  resetDraft(nextJob.selectableSpans[0]!)
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
  <div class="board">
    <div class="cloud cloud-a"></div>
    <div class="cloud cloud-b"></div>

    <header class="paper hero-sheet torn">
      <span class="pin pin-blue">📌</span>
      <span class="pin pin-red">●</span>
      <div class="hero-copy">
        <p class="kicker">Ministry of Everyday Corrections</p>
        <h1>Clerk Workstation</h1>
        <p class="hero-note">
          Short Chinese editing jobs, satirical office flavor, and quick score-and-review loops.
        </p>
      </div>
      <div class="hero-side">
        <div class="hero-stamp">Daily Pay<br />¥{{ dailyJob.payout }}</div>
        <div class="status-ribbon">
          <span>Rank: Assistant Editor II</span>
          <span>Funds: ¥4,230</span>
          <span>Streak: 6 days</span>
        </div>
      </div>
    </header>

    <nav class="paper nav-strip torn">
      <button :class="{ active: screen === 'dashboard' }" @click="screen = 'dashboard'">Dispatch</button>
      <button :class="{ active: screen === 'workspace' }" @click="screen = 'workspace'">Active Job</button>
      <button :class="{ active: screen === 'results' }" @click="screen = 'results'">Review</button>
      <button :class="{ active: screen === 'leaderboard' }" @click="screen = 'leaderboard'">Leaderboard</button>
      <div class="search-strip">Queue search</div>
    </nav>

    <main v-if="screen === 'dashboard'" class="page">
      <section class="card-grid">
        <article class="paper feature-card torn">
          <span class="pin pin-blue">●</span>
          <h2>Daily Contract</h2>
          <p class="card-subtitle">{{ dailyJob.title }} · {{ dailyJob.department }}</p>
          <p>
            The high-priority assignment for today. Better payout, fixed job length, and a clear review path.
          </p>
          <div class="mini-tags">
            <span>{{ dailyJob.level }}</span>
            <span>{{ dailyJob.difficulty }}</span>
            <span>Base ¥{{ dailyJob.payout }}</span>
          </div>
          <button class="ink-button" @click="startJob(dailyJob.id)">Start Daily Contract</button>
        </article>

        <article class="paper feature-card torn accent-red">
          <span class="pin pin-red">📌</span>
          <h2>Department Ladder</h2>
          <ul class="department-list">
            <li v-for="department in departments" :key="department" :class="{ current: department.includes('Office') || department.includes('Review Dept.') }">
              {{ department }}
            </li>
          </ul>
          <div class="progress-note">
            <span>Promotion meter</span>
            <strong>{{ rankProgress }}%</strong>
          </div>
          <div class="progress-track">
            <div class="progress-fill" :style="{ width: `${rankProgress}%` }"></div>
          </div>
        </article>

        <article class="paper feature-card torn accent-blue">
          <span class="pin pin-blue">📌</span>
          <h2>Practice Refunds</h2>
          <p>Miss a correction, then earn money back with short drills after grading.</p>
          <div class="mini-tags">
            <span>Fill in the blank</span>
            <span>Multiple choice</span>
            <span>Bilingual review</span>
          </div>
          <button class="ghost-button" @click="screen = 'results'">Preview Review Flow</button>
        </article>
      </section>

      <section class="bottom-layout">
        <article class="paper wide-sheet torn">
          <span class="pin pin-blue">●</span>
          <div class="section-head">
            <h2>Incoming Job Queue</h2>
            <span>3-5 minute tasks</span>
          </div>
          <div class="queue-list">
            <button v-for="job in jobs" :key="job.id" class="queue-row" @click="startJob(job.id)">
              <div>
                <strong>{{ job.title }}</strong>
                <p>{{ job.department }} · {{ job.level }}</p>
              </div>
              <div class="queue-meta">
                <span>{{ job.difficulty }}</span>
                <span>¥{{ job.payout }}</span>
              </div>
            </button>
          </div>
        </article>

        <aside class="side-column">
          <article class="paper memo-card torn small-note">
            <span class="pin pin-red">●</span>
            <h3>Today’s Focus</h3>
            <p>Wrong Character, Word Order, and Measure Word issues are appearing often in the queue.</p>
          </article>

          <article class="paper memo-card torn small-note angle">
            <span class="pin pin-blue">📌</span>
            <h3>Office Notice</h3>
            <p>This pass is visual only. Annotation selection is still mocked with clickable spans.</p>
          </article>
        </aside>
      </section>
    </main>

    <main v-else-if="screen === 'workspace'" class="page">
      <section class="workspace-layout">
        <article class="paper board-column torn">
          <span class="pin pin-red">●</span>
          <div class="section-head">
            <h2>{{ currentJob.title }}</h2>
            <span>{{ currentJob.department }}</span>
          </div>
          <p class="card-subtitle">{{ currentJob.level }} · {{ currentJob.difficulty }} · Base ¥{{ currentJob.payout }}</p>
          <ul class="scenario-list">
            <li v-for="line in currentJob.scenario" :key="line">{{ line }}</li>
          </ul>
          <div class="memo-box">
            <strong>Clerk memo</strong>
            <p>{{ currentJob.memo }}</p>
          </div>
          <button class="ghost-button" @click="screen = 'dashboard'">Back to dispatch</button>
        </article>

        <article class="paper document-sheet torn">
          <span class="pin pin-blue">📌</span>
          <div class="section-head">
            <h2>Document Passage</h2>
            <span>Select candidate spans</span>
          </div>
          <p class="document-text">{{ currentJob.text }}</p>
          <div class="span-palette">
            <button
              v-for="span in currentJob.selectableSpans"
              :key="span"
              :class="{ active: selectedSpan === span }"
              @click="chooseSpan(span)"
            >
              {{ span }}
            </button>
          </div>
        </article>

        <article class="paper board-column torn angle">
          <span class="pin pin-red">📌</span>
          <div class="section-head">
            <h2>Annotation Editor</h2>
            <span>1 note per mistake</span>
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
          <div class="action-row">
            <button class="ghost-button" @click="resetDraft(selectedSpan)">Reset</button>
            <button class="ink-button" @click="saveAnnotation">Save Annotation</button>
          </div>
          <div class="annotation-stack">
            <div class="annotation-note" v-for="annotation in annotations" :key="annotation.id">
              <div>
                <strong>{{ annotation.span }}</strong>
                <p>{{ annotation.errorType }} → {{ annotation.correction }}</p>
              </div>
              <div class="tiny-actions">
                <button class="tiny-button" @click="editAnnotation(annotation)">Edit</button>
                <button class="tiny-button alert" @click="deleteAnnotation(annotation.id)">Delete</button>
              </div>
            </div>
          </div>
          <button class="ink-button full" @click="submitJob">Submit For Grading</button>
        </article>
      </section>
    </main>

    <main v-else-if="screen === 'results'" class="page">
      <section class="bottom-layout">
        <article class="paper wide-sheet torn">
          <span class="pin pin-blue">📌</span>
          <div class="section-head">
            <h2>{{ currentJob.title }} graded</h2>
            <span>Moment of truth</span>
          </div>
          <div class="results-summary">
            <div class="score-paper">
              <strong>{{ accuracyScore }}%</strong>
              <span>Accuracy</span>
            </div>
            <div class="score-paper">
              <strong>¥{{ payoutScore }}</strong>
              <span>Final payout</span>
            </div>
            <div class="score-paper">
              <strong>{{ matchedErrors.length }}/{{ currentJob.errors.length }}</strong>
              <span>Accepted annotations</span>
            </div>
          </div>
          <div class="review-grid">
            <div class="review-panel">
              <h3>Breakdown</h3>
              <p>Missed errors: {{ missedErrors.length }}</p>
              <p>False positives: {{ falsePositives.length }}</p>
              <p>Base fee: ¥{{ currentJob.payout }}</p>
            </div>
            <div class="review-panel">
              <h3>Explanation Language</h3>
              <div class="language-toggle">
                <button :class="{ active: showReviewLanguage === 'en' }" @click="showReviewLanguage = 'en'">English</button>
                <button :class="{ active: showReviewLanguage === 'zh' }" @click="showReviewLanguage = 'zh'">中文</button>
              </div>
            </div>
          </div>
          <div class="explanation-stack">
            <div class="review-note" v-for="error in currentJob.errors" :key="error.id">
              <strong>{{ error.incorrect }} → {{ error.correction }}</strong>
              <p>{{ showReviewLanguage === 'en' ? error.explanationEn : error.explanationZh }}</p>
            </div>
          </div>
        </article>

        <aside class="side-column">
          <article class="paper memo-card torn small-note">
            <span class="pin pin-red">●</span>
            <h3>Missed / incorrect</h3>
            <div v-if="missedErrors.length === 0" class="soft-copy">No missed errors in this mock run.</div>
            <div v-for="error in missedErrors" :key="error.id" class="review-chip">
              {{ error.label }}
            </div>
            <div v-for="annotation in falsePositives" :key="annotation.id" class="review-chip warning">
              False positive: {{ annotation.span }}
            </div>
          </article>

          <article class="paper memo-card torn small-note angle">
            <span class="pin pin-blue">📌</span>
            <h3>Practice Refunds</h3>
            <div class="practice-stack">
              <div class="practice-row" v-for="item in practiceItems" :key="item.id">
                <div>
                  <strong>{{ item.prompt }}</strong>
                  <p>{{ item.answer }}</p>
                </div>
                <button class="tiny-button" @click="completePractice(item)">
                  {{ practiceCompleted.includes(item.id) ? 'Done' : `+¥${item.refund}` }}
                </button>
              </div>
            </div>
            <p class="soft-copy">{{ exerciseResult || 'Complete drills to recover part of the lost payout.' }}</p>
            <div class="action-row">
              <button class="ghost-button" @click="screen = 'workspace'">Revise</button>
              <button class="ink-button" @click="screen = 'leaderboard'">Standings</button>
            </div>
          </article>
        </aside>
      </section>
    </main>

    <main v-else class="page">
      <section class="bottom-layout">
        <article class="paper wide-sheet torn">
          <span class="pin pin-red">📌</span>
          <div class="section-head">
            <h2>Clerk Leaderboard</h2>
            <span>Daily and weekly standings</span>
          </div>
          <div class="leaderboard-sheet">
            <div class="leader-row" v-for="entry in leaderboard" :key="entry.name">
              <strong>{{ entry.name }}</strong>
              <span>{{ entry.earnings }}</span>
              <small>{{ entry.accuracy }} accuracy</small>
            </div>
          </div>
        </article>

        <aside class="side-column">
          <article class="paper memo-card torn small-note">
            <span class="pin pin-blue">●</span>
            <h3>Career Snapshot</h3>
            <div class="snapshot-grid">
              <div class="snapshot-tile">
                <strong>42</strong>
                <span>Jobs completed</span>
              </div>
              <div class="snapshot-tile">
                <strong>88%</strong>
                <span>Average accuracy</span>
              </div>
              <div class="snapshot-tile">
                <strong>2 / 5</strong>
                <span>Departments unlocked</span>
              </div>
              <div class="snapshot-tile">
                <strong>¥278</strong>
                <span>Best daily payout</span>
              </div>
            </div>
          </article>

          <article class="paper memo-card torn small-note angle">
            <span class="pin pin-red">●</span>
            <h3>Next move</h3>
            <p>Jump back into dispatch for another short contract and a new set of mistakes to catch.</p>
            <button class="ink-button" @click="screen = 'dashboard'">Take another job</button>
          </article>
        </aside>
      </section>
    </main>
  </div>
</template>

<style scoped>
:global(body) {
  margin: 0;
  font-family: 'Trebuchet MS', 'Segoe UI', sans-serif;
  color: #36506f;
  background:
    radial-gradient(circle at 10% 10%, rgba(255, 255, 255, 0.35), transparent 8rem),
    radial-gradient(circle at 80% 4%, rgba(255, 255, 255, 0.28), transparent 9rem),
    linear-gradient(180deg, #8cc8f2 0%, #cfe7fb 55%, #dfeefb 100%);
}

:global(*) {
  box-sizing: border-box;
}

.board {
  position: relative;
  min-height: 100vh;
  padding: 2.5rem 1.25rem 3rem;
  overflow: hidden;
}

.cloud {
  position: absolute;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.22);
  filter: blur(2px);
}

.cloud::before,
.cloud::after {
  content: '';
  position: absolute;
  border-radius: 999px;
  background: inherit;
}

.cloud-a {
  top: 1.25rem;
  left: 2rem;
  width: 9rem;
  height: 3.5rem;
}

.cloud-a::before {
  width: 4.5rem;
  height: 4.5rem;
  left: 1rem;
  top: -1.25rem;
}

.cloud-a::after {
  width: 5rem;
  height: 4rem;
  right: 0.5rem;
  top: -0.75rem;
}

.cloud-b {
  top: 0.25rem;
  right: 4rem;
  width: 10rem;
  height: 4rem;
}

.cloud-b::before {
  width: 5rem;
  height: 5rem;
  left: 1rem;
  top: -1.75rem;
}

.cloud-b::after {
  width: 4rem;
  height: 4rem;
  right: 1rem;
  top: -0.75rem;
}

.page,
.card-grid,
.bottom-layout,
.workspace-layout,
.side-column,
.annotation-stack,
.practice-stack,
.explanation-stack,
.queue-list,
.leaderboard-sheet,
.snapshot-grid {
  display: grid;
  gap: 1.25rem;
}

.paper {
  position: relative;
  background:
    linear-gradient(135deg, rgba(255, 255, 255, 0.95), rgba(248, 252, 255, 0.9)),
    repeating-linear-gradient(
      -35deg,
      rgba(101, 157, 210, 0.03),
      rgba(101, 157, 210, 0.03) 14px,
      rgba(255, 255, 255, 0.05) 14px,
      rgba(255, 255, 255, 0.05) 28px
    );
  border-radius: 0.9rem;
  box-shadow:
    0 14px 28px rgba(69, 114, 156, 0.15),
    0 2px 0 rgba(255, 255, 255, 0.8) inset;
  padding: 1.5rem;
}

.torn::before,
.torn::after {
  content: '';
  position: absolute;
  left: 0;
  width: 100%;
  height: 12px;
  background:
    radial-gradient(circle at 8px 0, transparent 8px, rgba(255, 255, 255, 0.9) 8px) repeat-x;
  background-size: 22px 12px;
  pointer-events: none;
}

.torn::before {
  top: -6px;
}

.torn::after {
  bottom: -6px;
  transform: rotate(180deg);
}

.pin {
  position: absolute;
  top: -0.7rem;
  font-size: 1.15rem;
  text-shadow: 0 6px 8px rgba(40, 70, 103, 0.22);
}

.pin-blue {
  left: 1rem;
  color: #3496eb;
}

.pin-red {
  right: 1.2rem;
  color: #ff655f;
}

.hero-sheet {
  display: grid;
  grid-template-columns: 1.5fr 1fr;
  gap: 1.5rem;
  max-width: 1180px;
  margin: 0 auto 1rem;
  padding: 2rem;
}

.hero-copy h1,
.section-head h2,
.feature-card h2,
.memo-card h3 {
  margin: 0;
  font-family: 'Brush Script MT', 'Segoe Print', cursive;
  font-weight: 400;
  color: #3a5d92;
}

.hero-copy h1 {
  font-size: clamp(2.4rem, 5vw, 4rem);
}

.hero-copy .kicker {
  margin: 0 0 0.35rem;
  letter-spacing: 0.14em;
  text-transform: uppercase;
  color: #ef5a55;
  font-size: 0.75rem;
}

.hero-note,
.card-subtitle,
.soft-copy,
.memo-box p,
.queue-row p,
.review-note p,
.practice-row p,
.annotation-note p {
  color: #6d87a2;
}

.hero-side {
  display: grid;
  gap: 1rem;
  align-content: center;
}

.hero-stamp {
  justify-self: end;
  width: 8.5rem;
  padding: 1rem;
  border: 3px dashed rgba(239, 90, 85, 0.45);
  border-radius: 0.8rem;
  text-align: center;
  color: #ef5a55;
  font-weight: 700;
  transform: rotate(-4deg);
}

.status-ribbon {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
  justify-content: flex-end;
}

.status-ribbon span,
.mini-tags span,
.queue-meta span,
.review-chip {
  padding: 0.4rem 0.75rem;
  border-radius: 999px;
  background: rgba(95, 180, 239, 0.12);
  color: #3b6b98;
  font-size: 0.9rem;
}

.nav-strip {
  max-width: 1180px;
  margin: 0 auto 1.5rem;
  display: flex;
  gap: 0.75rem;
  align-items: center;
  flex-wrap: wrap;
  padding: 1rem 1.25rem;
}

.nav-strip button,
.ink-button,
.ghost-button,
.tiny-button,
.language-toggle button,
.span-palette button {
  border: 0;
  border-radius: 999px;
  font: inherit;
  cursor: pointer;
  transition:
    transform 140ms ease,
    background 140ms ease,
    color 140ms ease;
}

.nav-strip button:hover,
.ink-button:hover,
.ghost-button:hover,
.tiny-button:hover,
.language-toggle button:hover,
.span-palette button:hover {
  transform: translateY(-1px);
}

.nav-strip button,
.ghost-button,
.language-toggle button,
.span-palette button,
.tiny-button {
  padding: 0.7rem 1rem;
  background: rgba(101, 170, 223, 0.12);
  color: #3d6892;
}

.nav-strip button.active,
.language-toggle button.active,
.span-palette button.active {
  background: #4aa4ec;
  color: white;
}

.search-strip {
  margin-left: auto;
  min-width: 9rem;
  padding: 0.7rem 1rem;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.72);
  color: #98adbf;
  text-align: center;
}

.card-grid {
  grid-template-columns: repeat(3, minmax(0, 1fr));
  max-width: 1180px;
  margin: 0 auto;
}

.feature-card,
.memo-card {
  min-height: 16rem;
}

.accent-red h2,
.accent-red .pin {
  color: #ef5a55;
}

.accent-blue h2 {
  color: #3a9ae0;
}

.mini-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin: 1rem 0 1.25rem;
}

.ink-button {
  padding: 0.8rem 1.2rem;
  background: #ef5a55;
  color: white;
}

.ghost-button {
  padding: 0.8rem 1.2rem;
}

.bottom-layout {
  grid-template-columns: 2fr 0.95fr;
  max-width: 1180px;
  margin: 1.5rem auto 0;
  align-items: start;
}

.wide-sheet {
  min-height: 20rem;
}

.small-note {
  min-height: 10rem;
}

.angle {
  transform: rotate(-1.2deg);
}

.section-head {
  display: flex;
  justify-content: space-between;
  gap: 1rem;
  align-items: start;
  margin-bottom: 1rem;
}

.section-head span {
  color: #88a6c0;
  font-size: 0.95rem;
}

.queue-row,
.leader-row,
.practice-row,
.annotation-note {
  display: flex;
  justify-content: space-between;
  gap: 1rem;
  align-items: center;
}

.queue-row {
  width: 100%;
  padding: 1rem 1.1rem;
  border: 0;
  border-radius: 1rem;
  background: rgba(78, 161, 229, 0.08);
  text-align: left;
  cursor: pointer;
}

.queue-row strong,
.leader-row strong,
.annotation-note strong,
.practice-row strong,
.review-note strong,
.score-paper strong,
.snapshot-tile strong {
  color: #3d5f8a;
}

.queue-row p,
.review-note p,
.practice-row p,
.annotation-note p {
  margin: 0.25rem 0 0;
}

.department-list,
.scenario-list {
  margin: 0;
  padding-left: 1.15rem;
  color: #54779a;
}

.department-list li + li,
.scenario-list li + li {
  margin-top: 0.45rem;
}

.department-list .current {
  color: #ef5a55;
}

.progress-note {
  display: flex;
  justify-content: space-between;
  margin: 1rem 0 0.55rem;
  color: #6d87a2;
}

.progress-track {
  height: 0.8rem;
  border-radius: 999px;
  background: rgba(74, 164, 236, 0.14);
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  border-radius: inherit;
  background: linear-gradient(90deg, #ef5a55, #4aa4ec);
}

.workspace-layout {
  grid-template-columns: 0.95fr 1.35fr 1fr;
  max-width: 1180px;
  margin: 0 auto;
  align-items: start;
}

.board-column,
.document-sheet {
  min-height: 33rem;
}

.memo-box,
.review-panel,
.score-paper,
.snapshot-tile {
  padding: 1rem;
  border-radius: 1rem;
  background: rgba(80, 173, 240, 0.08);
}

.document-text {
  min-height: 16rem;
  padding: 1.4rem;
  border-radius: 1rem;
  background:
    linear-gradient(transparent calc(100% - 1px), rgba(95, 164, 221, 0.18) 0) 0 0 / 100% 2.1rem,
    rgba(255, 255, 255, 0.9);
  color: #355575;
  font-size: 1.2rem;
  line-height: 1.75;
}

.span-palette {
  display: flex;
  flex-wrap: wrap;
  gap: 0.7rem;
  margin-top: 1rem;
}

label {
  display: grid;
  gap: 0.45rem;
  margin-bottom: 0.9rem;
  color: #6281a0;
}

input,
select {
  width: 100%;
  padding: 0.85rem 0.95rem;
  border: 1px solid rgba(92, 152, 201, 0.2);
  border-radius: 1rem;
  background: rgba(255, 255, 255, 0.86);
  color: #36506f;
  font: inherit;
}

.action-row,
.tiny-actions,
.language-toggle,
.results-summary,
.review-grid {
  display: flex;
  gap: 0.75rem;
  flex-wrap: wrap;
}

.annotation-stack {
  margin: 1rem 0;
}

.annotation-note,
.review-note,
.practice-row,
.leader-row {
  padding: 0.95rem 1rem;
  border-radius: 1rem;
  background: rgba(255, 255, 255, 0.76);
}

.tiny-button {
  padding: 0.5rem 0.8rem;
  font-size: 0.9rem;
}

.tiny-button.alert,
.review-chip.warning {
  color: #ef5a55;
}

.full {
  width: 100%;
}

.results-summary {
  margin: 1rem 0 1.25rem;
}

.score-paper {
  min-width: 10rem;
  text-align: center;
}

.score-paper strong {
  display: block;
  font-size: 2rem;
}

.review-grid {
  margin-bottom: 1rem;
}

.review-panel {
  flex: 1 1 15rem;
}

.practice-stack,
.explanation-stack {
  margin-top: 0.85rem;
}

.review-chip {
  display: inline-flex;
  margin: 0.35rem 0.35rem 0 0;
}

.leaderboard-sheet {
  margin-top: 1rem;
}

.snapshot-grid {
  grid-template-columns: repeat(2, minmax(0, 1fr));
}

.snapshot-tile {
  text-align: center;
}

.snapshot-tile span {
  display: block;
  margin-top: 0.35rem;
  color: #6d87a2;
}

@media (max-width: 980px) {
  .hero-sheet,
  .card-grid,
  .bottom-layout,
  .workspace-layout {
    grid-template-columns: 1fr;
  }

  .hero-side,
  .status-ribbon {
    justify-content: start;
  }

  .search-strip {
    margin-left: 0;
    width: 100%;
  }
}
</style>
