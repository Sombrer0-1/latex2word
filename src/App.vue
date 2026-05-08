<template>
  <div>
    <h1>LaTeX → MathML Converter</h1>
    <p class="author-credit">
      by <a href="https://github.com/Sombrer0-1" target="_blank">Sombrer0-1</a> · 930937337@qq.com
    </p>

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
      <div class="scroll-nav-top">
        <button class="btn-scroll" @click="scrollPageToBottom">↓ 传送到底部</button>
      </div>

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
          <label class="checkbox-label">
            <input type="checkbox" v-model="pureMode" @change="convertLongText" />
            <span>纯净模式</span>
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

      <div class="scroll-nav-btm">
        <button class="btn-scroll" @click="scrollPageToTop">↑ 返回顶部</button>
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
const pureMode = ref(false)

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

const TABLE_ENVS = new Set([
  'tabular', 'tabular*', 'tabularx', 'tabulary', 'longtable',
])

const ALGO_ENVS = new Set([
  'algorithm',
])

function matchBraceGroup(str, pos, openChar = '{') {
  const closeChar = openChar === '{' ? '}' : ']'
  let depth = 0
  let i = pos
  while (i < str.length) {
    if (str[i] === '\\') { i += 2; continue }
    if (str[i] === openChar) depth++
    else if (str[i] === closeChar) { depth--; if (depth === 0) return i }
    i++
  }
  return -1
}

function splitTableRow(row) {
  const cells = []
  let current = ''
  let depth = 0
  for (let i = 0; i < row.length; i++) {
    const ch = row[i]
    if (ch === '\\') { current += ch + (row[i + 1] || ''); i++; continue }
    if (ch === '{') depth++
    else if (ch === '}') depth--
    else if (ch === '&' && depth === 0) {
      cells.push(current.trim())
      current = ''
      continue
    }
    current += ch
  }
  cells.push(current.trim())
  return cells
}

function processLatexFormatting(content) {
  const commands = [
    { cmd: /\\textbf\{/g, tag: 'b' },
    { cmd: /\\textit\{/g, tag: 'i' },
    { cmd: /\\emph\{/g, tag: 'em' },
  ]
  for (const { cmd, tag } of commands) {
    let result = ''
    let pos = 0
    let m
    while ((m = cmd.exec(content)) !== null) {
      result += content.slice(pos, m.index)
      const start = m.index + m[0].length - 1
      const end = matchBraceGroup(content, start)
      if (end !== -1) {
        const inner = processLatexFormatting(content.slice(start + 1, end))
        result += `<${tag}>${inner}</${tag}>`
        pos = end + 1
        cmd.lastIndex = pos
      } else {
        result += m[0]
        pos = m.index + m[0].length
        cmd.lastIndex = pos
      }
    }
    result += content.slice(pos)
    content = result
  }
  return content
}

function unescapeLatexChars(cell) {
  return cell
    .replace(/\\%/g, '%')
    .replace(/\\\$/g, '$')
    .replace(/\\&/g, '&')
    .replace(/\\_/g, '_')
    .replace(/\\#/g, '#')
    .replace(/\\\{/g, '\u0001')   // placeholder, restored after formatting
    .replace(/\\\}/g, '\u0002')
}

function unescapeLatexText(str) {
  return str
    .replace(/\\%/g, '%')
    .replace(/\\\$/g, '$')
    .replace(/\\&/g, '&')
    .replace(/\\_/g, '_')
    .replace(/\\#/g, '#')
    .replace(/\\\{/g, '{')
    .replace(/\\\}/g, '}')
}

function restoreLatexBraces(cell) {
  return cell
    .replace(/\u0001/g, '{')
    .replace(/\u0002/g, '}')
}

function processInlineMath(cell) {
  const placeholders = []
  const result = cell.replace(/\$(.+?)\$/g, (match, latex) => {
    try {
      const rendered = temml.renderToString(latex, { displayMode: false, xml: true })
      placeholders.push(rendered)
      return `\u0003${placeholders.length - 1}\u0004`
    } catch {
      placeholders.push(match)
      return `\u0003${placeholders.length - 1}\u0004`
    }
  })
  return { text: result, placeholders }
}

function restoreInlineMath(text, placeholders) {
  return text.replace(/\u0003(\d+)\u0004/g, (_, idx) => placeholders[Number(idx)] || '')
}

function processTableCellContent(content) {
  let cell = content
  // Unwrap \multicolumn{n}{col}{content}
  const mcMatch = cell.match(/^\\multicolumn\{(\d+)\}\{[^}]*\}\{/)
  if (mcMatch) {
    const start = mcMatch.index + mcMatch[0].length - 1
    const end = matchBraceGroup(cell, start)
    if (end !== -1) {
      cell = cell.slice(start + 1, end)
    }
  }
  // Step 1: Process inline math while content is raw LaTeX
  const { text: mathProcessed, placeholders } = processInlineMath(cell)
  cell = mathProcessed
  // Step 2: Convert LaTeX typographic escapes
  cell = unescapeLatexChars(cell)
  // Step 3: HTML escape
  cell = cell
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/~/g, '&nbsp;')
  // Step 4: Formatting
  cell = processLatexFormatting(cell)
  // Step 5: Restore inline math and brace placeholders
  cell = restoreInlineMath(cell, placeholders)
  cell = restoreLatexBraces(cell)
  // Step 6: Strip remaining LaTeX commands
  cell = cell.replace(/\\[a-zA-Z]+\*?\s*(\{[^}]*\})*/g, '')
  return cell || '&nbsp;'
}

function parseTableContent(rawTable) {
  const beginMatch = rawTable.match(/\\begin\{(\w+\*?)\}/)
  if (!beginMatch) return null

  const envName = beginMatch[1]
  let pos = beginMatch.index + beginMatch[0].length

  // Skip optional [pos] arg
  while (pos < rawTable.length && /\s/.test(rawTable[pos])) pos++
  if (rawTable[pos] === '[') {
    const bracketEnd = matchBraceGroup(rawTable, pos, '[')
    if (bracketEnd === -1) return null
    pos = bracketEnd + 1
  }
  while (pos < rawTable.length && /\s/.test(rawTable[pos])) pos++

  // Parse column spec {cols}
  if (rawTable[pos] !== '{') return null
  const colSpecEnd = matchBraceGroup(rawTable, pos)
  if (colSpecEnd === -1) return null
  pos = colSpecEnd + 1

  // Locate closing \end{env}
  const endStr = `\\end{${envName}}`
  const endIdx = rawTable.lastIndexOf(endStr)
  if (endIdx === -1) return null

  const body = rawTable.slice(pos, endIdx).trim()

  // Split body into rows by \\
  const lines = body.split(/\\\\\s*(?:\[[^\]]*\])?\s*/)
    .map((line) => line.trim())
    .filter((line) => line.length > 0)

  // Parse rows, stripping \hline and booktabs rules
  const parsedRows = []
  for (const line of lines) {
    // Standalone rule commands
    if (/^\\(?:hline|toprule|midrule|bottomrule)\s*$/.test(line)) continue
    if (/^\\cline\{[^}]+\}\s*$/.test(line)) continue
    // Strip rule commands within rows
    let cleaned = line.replace(/\\(?:hline|toprule|midrule|bottomrule)\s*/g, '')
    if (cleaned.length === 0) continue
    parsedRows.push(cleaned)
  }

  // Build HTML table
  let html = '<table border="1" cellspacing="0" cellpadding="0">\n'
  let plainText = ''

  for (const row of parsedRows) {
    const cells = splitTableRow(row)
    html += '  <tr>\n'
    const plainRow = []
    for (const cell of cells) {
      const content = processTableCellContent(cell)
      html += `    <td>${content}</td>\n`
      // Plain text: strip LaTeX commands, keep cell text
      plainRow.push(cell.replace(/\\[a-zA-Z]+\*?\s*(\{[^}]*\})*/g, '').trim() || ' ')
    }
    html += '  </tr>\n'
    plainText += plainRow.join('\t') + '\n'
  }

  html += '</table>'
  return { html, plainText: plainText.trimEnd() }
}

// --- Algorithm Environment Parsing ---

function processAlgoText(text) {
  let t = text.trim()

  // Inline math → temml (placeholder approach to protect from HTML escaping)
  const mathData = processInlineMath(t)
  t = mathData.text

  // Symbol replacements (before HTML escape)
  t = t
    .replace(/\\leftarrow\b/g, '\u2190')
    .replace(/\\gets\b/g, '\u2190')
    .replace(/\\rightarrow\b/g, '\u2192')
    .replace(/\\to\b/g, '\u2192')
    .replace(/\\Leftarrow\b/g, '\u21D0')
    .replace(/\\Rightarrow\b/g, '\u21D2')
    .replace(/\\leftrightarrow\b/g, '\u2194')
    .replace(/\\uparrow\b/g, '\u2191')
    .replace(/\\downarrow\b/g, '\u2193')
    .replace(/\\neq\b/g, '\u2260')
    .replace(/\\le\b/g, '\u2264')
    .replace(/\\ge\b/g, '\u2265')

  // LaTeX escapes
  t = unescapeLatexText(t)

  // HTML escape
  t = t
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')

  // Formatting
  t = processLatexFormatting(t)

  // Restore inline math
  t = restoreInlineMath(t, mathData.placeholders)

  // Strip remaining LaTeX commands
  t = t.replace(/\\[a-zA-Z]+\*?\s*(\{[^}]*\})*/g, '')

  return t || '&nbsp;'
}

function algoContentToHtml(content) {
  const parts = []
  if (content.keyword) parts.push(`<b>${content.keyword}</b>`)
  if (content.cond) parts.push(` ${content.cond}`)
  if (content.keyword === 'else if' || content.keyword === 'if' || content.keyword === 'for' || content.keyword === 'while') {
    parts.push(' <b>then</b>')
  } else if (content.keyword === 'for' || content.keyword === 'while') {
    parts.push(' <b>do</b>')
  }
  if (content.text) parts.push(` ${content.text}`)
  if (content.then) parts.push(` ${content.then}`)
  return parts.join('')
}

function algoContentToPlain(content) {
  const parts = []
  if (content.keyword) parts.push(content.keyword.toUpperCase())
  if (content.cond) parts.push(` ${content.cond}`)
  if (content.text) parts.push(` ${content.text}`)
  if (content.then) parts.push(` ${content.then}`)
  return parts.join('')
}

function parseAlgoLine(line, indent) {
  let cmd, content, newIndent = indent

  const cmdMatch = line.match(/^\\([A-Z][A-Za-z]*)\b\s*/)
  if (cmdMatch) {
    cmd = cmdMatch[1]
    let rest = line.slice(cmdMatch[0].length)

    switch (cmd) {
      case 'REQUIRE':
        content = { keyword: 'Require:', text: processAlgoText(rest) }
        break
      case 'ENSURE':
        content = { keyword: 'Ensure:', text: processAlgoText(rest) }
        break
      case 'STATE':
        content = { text: processAlgoText(rest) }
        break
      case 'IF':
      case 'ELSIF': {
        let cond = ''
        const condMatch = rest.match(/^\{/)
        if (condMatch) {
          const end = matchBraceGroup(rest, condMatch.index)
          cond = rest.slice(condMatch.index + 1, end)
          rest = rest.slice(end + 1)
        }
        const label = cmd === 'IF' ? 'if' : 'else if'
        content = { keyword: label, cond: processAlgoText(cond), text: processAlgoText(rest) }
        if (cmd === 'IF') newIndent = indent + 1
        break
      }
      case 'ELSE':
        content = { keyword: 'else' }
        return { content, indent: Math.max(0, indent - 1), indentNext: indent }
      case 'ENDIF':
        content = { keyword: 'end if' }
        newIndent = Math.max(0, indent - 1)
        return { content, indent: newIndent, indentNext: newIndent }
      case 'FOR': {
        let cond = ''
        const condMatch = rest.match(/^\{/)
        if (condMatch) {
          const end = matchBraceGroup(rest, condMatch.index)
          cond = rest.slice(condMatch.index + 1, end)
        }
        content = { keyword: 'for', cond: processAlgoText(cond), text: 'do' }
        newIndent = indent + 1
        break
      }
      case 'ENDFOR':
        content = { keyword: 'end for' }
        newIndent = Math.max(0, indent - 1)
        return { content, indent: newIndent, indentNext: newIndent }
      case 'ENDWHILE':
        content = { keyword: 'end while' }
        newIndent = Math.max(0, indent - 1)
        return { content, indent: newIndent, indentNext: newIndent }
      case 'WHILE': {
        let cond = ''
        const condMatch = rest.match(/^\{/)
        if (condMatch) {
          const end = matchBraceGroup(rest, condMatch.index)
          cond = rest.slice(condMatch.index + 1, end)
        }
        content = { keyword: 'while', cond: processAlgoText(cond), text: 'do' }
        newIndent = indent + 1
        break
      }
      case 'ENDWHILE':
        content = { keyword: 'end while' }
        newIndent = Math.max(0, indent - 1)
        break
      case 'RETURN':
        content = { keyword: 'return', text: processAlgoText(rest) }
        break
      case 'PRINT':
        content = { keyword: 'print', text: processAlgoText(rest) }
        break
      case 'COMMENT':
        content = { keyword: '//', text: processAlgoText(rest) }
        break
      default:
        content = { text: processAlgoText(line) }
    }
  } else {
    content = { text: processAlgoText(line) }
  }

  return { content, indent, indentNext: newIndent }
}

function parseAlgorithmicBody(body, startNum) {
  const lines = body.split(/\r?\n/).map((l) => l.trim()).filter((l) => l.length > 0 && !l.startsWith('%'))
  const rows = []
  let lineNum = startNum
  let indent = 0

  for (const line of lines) {
    const { content, indent: rowIndent, indentNext } = parseAlgoLine(line, indent)
    const isHeader = /^\\(?:REQUIRE|ENSURE)\b/.test(line)
    const num = (!isHeader && content) ? lineNum : ''
    rows.push({
      number: num,
      indent: rowIndent,
      html: content ? algoContentToHtml(content) : processAlgoText(line),
      plainText: content ? algoContentToPlain(content) : processAlgoText(line).replace(/<[^>]+>/g, ''),
    })
    indent = indentNext
    if (!isHeader && content) lineNum++
  }

  return rows
}

function parseAlgorithmContent(raw) {
  const beginMatch = raw.match(/\\begin\{algorithm\}/)
  if (!beginMatch) return null

  let pos = beginMatch.index + beginMatch[0].length
  while (pos < raw.length && /\s/.test(raw[pos])) pos++
  if (raw[pos] === '[') {
    const end = matchBraceGroup(raw, pos, '[')
    if (end === -1) return null
    pos = end + 1
  }

  const endIdx = raw.lastIndexOf('\\end{algorithm}')
  if (endIdx === -1) return null

  const body = raw.slice(pos, endIdx).trim()

  // Extract caption
  let captionText = ''
  const capMatch = body.match(/\\caption\{([^}]*)\}/)
  if (capMatch) captionText = capMatch[1].trim()

  // Extract algorithmic environment
  let lineNum = 1
  let algoBody = ''
  const algoMatch = body.match(/\\begin\{algorithmic\}(?:\[(\d+)\])?/)
  if (algoMatch) {
    if (algoMatch[1]) lineNum = parseInt(algoMatch[1])
    const algoStart = algoMatch.index + algoMatch[0].length
    const algoEndMatch = body.match(/\\end\{algorithmic\}/)
    const algoEnd = algoEndMatch ? algoEndMatch.index : body.length
    algoBody = body.slice(algoStart, algoEnd).trim()
  } else {
    algoBody = body
      .replace(/\\caption\{[^}]*\}\s*/g, '')
      .replace(/\\label\{[^}]*\}\s*/g, '')
      .trim()
  }

  const rows = parseAlgorithmicBody(algoBody, lineNum)

  let html = '<table border="1" cellspacing="0" cellpadding="0">\n'
  let plainText = ''

  if (captionText) {
    html += `  <tr><td><b>${captionText}</b></td></tr>\n`
    plainText += captionText + '\n'
  }

  // Single cell containing all code lines
  let codeHtml = ''
  let codePlain = ''
  for (const row of rows) {
    const num = row.number ? `${row.number}:` : ''
    const padding = '&nbsp;'.repeat(row.indent * 4)
    codeHtml += num + padding + row.html + '<br>'
    codePlain += (row.number ? row.number + '. ' : '') + '  '.repeat(row.indent) + row.plainText + '\n'
  }

  html += `  <tr><td>${codeHtml}</td></tr>\n`
  html += '</table>'
  plainText += codePlain
  return { html, plainText: plainText.trimEnd() }
}

// --- Pure Mode: strip LaTeX boilerplate ---

function stripCommandWithArgs(text, cmdName) {
  const re = new RegExp('\\\\' + cmdName + '\\*?', 'g')
  let out = ''
  let pos = 0
  let m
  while ((m = re.exec(text)) !== null) {
    out += text.slice(pos, m.index)
    let i = m.index + m[0].length
    while (i < text.length && /[\s%]/.test(text[i]) && text[i] !== '\n') i++
    while (i < text.length) {
      if (text[i] === '{') {
        const end = matchBraceGroup(text, i)
        if (end === -1) break
        i = end + 1
      } else if (text[i] === '[') {
        const end = matchBraceGroup(text, i, '[')
        if (end === -1) break
        i = end + 1
      } else {
        break
      }
    }
    pos = i
    re.lastIndex = i
  }
  out += text.slice(pos)
  return out
}

function unwrapResizebox(text) {
  const re = /\\resizebox\*?\s*/g
  let out = ''
  let pos = 0
  let m
  while ((m = re.exec(text)) !== null) {
    out += text.slice(pos, m.index)
    let i = m.index + m[0].length
    // Skip first brace group: {\textwidth}
    while (i < text.length && /\s/.test(text[i])) i++
    if (text[i] === '{') {
      const end = matchBraceGroup(text, i)
      if (end === -1) break
      i = end + 1
    }
    // Skip second brace group: {!}
    while (i < text.length && /\s/.test(text[i])) i++
    if (text[i] === '{') {
      const end = matchBraceGroup(text, i)
      if (end === -1) break
      i = end + 1
    }
    // Keep third brace group content (the actual table/box)
    while (i < text.length && /\s/.test(text[i])) i++
    if (text[i] === '{') {
      const end = matchBraceGroup(text, i)
      if (end === -1) break
      // Extract inner content (without outer braces)
      const inner = text.slice(i + 1, end)
      out += inner
      i = end + 1
    }
    pos = i
    re.lastIndex = i
  }
  out += text.slice(pos)
  return out
}

function cleanTableFloatBlock(block) {
  // extract \caption text, strip everything else, keep tabular
  let captionText = ''
  const capMatch = block.match(/\\caption\{([^}]*)\}/)
  if (capMatch) {
    captionText = capMatch[1].trim()
  }
  // Unwrap \resizebox
  let cleaned = unwrapResizebox(block)
  // Strip \begin{table}[...] and \end{table}
  cleaned = cleaned.replace(/\\begin\{table\}(\[[^\]]*\])?\s*/g, '')
  cleaned = cleaned.replace(/\\end\{table\}\s*/g, '')
  // Strip \centering, \label, \caption
  cleaned = cleaned.replace(/\\centering\s*/g, '')
  cleaned = cleaned.replace(/\\label\{[^}]*\}\s*/g, '')
  cleaned = cleaned.replace(/\\caption\{[^}]*\}\s*/g, '')
  // Add caption text above the table if present
  if (captionText) {
    cleaned = captionText + '\n' + cleaned.trim()
  }
  return cleaned.trim()
}

function cleanTableFloats(text) {
  let result = ''
  let pos = 0
  const tablePattern = /\\begin\{table\}/g
  let m
  while ((m = tablePattern.exec(text)) !== null) {
    result += text.slice(pos, m.index)
    const start = m.index
    let depth = 1
    const innerPattern = /\\begin\{table\}|\\end\{table\}/g
    innerPattern.lastIndex = tablePattern.lastIndex
    let im
    while ((im = innerPattern.exec(text)) !== null) {
      if (im[0].startsWith('\\begin{')) depth++
      else depth--
      if (depth === 0) {
        const block = text.slice(start, im.index + im[0].length)
        result += cleanTableFloatBlock(block) + '\n'
        pos = im.index + im[0].length
        tablePattern.lastIndex = pos
        break
      }
    }
  }
  result += text.slice(pos)
  return result
}

const BOILERPLATE_NO_ARG = [
  'tableofcontents', 'listoffigures', 'listoftables',
  'clearpage', 'newpage', 'pagebreak', 'linebreak',
  'bigskip', 'medskip', 'smallskip',
  'noindent', 'indent', 'centering',
  'raggedright', 'raggedleft',
]

const BOILERPLATE_WITH_ARGS = [
  'usepackage', 'documentclass', 'setstretch', 'setlist', 'setcounter',
  'addtocounter', 'setlength', 'addtolength', 'settodepth', 'addtodepth',
  'newcommand', 'renewcommand', 'providecommand', 'def',
  'newenvironment', 'renewenvironment', 'newtheorem',
  'DeclareMathOperator', 'DeclareMathSymbol',
  'hypersetup', 'allowdisplaybreaks', 'setenumerate',
  'newcolumntype', 'definecolor', 'renewcommand',
]

function cleanLatexSource(text) {
  let s = text

  // 1. Strip preamble (everything before \begin{document})
  const docStart = s.match(/\\begin\{document\}/)
  if (docStart) {
    s = s.slice(docStart.index + docStart[0].length)
  }

  // 2. Strip \end{document} and trailing content
  s = s.replace(/\\end\{document\}[\s\S]*$/, '')

  // 3. Extract headings: \chapter/section/subsection/subsubsection to plain text
  s = s.replace(/\\chapter\*?\{([^}]*)\}/g, '\n$1\n')
  s = s.replace(/\\section\*?\{([^}]*)\}/g, '\n$1\n')
  s = s.replace(/\\subsection\*?\{([^}]*)\}/g, '\n$1\n')
  s = s.replace(/\\subsubsection\*?\{([^}]*)\}/g, '\n$1\n')

  // 4. Clean table floats
  s = cleanTableFloats(s)

  // 5. Strip no-argument boilerplate commands
  for (const cmd of BOILERPLATE_NO_ARG) {
    s = s.replace(new RegExp('\\\\' + cmd + '\\*?\\s*', 'g'), '')
  }

  // 6. Strip boilerplate commands with arguments (brace-aware)
  for (const cmd of BOILERPLATE_WITH_ARGS) {
    s = stripCommandWithArgs(s, cmd)
  }

  // 7. Strip comments (lines starting with %, but not \%)
  s = s.replace(/(?<!\\)%.*$/gm, '')

  // 8. Clean up: remove leading/trailing whitespace on each line, collapse 3+ blank lines
  s = s.replace(/[ \t]+$/gm, '')
  s = s.replace(/\n{3,}/g, '\n\n')
  s = s.trim()

  return s
}

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

      // Handle table environments
      if (TABLE_ENVS.has(envName)) {
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
            const tableData = parseTableContent(raw)
            if (tableData) {
              segments.push({ type: 'table', html: tableData.html, plainText: tableData.plainText, raw })
            } else {
              segments.push({ type: 'text', content: raw })
            }
            pos = start + raw.length
            tokenPattern.lastIndex = pos
            break
          }
        }
        continue
      }

      // Handle algorithm environments
      if (ALGO_ENVS.has(envName)) {
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
            const algoData = parseAlgorithmContent(raw)
            if (algoData) {
              segments.push({ type: 'table', html: algoData.html, plainText: algoData.plainText, raw })
            } else {
              segments.push({ type: 'text', content: raw })
            }
            pos = start + raw.length
            tokenPattern.lastIndex = pos
            break
          }
        }
        continue
      }

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
    const srcText = pureMode.value ? cleanLatexSource(longTextInput.value) : longTextInput.value
    const segments = parseTextWithMath(srcText)
    let html = ''
    let text = ''
    let mathCount = 0
    const clipboardParts = []

    for (const seg of segments) {
      if (seg.type === 'text') {
        const segContent = removeEmptyLines.value
          ? seg.content.split(/\r?\n/).filter((line) => line.trim() !== '').join('\n')
          : seg.content
        const unescaped = unescapeLatexText(segContent)
        html += escapeHtml(unescaped)
        text += unescaped
        clipboardParts.push({ type: 'text', content: unescaped })
      } else if (seg.type === 'table') {
        html += seg.html
        text += seg.plainText + '\n'
        clipboardParts.push({ type: 'table', html: seg.html, plainText: seg.plainText })
      } else {
        try {
          const mathHtml = temml.renderToString(seg.content, { displayMode: seg.displayMode, xml: true })
          const mathText = formatXml(mathHtml)
          html += mathHtml
          text += mathText
          clipboardParts.push({
            type: 'math',
            mathml: unwrapSingleChildRows(mathHtml),
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
  pureMode.value = false
}

function scrollPageToBottom() {
  window.scrollTo({ top: document.body.scrollHeight, behavior: 'smooth' })
}

function scrollPageToTop() {
  window.scrollTo({ top: 0, behavior: 'smooth' })
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

function unwrapSingleChildRows(mathml) {
  if (typeof DOMParser === 'undefined') return mathml
  const doc = new DOMParser().parseFromString(mathml, 'application/xml')
  if (doc.getElementsByTagName('parsererror').length) return mathml

  let changed = true
  while (changed) {
    changed = false
    const rows = Array.from(doc.getElementsByTagName('mrow'))
    for (const row of rows) {
      const children = Array.from(row.childNodes).filter((n) => n.nodeType === 1)
      if (children.length === 1 && !row.hasAttributes()) {
        const parent = row.parentNode
        if (parent) {
          parent.replaceChild(children[0], row)
          changed = true
        }
      }
    }
  }

  return new XMLSerializer().serializeToString(doc.documentElement)
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
    if (part.displayMode) {
      if (current().length) startParagraph()
      current().push(part.mathml || part.omml)
      startParagraph()
    } else {
      if (currentHasDisplayMath()) startParagraph()
      current().push(part.mathml || part.omml)
    }
  }

  parts.forEach((part) => {
    if (part.type === 'text') {
      appendText(part.content)
    } else if (part.type === 'table') {
      if (current().length) startParagraph()
      current().push(part.html)
      startParagraph()
    } else {
      appendMath(part)
    }
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
