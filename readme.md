# ReLaXed

「 使用网络技术创建PDF文档 」

[![explain](http://llever.com/explain.svg)](https://github.com/chinanf-boy/Source-Explain)
    
Explanation

> "version": "0.1.3"

[github source](https://github.com/RelaxedJS/ReLaXed)

~~[english](./README.en.md)~~

---

我想这个项目只需要用作者的话来阐述就很好了·

轻松的支持`Markdown, LaTeX-style`数学方程式 (通过[mathjax](https://www.mathjax.org/)) ,csv转换为html表格,绘图生成 (通过[Vega-Lite - 精简版](https://vega.github.io/vega-lite/)要么[chart.js](https://www.chartjs.org/)) 和图表生成 (通过[mermaidjs](https://mermaidjs.github.io/)) 🥄通过导入现有的javascript或css框架,可以添加更多功能. 

---

本目录

---

## package.json

``` js
  "main": "src/index.js",
  "bin": {
    "relaxed": "src/index.js"
  },
```

## index

- [x] [require](#require)  导入
- [x] [cli-commander](#cli-commander)
- [x] [main](#main)
- [x] [chokidar](#chokidar)
- [x] [converters](#converters) 

### require

``` js
#!/usr/bin/env node
const colors = require('colors') // 颜色
const program = require('commander') // 命令行
const chokidar = require('chokidar') // 观察
const puppeteer = require('puppeteer') // 命令行-chrome
const { performance } = require('perf_hooks')
const path = require('path')
const fs = require('fs')
const converters = require('./converters.js') // 多种转译器

```

<details>

<summary> lib github source </summary>

https://github.com/Marak/colors.js
https://github.com/tj/commander.js
https://github.com/paulmillr/chokidar
https://github.com/GoogleChrome/puppeteer
https://nodejs.org/api/perf_hooks.html

</details>

cli-commander

### cli-commander

``` js
var input, output

program
  .version('0.0.1')
  .usage('<input> [output] [options]')
  .arguments('<input> [output] [options]')
  .option('--no-sandbox', 'disable puppeteer sandboxing')
  .option('-w, --watch <locations>', 'Watch other locations', [])
  .option('-t, --temp [location]', 'Directory for temp file')
  .action(function (inp, out) {
    input = inp
    output = out
  })

program.parse(process.argv) // 开启命令解析

if (!input) { // 返回错误
  console.error('Please specifiy an input file');
  process.exit(1)
}

const inputPath = ·path.resolve(input)
const inputDir = path.resolve(inputPath, '..') // 文件的上层目录·
const inputFilenameNoExt = path.basename(input, path.extname(input))

if (!output) {
  output = path.join(inputDir, inputFilenameNoExt + '.pdf')
}

const outputPath = path.resolve(output) // 输出目录

var tempDir 
if (program.temp) {
  var validTempPath = fs.existsSync(program.temp) && fs.statSync(program.temp).isDirectory()
  if (validTempPath) {
    tempDir = path.resolve(program.temp)
  } else {
    console.error('Could not find specified --temp directory: ' + program.temp)
    process.exit(1)
  }
} else {
  tempDir = inputDir
} // 临时文件的目录 tempDir

const tempHTMLPath = path.join(tempDir, inputFilenameNoExt + '_temp.htm')

let watchLocations = [inputDir]
if (program.watch) { // 添加 观察文件赎罪
  watchLocations = watchLocations.concat(program.watch)
}

const puppeteerConfig = { // 设置浏览器命令配置
  headless: true,
  args: program.sandbox ? ['--no-sandbox'] : []
}

```

main

### main

``` js
async function main () {
  console.log('Watching ' + input + ' and its directory tree.')
  const browser = await puppeteer.launch(puppeteerConfig);
  const page = await browser.newPage()
  page.on('pageerror', function (err) { // 浏览器命令 错误 配置
    console.log('Page error: ' + err.toString())
  }).on('error', function (err) {
    console.log('Error: ' + err.toString())
  })

```

chokidar

### chokidar

> 观察文件开始

[chokidar config](https://github.com/paulmillr/chokidar#performance)

``` js

  chokidar.watch(watchLocations, { // 观察文件
    awaitWriteFinish: {
      stabilityThreshold: 50,
      pollInterval: 100
    }
  }).on('change', (filepath) => {
    if (!(['.pug', '.md', '.html', '.css', '.scss', '.svg', '.mermaid',
           '.chart.js', '.png', '.flowchart', '.flowchart.json',
           '.vegalite.json', '.table.csv', 'htable.csv'].some(ext => filepath.endsWith(ext)))) {
      return
    }
    console.log(`\nProcessing detected change in ${filepath.replace(inputDir, '')}...`.magenta.bold)
    var t0 = performance.now() // 开始计数
    var taskPromise = null

```

converters

### converters

转译器 使用

[converters explain](./converters.md)

``` js
    if (['.pug', '.md', '.html', '.css', '.scss', '.svg', '.png'].some(ext => filepath.endsWith(ext))) {
        // (输入完整路径, 浏览器APi, 缓存完整路径, 输出完整路径)
      taskPromise = converters.masterDocumentToPDF(inputPath, page, tempHTMLPath, outputPath)
    } 
    // 正常情况  文档 变 PDf
    
    else if (filepath.endsWith('.chart.js')) {
      taskPromise = converters.chartjsToPNG(filepath, page)
    } 
    // .chart.js 变 png 图片
    
    
    else if (filepath.endsWith('.mermaid')) {
      taskPromise = converters.mermaidToSvg(filepath, page)
    } 
    // .mermaid 变 Svg
    
    
    else if (filepath.endsWith('.flowchart')) {
      taskPromise = converters.flowchartToSvg(filepath, page)
    } 
    // .flowchart 变 Svg
    
    
    else if (filepath.endsWith('.flowchart.json')) {
      var flowchartFile = filepath.substr(0, filepath.length - 5)
      taskPromise = converters.flowchartToSvg(flowchartFile, page)
    } 
      // .flowchart.json 变 SvgartFile, page)
    
    
    else if (filepath.endsWith('.vegalite.json')) {
      taskPromise = converters.vegaliteToSvg(filepath, page)
    } 
    // .vegalite.json 变 Svg
    
    
    else if (['.table.csv', '.htable.csv'].some(ext => filepath.endsWith(ext))) {
      converters.tableToPug(filepath)
    }
    // '.table.csv', '.htable.csv' 变 pug



    if (taskPromise) { // 要正确 才能显示信息
      taskPromise.then(function () {
        var duration = ((performance.now() - t0) / 1000).toFixed(2)
        console.log(`... done in ${duration}s`.magenta.bold)
      })
    }
  })
}

main() // <===== 开启

```