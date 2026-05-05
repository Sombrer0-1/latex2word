<template>
  <div>
    <h1>LaTeX → MathML Converter</h1>

    <div class="mode-tabs">
      <button
        :class="{ active: mode === 'single' }"
        @click="switchMode('single')"
      >Single Formula</button>
      <button
        :class="{ active: mode === 'longtext' }"
        @click="switchMode('longtext')"
      >Long Text</button>
    </div>

    <!-- Single Formula Mode -->
    <template v-if="mode === 'single'">
      <div class="input-section">
        <label class="section-label">LaTeX Input</label>
        <textarea
          v-model="latexInput"
          placeholder="Enter LaTeX formula, e.g. \frac{a}{b} or E = mc^2"
          @input="convertSingle"
        ></textarea>

        <div class="examples">
          <button @click="loadExample(ex)" v-for="ex in examples" :key="ex.label">
            {{ ex.label }}
          </button>
        </div>

        <div class="toolbar">
          <button class="btn-primary" @click="convertSingle">Convert</button>
          <button class="btn-secondary" @click="clearSingle">Clear</button>
        </div>
      </div>

      <div class="output-section">
        <label class="section-label">Rendered Preview</label>
        <div
          v-if="!error && mathmlHtml"
          class="preview-area mathml-output"
          v-html="mathmlHtml"
        ></div>
        <div v-else-if="error" class="error-msg">{{ error }}</div>
        <div v-else class="preview-area empty">Preview will appear here</div>

        <div class="toolbar" v-if="mathmlHtml">
          <button class="btn-copy" @click="copyText(mathmlOutput)" v-if="!copied">
            Copy MathML
          </button>
          <span v-else class="copied-toast">{{ copyMessage }}</span>
          <button class="btn-secondary" @click="showRaw = !showRaw">
            {{ showRaw ? 'Hide' : 'Show' }} Raw MathML
          </button>
        </div>

        <div class="raw-output" v-if="showRaw && mathmlOutput">
          <pre>{{ mathmlOutput }}</pre>
        </div>
      </div>
    </template>

    <!-- Long Text Mode -->
    <template v-if="mode === 'longtext'">
      <div class="input-section">
        <label class="section-label">Mixed Text Input</label>
        <textarea
          v-model="longTextInput"
          class="long-textarea"
          placeholder="Paste your mixed text here...&#10;&#10;Example: The equation $E=mc^2$ shows energy-mass equivalence.&#10;Display math like \begin{equation} a^2+b^2=c^2 \end{equation} is also supported."
          @input="convertLongText"
        ></textarea>

        <div class="toolbar options-bar">
          <label class="checkbox-label">
            <input type="checkbox" v-model="removeEmptyLines" @change="convertLongText" />
            <span>去除空行</span>
          </label>
        </div>

        <div class="toolbar">
          <button class="btn-primary" @click="convertLongText">Convert</button>
          <button class="btn-secondary" @click="loadLongExample">Load Example</button>
          <button class="btn-secondary" @click="clearLongText">Clear</button>
        </div>

        <div class="detected-info" v-if="detectedCount > 0">
          Detected <strong>{{ detectedCount }}</strong> formula(s) in the text
        </div>
      </div>

      <div class="output-section">
        <label class="section-label">Rendered Result</label>
        <div
          v-if="!error && renderedHtml"
          class="preview-area long-text-output"
          v-html="renderedHtml"
        ></div>
        <div v-else-if="error" class="error-msg">{{ error }}</div>
        <div v-else class="preview-area empty">Rendered text will appear here</div>

        <div class="toolbar" v-if="renderedHtml">
          <button class="btn-copy" @click="copyAll" v-if="!copied">
            Copy All
          </button>
          <span v-else class="copied-toast">{{ copyMessage }}</span>
        </div>
      </div>
    </template>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import temml from 'temml'
import { mml2omml } from 'mathml2omml'

const mode = ref('single')

// --- Single Formula Mode ---
const latexInput = ref('')
const mathmlHtml = ref('')
const mathmlOutput = ref('')
const error = ref('')
const copied = ref(false)
const copyMessage = ref('Copied!')
const showRaw = ref(false)

const examples = [
  { label: 'Fraction', latex: '\\frac{a}{b}' },
  { label: 'Sum', latex: '\\sum_{i=1}^{n} x_i' },
  { label: 'Integral', latex: '\\int_{0}^{\\infty} e^{-x^2} dx' },
  { label: 'Matrix', latex: '\\begin{pmatrix} a & b \\\\ c & d \\end{pmatrix}' },
  { label: 'Equation', latex: 'E = mc^2' },
  { label: 'Limit', latex: '\\lim_{x \\to \\infty} \\frac{1}{x} = 0' },
]

function loadExample(ex) {
  latexInput.value = ex.latex
  convertSingle()
}

function convertSingle() {
  error.value = ''
  mathmlHtml.value = ''
  mathmlOutput.value = ''
  copied.value = false

  if (!latexInput.value.trim()) return

  try {
    const html = temml.renderToString(latexInput.value, { displayMode: true, xml: true })
    mathmlHtml.value = html
    mathmlOutput.value = formatXml(html)
  } catch (e) {
    error.value = 'Conversion error: ' + e.message
  }
}

function clearSingle() {
  latexInput.value = ''
  mathmlHtml.value = ''
  mathmlOutput.value = ''
  error.value = ''
  copied.value = false
  showRaw.value = false
}

// --- Long Text Mode ---
const longTextInput = ref('')
const renderedHtml = ref('')
const renderedText = ref('')
const renderedClipboardHtml = ref('')
const detectedCount = ref(0)
const removeEmptyLines = ref(false)

const longExample = `时间序列是按时间顺序排列的观测集合。单变量时间序列可表示为
\\begin{equation}
\\{x_t\\}_{t=1}^{T},\\qquad x_t\\in\\mathbb{R},
\\end{equation}
其中 $t$ 为离散时间索引，$T$ 为序列长度。实际系统通常由多个变量共同刻画。设变量维度为 $D$，第 $t$ 个时刻的观测向量为`

function loadLongExample() {
  longTextInput.value = longExample
  convertLongText()
}

const MATH_ENVS = new Set([
  'equation', 'equation*',
  'align', 'align*',
  'gather', 'gather*',
  'multline', 'multline*',
  'eqnarray', 'eqnarray*',
  'cases', 'cases*',
  'matrix', 'matrix*',
  'pmatrix', 'pmatrix*',
  'bmatrix', 'bmatrix*',
  'vmatrix', 'vmatrix*',
  'Bmatrix', 'Bmatrix*',
  'Vmatrix', 'Vmatrix*',
  'smallmatrix', 'smallmatrix*',
  'aligned', 'aligned*',
  'gathered', 'gathered*',
  'split', 'split*',
  'array',
  'displaymath',
  'math',
])

function parseTextWithMath(text) {
  const segments = []
  let pos = 0

  // Match \begin{...}, \[...\], $$...$$, or $...$
  const tokenPattern = /\\begin\{[^}]+\}|\\\[|\\\]|\$\$|\$/g
  let match

  while ((match = tokenPattern.exec(text)) !== null) {
    const token = match[0]
    const start = match.index

    if (token.startsWith('\\begin{')) {
      const envName = token.match(/\\begin\{([^}]+)\}/)[1]

      // Skip non-math environments — leave them as plain text
      if (!MATH_ENVS.has(envName)) continue

      // Find matching \end{env} with nesting support
      let depth = 1
      const innerPattern = new RegExp('\\\\begin\\{' + escapeRegex(envName) + '\\}|\\\\end\\{' + escapeRegex(envName) + '\\}', 'g')
      innerPattern.lastIndex = tokenPattern.lastIndex

      let m
      while ((m = innerPattern.exec(text)) !== null) {
        if (m[0].startsWith('\\begin{')) depth++
        else depth--
        if (depth === 0) {
          const raw = text.slice(start, m.index + m[0].length)
          if (start > pos) {
            segments.push({ type: 'text', content: text.slice(pos, start) })
          }
          segments.push({ type: 'math', content: raw, displayMode: true, raw })
          pos = start + raw.length
          tokenPattern.lastIndex = pos
          break
        }
      }
    } else if (token === '\\[') {
      // Display math \[...\]
      const endIdx = text.indexOf('\\]', start + 2)
      if (endIdx === -1) continue
      const raw = text.slice(start, endIdx + 2)
      if (start > pos) {
        segments.push({ type: 'text', content: text.slice(pos, start) })
      }
      segments.push({ type: 'math', content: raw, displayMode: true, raw })
      pos = endIdx + 2
      tokenPattern.lastIndex = pos
    } else if (token === '$$') {
      // Display math $$...$$
      const endIdx = text.indexOf('$$', start + 2)
      if (endIdx === -1) continue
      const raw = text.slice(start, endIdx + 2)
      if (start > pos) {
        segments.push({ type: 'text', content: text.slice(pos, start) })
      }
      segments.push({ type: 'math', content: raw.slice(2, -2), displayMode: true, raw })
      pos = endIdx + 2
      tokenPattern.lastIndex = pos
    } else {
      // Inline math $...$
      const endIdx = text.indexOf('$', start + 1)
      if (endIdx === -1) continue
      if (endIdx === start + 1) continue
      const raw = text.slice(start, endIdx + 1)
      if (start > pos) {
        segments.push({ type: 'text', content: text.slice(pos, start) })
      }
      segments.push({ type: 'math', content: raw.slice(1, -1), displayMode: false, raw })
      pos = endIdx + 1
      tokenPattern.lastIndex = pos
    }
  }

  if (pos < text.length) {
    segments.push({ type: 'text', content: text.slice(pos) })
  }

  return segments
}

function escapeRegex(str) {
  return str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
}

function convertLongText() {
  error.value = ''
  renderedHtml.value = ''
  renderedText.value = ''
  renderedClipboardHtml.value = ''
  detectedCount.value = 0
  copied.value = false

  if (!longTextInput.value.trim()) return

  try {
    const segments = parseTextWithMath(longTextInput.value)
    let html = ''
    let text = ''
    let mathCount = 0
    const clipboardParts = []

    for (const seg of segments) {
      if (seg.type === 'text') {
        const segContent = removeEmptyLines.value
          ? seg.content.split(/\r?\n/).filter((line) => line.trim() !== '').join('\n')
          : seg.content
        html += escapeHtml(segContent)
        text += segContent
        clipboardParts.push({ type: 'text', content: segContent })
      } else {
        try {
          const mathHtml = temml.renderToString(seg.content, { displayMode: seg.displayMode, xml: true })
          const mathText = formatXml(mathHtml)
          html += mathHtml
          text += mathText
          clipboardParts.push({
            type: 'math',
            mathml: mathHtml,
            omml: toWordMathHtml(mathHtml, seg.displayMode),
            displayMode: seg.displayMode,
          })
          mathCount++
        } catch {
          html += '<code>' + escapeHtml(seg.raw) + '</code>'
          text += seg.raw
          clipboardParts.push({ type: 'text', content: seg.raw })
        }
      }
    }

    renderedHtml.value = html
    renderedText.value = text
    renderedClipboardHtml.value = buildWordClipboardHtml(clipboardParts)
    detectedCount.value = mathCount
  } catch (e) {
    error.value = 'Conversion error: ' + e.message
  }
}

function clearLongText() {
  longTextInput.value = ''
  renderedHtml.value = ''
  renderedText.value = ''
  renderedClipboardHtml.value = ''
  detectedCount.value = 0
  error.value = ''
  copied.value = false
  removeEmptyLines.value = false
}

// --- Shared ---
function switchMode(m) {
  mode.value = m
  error.value = ''
  copied.value = false
}

async function copyAll() {
  await copyRichText(renderedClipboardHtml.value, renderedText.value)
}

async function copyRichText(html, text) {
  if (html) {
    try {
      copyHtmlWithExecCommand(html, text)
      markCopied('Copied for Word')
      return
    } catch {
      // Try the async clipboard API next.
    }

    try {
      if (navigator.clipboard?.write && window.ClipboardItem) {
        await navigator.clipboard.write([
          new ClipboardItem({
            'text/html': new Blob([html], { type: 'text/html' }),
            'text/plain': new Blob([text], { type: 'text/plain' }),
          }),
        ])
        markCopied('Copied for Word')
        return
      }
    } catch {
      // Fall back to plain text when rich clipboard access is blocked.
    }
  }

  await copyText(text, 'Copied as plain text')
}

async function copyText(text, message = 'Copied!') {
  try {
    await navigator.clipboard.writeText(text)
    markCopied(message)
  } catch {
    const ta = document.createElement('textarea')
    ta.value = text
    document.body.appendChild(ta)
    ta.select()
    document.execCommand('copy')
    document.body.removeChild(ta)
    markCopied(message)
  }
}

function copyHtmlWithExecCommand(html, text) {
  const holder = document.createElement('div')
  holder.setAttribute('contenteditable', 'true')
  holder.style.position = 'fixed'
  holder.style.left = '-10000px'
  holder.style.top = '0'
  holder.style.width = '1px'
  holder.style.height = '1px'
  holder.style.overflow = 'hidden'
  holder.innerHTML = html
  document.body.appendChild(holder)

  const range = document.createRange()
  range.selectNodeContents(holder)
  const selection = window.getSelection()
  selection?.removeAllRanges()
  selection?.addRange(range)

  const onCopy = (event) => {
    event.clipboardData.setData('text/html', html)
    event.clipboardData.setData('text/plain', text)
    event.preventDefault()
  }

  document.addEventListener('copy', onCopy)
  try {
    const ok = document.execCommand('copy')

    if (!ok) {
      throw new Error('HTML copy failed')
    }
  } finally {
    document.removeEventListener('copy', onCopy)
    selection?.removeAllRanges()
    document.body.removeChild(holder)
  }
}

function markCopied(message = 'Copied!') {
  copyMessage.value = message
  copied.value = true
  setTimeout(() => (copied.value = false), 2000)
}

function escapeHtml(str) {
  return str
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/\n/g, '<br>')
    .replace(/  /g, ' &nbsp;')
}

function escapeHtmlInline(str) {
  return str
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/ {2,}/g, (spaces) => ' ' + '&nbsp;'.repeat(spaces.length - 1))
}

function toWordMathHtml(mathml, displayMode) {
  const omml = normalizeOmmlForWordHtml(mml2omml(cleanMathmlForOmml(mathml)))
  return displayMode ? `<m:oMathPara>${omml}</m:oMathPara>` : omml
}

function cleanMathmlForOmml(mathml) {
  if (typeof DOMParser === 'undefined' || typeof XMLSerializer === 'undefined') {
    return cleanMathmlString(mathml)
  }

  const doc = new DOMParser().parseFromString(mathml, 'application/xml')

  if (doc.getElementsByTagName('parsererror').length) {
    return cleanMathmlString(mathml)
  }

  const math = doc.documentElement
  unwrapTemmlEquationTable(math)
  unwrapMathmlElements(math, new Set(['mpadded', 'span']))

  return new XMLSerializer().serializeToString(math)
}

function cleanMathmlString(mathml) {
  return mathml
    .replace(/<span\b[^>]*><\/span>/g, '')
    .replace(/<span\b[^>]*>/g, '')
    .replace(/<\/span>/g, '')
    .replace(/<mpadded\b[^>]*>/g, '')
    .replace(/<\/mpadded>/g, '')
}

function unwrapTemmlEquationTable(math) {
  const children = elementChildren(math)

  if (children.length !== 1 || children[0].localName !== 'mtable') return

  const cells = Array.from(children[0].getElementsByTagName('mtd'))
  const contentCell = cells.find((cell) => (cell.getAttribute('class') || '').includes('tml-left'))

  if (!contentCell) return

  const content = Array.from(contentCell.childNodes).map((node) => node.cloneNode(true))

  while (math.firstChild) {
    math.removeChild(math.firstChild)
  }

  content.forEach((node) => math.appendChild(node))
}

function unwrapMathmlElements(root, localNames) {
  let changed = true

  while (changed) {
    changed = false
    const targets = Array.from(root.getElementsByTagName('*')).filter((node) => localNames.has(node.localName))

    targets.forEach((node) => {
      const parent = node.parentNode
      if (!parent) return

      while (node.firstChild) {
        parent.insertBefore(node.firstChild, node)
      }

      parent.removeChild(node)
      changed = true
    })
  }
}

function elementChildren(node) {
  return Array.from(node.childNodes).filter((child) => child.nodeType === 1)
}

function normalizeOmmlForWordHtml(omml) {
  return cleanOmmlForWord(replaceEmptyNaryOperators(omml))
    .replaceAll(
      'http://schemas.openxmlformats.org/officeDocument/2006/math',
      'http://schemas.microsoft.com/office/2004/12/omml'
    )
    .replace(/\sxmlns:w="[^"]+"/g, '')
}

function cleanOmmlForWord(omml) {
  return omml.replaceAll('⁡', '')
}

function replaceEmptyNaryOperators(omml) {
  if (typeof DOMParser === 'undefined' || typeof XMLSerializer === 'undefined') {
    return omml
  }

  const doc = new DOMParser().parseFromString(omml, 'application/xml')

  if (doc.getElementsByTagName('parsererror').length) {
    return omml
  }

  const naries = Array.from(doc.getElementsByTagName('m:nary'))

  naries.forEach((nary) => {
    const expression = directOmmlChild(nary, 'e')

    if (!expression || hasOmmlContent(expression)) return

    const mathNs = nary.namespaceURI
    const naryPr = directOmmlChild(nary, 'naryPr')
    const operator = getOmmlAttribute(directOmmlChild(naryPr, 'chr'), 'val') || '∑'
    const limLoc = getOmmlAttribute(directOmmlChild(naryPr, 'limLoc'), 'val')
    const sub = directOmmlChild(nary, 'sub')
    const sup = directOmmlChild(nary, 'sup')
    const hasSub = hasOmmlContent(sub)
    const hasSup = hasOmmlContent(sup)

    const replacement = buildScriptOperator(doc, mathNs, operator, sub, sup, hasSub, hasSup)

    nary.parentNode.replaceChild(replacement, nary)
  })

  return new XMLSerializer().serializeToString(doc.documentElement)
}

function directOmmlChild(parent, localName) {
  if (!parent) return null

  return Array.from(parent.childNodes).find(
    (node) => node.nodeType === 1 && node.localName === localName
  ) || null
}

function hasOmmlContent(element) {
  return Boolean(element) && Array.from(element.childNodes).some(
    (node) => node.nodeType === 1 || node.textContent.trim()
  )
}

function getOmmlAttribute(element, localName) {
  if (!element) return ''

  return element.getAttribute(`m:${localName}`)
    || element.getAttribute(localName)
    || element.getAttributeNS(element.namespaceURI, localName)
    || ''
}

function buildScriptOperator(doc, mathNs, operator, sub, sup, hasSub, hasSup) {
  if (hasSub && hasSup) {
    const wrapper = createOmmlElement(doc, mathNs, 'sSubSup')
    wrapper.append(createScriptProps(doc, mathNs, 'sSubSupPr'))
    wrapper.append(createOperatorExpression(doc, mathNs, operator), sub, sup)
    return wrapper
  }

  if (hasSub) {
    const wrapper = createOmmlElement(doc, mathNs, 'sSub')
    wrapper.append(createScriptProps(doc, mathNs, 'sSubPr'))
    wrapper.append(createOperatorExpression(doc, mathNs, operator), sub)
    return wrapper
  }

  if (hasSup) {
    const wrapper = createOmmlElement(doc, mathNs, 'sSup')
    wrapper.append(createScriptProps(doc, mathNs, 'sSupPr'))
    wrapper.append(createOperatorExpression(doc, mathNs, operator), sup)
    return wrapper
  }

  return createOperatorRun(doc, mathNs, operator)
}

function createScriptProps(doc, mathNs, propName) {
  const props = createOmmlElement(doc, mathNs, propName)
  props.append(createOmmlElement(doc, mathNs, 'ctrlPr'))
  return props
}

function createOperatorExpression(doc, mathNs, operator) {
  const expression = createOmmlElement(doc, mathNs, 'e')
  expression.append(createOperatorRun(doc, mathNs, operator))
  return expression
}

function createOperatorRun(doc, mathNs, operator) {
  const run = createOmmlElement(doc, mathNs, 'r')
  const text = createOmmlElement(doc, mathNs, 't')
  text.setAttributeNS('http://www.w3.org/XML/1998/namespace', 'xml:space', 'preserve')
  text.textContent = operator
  run.append(text)
  return run
}

function createOmmlElement(doc, mathNs, localName) {
  return doc.createElementNS(mathNs, `m:${localName}`)
}

function buildWordClipboardHtml(parts) {
  const fragment = buildWordParagraphs(parts)

  return [
    '<!DOCTYPE html>',
    '<html xmlns="http://www.w3.org/1999/xhtml" xmlns:m="http://schemas.microsoft.com/office/2004/12/omml">',
    '<head>',
    '<meta charset="utf-8">',
    '<style>',
    'body{font-family:Arial,"Microsoft YaHei",sans-serif;font-size:11pt;}',
    'p{margin:0 0 8pt 0;}',
    '</style>',
    '</head>',
    `<body><!--StartFragment-->${fragment}<!--EndFragment--></body>`,
    '</html>',
  ].join('')
}

function buildWordParagraphs(parts) {
  const paragraphs = [[]]
  const current = () => paragraphs[paragraphs.length - 1]
  const currentHasDisplayMath = () => current().some((item) => item.includes('<m:oMathPara>'))

  const startParagraph = () => {
    paragraphs.push([])
  }

  const appendText = (content) => {
    const lines = content.replace(/\r\n?/g, '\n').split('\n')

    lines.forEach((line, index) => {
      if (index > 0) startParagraph()
      if (!line.length) return
      if (currentHasDisplayMath()) startParagraph()
      current().push(escapeHtmlInline(line))
    })
  }

  const appendMath = (part) => {
    if (part.displayMode && current().length) startParagraph()
    if (!part.displayMode && currentHasDisplayMath()) startParagraph()
    current().push(part.omml || part.mathml)
  }

  parts.forEach((part) => {
    if (part.type === 'text') appendText(part.content)
    else appendMath(part)
  })

  return paragraphs
    .map((paragraph) => {
      const body = paragraph.join('') || '&nbsp;'
      return `<p>${body}</p>`
    })
    .join('')
}

function formatXml(xml) {
  let formatted = ''
  let indent = ''
  const tab = '  '
  xml.split(/>\s*</).forEach((node) => {
    if (node.match(/^\/\w/)) indent = indent.substring(tab.length)
    formatted += indent + '<' + node + '>\n'
    if (node.match(/^<?\w[^>]*[^/]$/) && !node.startsWith('?')) indent += tab
  })
  return formatted.substring(1, formatted.length - 2)
}

onMounted(() => {
  latexInput.value = 'E = mc^2'
  convertSingle()
})
</script>
