## converters

- [x] [require](#require) 导入
- [x] [formatTemplate](#formattemplate) 将 `*.pug` 模版变 `html`, 这样可以 请求 相关库
- [x] [mermaidToSvg](#mermaidtosvg) mermaid 变 Svg
- [x] [flowchartToSvg](#flowcharttosvg) * 变 *
- [x] [vegaliteToSvg](#vegalitetosvg) * 变 *
- [x] [tableToPug](#tabletopug) * 变 *
- [x] [parseDataUrl](#parsedataurl)  图片 路径 获取
- [x] [chartjsToPNG](#chartjstopng) * 变 *
- [x] [asyncMathjax](#asyncmathjax) 异步设置html 中的数字符号
- [x] [getMatch](#getmatch) 匹配
- [x] [masterDocumentToPDF](#masterdocumenttopdf) `pug`类文档变`pdf` <==❤️

### require

``` js
const fs = require('fs')
const util = require('util')
const path = require('path')
const mjpage = require('mathjax-node-page')
// 此Node.js模块建立在 mathjax-node上 ，并提供较大内容片段的处理
const pug = require('pug')
const writeFile = util.promisify(fs.writeFile)
// async 版本 write
const cheerio = require('cheerio')
// 专为服务器设计的核心jQuery的快速，灵活和精益的实现。可将 html 变成 jQuery类操作
const csv = require('csvtojson')
// nodejs csv解析器将csv转换为json或列数组。它可以用作node.js库/命令行工具/或浏览器
const html2jade = require('html2jade');
// 将HTML转换为Jade模板。不完美但非常有用的非日常转换。
```

<details>

<summary> lib github source </summary>

https://github.com/pugjs/pug
https://github.com/pkra/mathjax-node-page
https://github.com/cheeriojs/cheerio
https://github.com/Keyang/node-csvtojson
https://github.com/donpark/html2jade

</details>

### formatTemplate

将 `*.pug` 模版变 `html`, 这样可以 请求 相关库

``` js
function formatTemplate (tempName, data) {
  return pug.renderFile(path.join(__dirname, 'templates', tempName + '.pug'), data)
}
```

比如 `templates/chartjs.pug`

<details>

``` pug
script(src='https://cdnjs.cloudflare.com/ajax/libs/Chart.js/2.7.2/Chart.min.js') 
<!-- 请求 -->
#chartContainer
  canvas#myChart
script.
  window.pngData = false
  var canvas = document.getElementById("myChart")
  var container = document.getElementById("chartContainer")
  function onComplete () {
    window.pngData = canvas.toDataURL();
  }
  var config=!{chartSpec};
  config.options.animation = {duration: 0, onComplete: onComplete}
  if (config.options.width) {
    canvas.style.width = config.options.width
    container.style.width = config.options.width
  }
  if (config.options.height) {
    canvas.style.height = config.options.height
    container.style.height = config.options.height
  }
  
  var ctx = document.getElementById("myChart").getContext('2d');
  var myChart = new Chart(ctx, config)
  
```

</details>

### mermaidToSvg

mermaid 变 Svg

``` js 
exports.mermaidToSvg = async function (mermaidPath, page) {
  var mermaidSpec = fs.readFileSync(mermaidPath, 'utf8') // 内容
  var html = formatTemplate('mermaid', { mermaidSpec }) // 获取库
  await page.setContent(html) // 赛进入
  await page.waitForSelector('#graph svg') // 等到 发生
  var svg = await page.evaluate(function () {
    var el = document.querySelector('#graph svg')
    el.removeAttribute('height')
    el.classList.add('mermaid-svg')
    return el.outerHTML
  })
  var svgPath = mermaidPath.substr(0, mermaidPath.lastIndexOf('.')) + '.svg'
  await writeFile(svgPath, svg) // 写入相关路径
}
```

`termplates/mermaid.pug`

<details>

``` pug
script(src="https://cdnjs.cloudflare.com/ajax/libs/mermaid/7.1.2/mermaid.min.js")
#graph.mermaid #{mermaidSpec}


```

</details>


### flowchartToSvg

flowchart 变 Svg

``` js 
exports.flowchartToSvg = async function (flowchartPath, page) {
  var flowchartSpec = fs.readFileSync(flowchartPath, 'utf8') // 读
  var flowchartConf = '{}'
  var possibleConfs = [
    path.join(path.resolve(flowchartPath, '..'), 'flowchart.default.json'),
    flowchartPath + '.json'
  ]
  for (var myPath of possibleConfs) {
    if (fs.existsSync(myPath)) {
      flowchartConf = fs.readFileSync(myPath, 'utf8')
    }
  }
  var html = formatTemplate('flowchart', { flowchartSpec, flowchartConf })
  // console.log(html)
  await page.setContent(html)
  await page.waitForSelector('#chart svg')
  var svg = await page.evaluate(function () {
    var el = document.querySelector('#chart svg')
    el.removeAttribute('height')
    el.removeAttribute('width')
    el.classList.add('flowchart-svg')
    return el.outerHTML
  })
  var svgPath = flowchartPath.substr(0, flowchartPath.lastIndexOf('.')) + '.svg'
  await writeFile(svgPath, svg) // 写入香港
}
```


`termplates/flowchart.pug`

<details>

``` pug
script(src='http://cdnjs.cloudflare.com/ajax/libs/raphael/2.2.0/raphael-min.js')
script(src='http://cdnjs.cloudflare.com/ajax/libs/jquery/1.11.0/jquery.min.js')
script(src='http://flowchart.js.org/flowchart-latest.js')
script.
  $(document).ready(function() {
    var chart = flowchart.parse(`!{flowchartSpec}`)
    var conf = {'symbols': []}
    for (var c of ['start', 'end', 'operation', 'subroutine', 'condition', 'inputoutput']) {
      conf.symbols[c] = {'class': c + '-element'}  
    }
    conf = Object.assign({}, conf, !{flowchartConf})
    console.log(conf)
    chart.drawSVG('chart', conf)
  });
body
  #chart

```

</details>


### vegaliteToSvg

vegalite 变 Svg

``` js 
exports.vegaliteToSvg = async function (vegalitePath, page) {
  var vegaliteSpec = fs.readFileSync(vegalitePath, 'utf8') // 读
  var html = formatTemplate('vegalite', { vegaliteSpec }) // 引入
  // var tempHTML = vegalitePath + '.htm'
  // await writeFile(tempHTML, html)
  // await page.goto('file:' + tempHTML);
  await page.setContent(html)
  await page.waitForSelector('#vis svg')
  var svg = await page.evaluate(function () { //
    var el = document.querySelector('#vis svg')
    el.removeAttribute('height')
    el.removeAttribute('width')
    return el.outerHTML
  })
  var svgPath = vegalitePath.substr(0, vegalitePath.length - '.vegalite.json'.length) + '.svg'
  await writeFile(svgPath, svg) // 写
}

```

`termplates/vegalite.pug`

<details>

``` pug
script(src='https://cdn.jsdelivr.net/npm/vega@3.0.10')
script(src='https://cdn.jsdelivr.net/npm/vega-lite@2.1.3')
script(src='https://cdn.jsdelivr.net/npm/vega-embed@3.0.0')
#vis
script(type='text/javascript').
  vegaEmbed('#vis', !{vegaliteSpec}, {'renderer': 'svg'});

```


</details>

### tableToPug

`csv` -> `table` 变 `Pug`

``` js 
exports.tableToPug = function (tablePath) {
  var extension, header
  var rows = []
  csv({noheader: true}) // 设置
    .fromFile(tablePath)
    .on('csv', (csvRow) => { rows.push(csvRow) })
    .on('done', (error) => {
      if (error) {
        console.log('error', error)
      } else {
        if (tablePath.endsWith('.htable.csv')) {
          extension = '.htable.csv'
          header = rows.shift()
        } else {
          extension = '.table.csv'
          header = null
        }
        var html = formatTemplate('table', { header: header, tbody: rows }) // 读
        var pugPath = tablePath.substr(0, tablePath.length - extension.length) + '.pug'
        html2jade.convertHtml(html, {bodyless: true}, function (err, jade) {
          if (err) {
            console.log(err)
          }
          writeFile(pugPath, jade)
        })
      }
    })
}

```

`termplates/table.pug`

<details>

``` pug
if header
  thead
    for th in header
      th= th
tbody
  for tr in tbody
    tr
      for td in tr
        td= td

```

</details>

### parseDataUrl

图片 路径 获取

``` js
function parseDataUrl (dataUrl) {
  // from https://intoli.com/blog/saving-images/
  const matches = dataUrl.match(/^data:(.+);base64,(.+)$/);
  if (matches.length !== 3) {
    throw new Error('Could not parse data URL.');
  }
  return { mime: matches[1], buffer: Buffer.from(matches[2], 'base64') };
};
```

### chartjsToPNG

chartjs 变成 PNG

```  js
exports.chartjsToPNG = async function (chartjsPath, page) {
  var chartSpec = fs.readFileSync(chartjsPath, 'utf8') // 读
  var html = formatTemplate('chartjs', { chartSpec }) // 引
  var tempHTML = chartjsPath + '.htm'
  await writeFile(tempHTML, html)
  // await page.goto('file:' + tempHTML);
  await page.setContent(html) // 塞
  await page.waitForFunction(() => window.pngData)
  const dataUrl = await page.evaluate(() => window.pngData)
  const { buffer } = parseDataUrl(dataUrl)
  var pngPath = chartjsPath.substr(0, chartjsPath.length - '.chart.js'.length) + '.png'
  await writeFile(pngPath, buffer, 'base64') // 写
}
```


`termplates/chartjs.pug`

<details>

``` pug
script(src='https://cdnjs.cloudflare.com/ajax/libs/Chart.js/2.7.2/Chart.min.js')
#chartContainer
  canvas#myChart
script.
  window.pngData = false
  var canvas = document.getElementById("myChart")
  var container = document.getElementById("chartContainer")
  function onComplete () {
    window.pngData = canvas.toDataURL();
  }
  var config=!{chartSpec};
  config.options.animation = {duration: 0, onComplete: onComplete}
  if (config.options.width) {
    canvas.style.width = config.options.width
    container.style.width = config.options.width
  }
  if (config.options.height) {
    canvas.style.height = config.options.height
    container.style.height = config.options.height
  }
  
  var ctx = document.getElementById("myChart").getContext('2d');
  var myChart = new Chart(ctx, config)
  
```

</details>

### asyncMathjax

异步 预处理 `html` 中 `TeX`

``` js
function asyncMathjax (html) {
  return new Promise(resolve => {
    mjpage.mjpage(html, {
      format: ['TeX'] // TeX
    }, {
      mml: true,
      css: true,
      html: true
//   mml ： false，//生成mml输出？
//   CSS ： false，//为HTML输出生成CSS？
//   html ： false 的算术数值，//生成HTML输出？
    }, response => resolve(response))
  })
}
```

### getMatch

匹配

``` js
function getMatch (string, query) {
  var result = string.match(query)
  if (result) {
    result = result[1]
  }
  return result
}
```

### masterDocumentToPDF

```  js
                                // (输入完整路径, 浏览器APi, 缓存完整路径, 输出完整路径)
exports.masterDocumentToPDF = async function (masterPath, page, tempHTML, outputPath) {
  var html
  if (masterPath.endsWith('.pug')) {
    try {
      html = pug.renderFile(masterPath)
    } catch (error) {
      console.log(error.message)
      console.error('There was a Pug error (see above)'.red)
      return
    }
  } else {
    html = fs.readFileSync(masterPath, 'utf8')
  }
  // 获得 html 格式 文件数据

  html = await asyncMathjax(html) // 处理 html 中 TeX 数字符号 // 这里有Bug

  var parsedHtml = cheerio.load(html) // 将 html 变成 JQuery类可操作格式
  html = parsedHtml.html() // adds html, body, head.
  var headerTemplate = parsedHtml('template.header').html()
  var footerTemplate = parsedHtml('template.footer').html()
// await page.setContent(html)
  await writeFile(tempHTML, html) // 写入缓存

  await page.goto('file:' + tempHTML, {waitUntil: 'networkidle2'});
  // await page.waitForNavigation({ waitUntil: 'networkidle2' })
// https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md#pagegoforwardoptions 
// networkidle2- 当网络连接数不超过2个时，导航至少需要500ms

  var options = {
    path: outputPath, // 输出
    displayHeaderFooter: headerTemplate || footerTemplate, // 显示页眉和页脚。默认为false。
    headerTemplate,
    footerTemplate,
    printBackground: true // 打印背景图形。默认为false。
  }
  var width = getMatch(html, /-relaxed-page-width: (\S+);/m)
  if (width) { 
      // 宽
    options.width = width
  }
  var height = getMatch(html, /-relaxed-page-height: (\S+);/m)
  if (height) { 
      // 高
    options.height = height
  }
  var size = getMatch(html, /-relaxed-page-size: (\S+);/m)
  if (size) { // 大小
    options.size = size
  }
  await page.pdf(options)
// https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md#pagepdfoptions
}

```