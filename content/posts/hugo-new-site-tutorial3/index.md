---
date: "2025-03-06T11:52:35+08:00"
draft: false
title: Hugo 架站教學 - PO 文撰寫
description: 輕鬆創建你的部落格內容
categories:
  - WebSite
tags:
  - Hugo
  - Markdown
image: markdown-featured-image.png
---

# 前情提要

終於來到 Hugo 架站 3 部曲的最後一部了，前兩篇說明了「建立網站」、「設定主題」但沒有網站內容，本篇會聚焦在 po 文撰寫，一步步豐富部落格，完成架設網站的最後一片拼圖。

# 本篇重點

1. Markdown
2. Front Matter
3. PO 文目錄架構

# Markdown

**_[Markdown](https://zh.wikipedia.org/zh-tw/Markdown)_** 是 Hugo 頁面和文章的基礎所有的頁面和創作內容，都要使用它，並在最後透過 Hugo 的編譯器，將 Markdown 編成 HTML。

## Hugo 的 Markdown 語法

基本用法，可以在 **_[Markdown Playground](https://dotmd-editor.vercel.app/)_** 測試結果

1. 標題
2. 清單（編號、無編號）
3. 程式碼區塊
4. 字型樣式（粗體、斜體）
5. 超連結
6. 圖片

### 標題

用「#」表示標題，通常只會用到 h2 ~ h4，因為 Hugo 預設只會顯示，\#\# ~ \#\#\#\# 的標題，如果要像我一樣顯示 \# 的標題需要在設定檔中額外設定

```markdown
# h1 標題

## h2 標題

### h3 標題

#### h4 標題

##### h5 標題

###### h6 標題
```

### 清單

使用「-」產生無編號清單，「-」與文字間要有一個空格（空白鍵）；使用「數字」產生有編號清號，與文字一樣需要空一格。

- 項目一
- 項目二
- 項目三

1. 編號 1
2. 編號 2
3. 編號 3

```markdown
- 項目一
- 項目二
- 項目三

1. 編號 1
2. 編號 2
3. 編號 3
```

### 程式碼區塊

使用「**\`\`\`**」共 3 個「\`（數字 1 號鍵左邊）」，然後接程式語言的名稱不需要空一格，如 \`\`\`Javascript，最後在要結束的地方在用三個\`\`\`表示結束。

\`\`\`javascript \
console.log("Hello world");\
\`\`\`

```javascript
console.log("Hello world");
```

### 字型樣式

有兩種

1. 粗體：兩個星號「\*\*」
2. 斜體：一個底線「\_」
3. 粗斜體：1 + 2\
   把要套用的字包起來

**粗體 - 兩個星號**\
_斜體 - 一個底線_\
**_粗斜體 - 兩個星號一個底線_**

```markdown
**粗體 - 兩個星號**
_斜體 - 一個底線_
**_粗斜體 - 兩個星號一個底線_**
```

### 超連結

**\[超連結顯示的文字\]\(連結網址\)**，以 Google 為例\
[Google](https://www.google.com/)

```markdown
[Google](https://www.google.com/)
```

### 圖片

跟超連結很像，在最前面加上「!」而已\
**\!\[圖片載入失敗要顯示的文字\]\(圖片網址\)**\
以套用的主題圖片為例

![Hugo-Theme-Stack](https://user-images.githubusercontent.com/5889006/190859441-141b5f81-8483-40d2-bd96-ebf85616a46d.png)

```markdown
![Hugo-Theme-Stack](https://user-images.githubusercontent.com/5889006/190859441-141b5f81-8483-40d2-bd96-ebf85616a46d.png)
```

## Markdown 實踐

上面紹介的幾個是寫文章比較常用到的，現在利用上一篇所產生的關於頁面來寫部落格的第一個內容。

※ Markdown 其實還有很多語法，不過 Hugo 沒有全部支援，使用上面幾個應該就夠了，其它寫法可以自己研究

開啟 vscode 再開啟 `content/page/about/index.md`，會看到之前寫好的內容，「**- - -**」包起來的內容叫 **Front Matter**，就是 About 這一頁的設定，下面的章結會介紹，本節主要是 Markdown 語法

![HugoSampleSite About](hugo-sample-site-about.png)

### 新增標題及內容

```markdown
## 關於

這是一個使用 Hugo 建立的範例網站，作為教學用，主要在幫助使用者了解 Hugo 的基本操作及功能，並一步步建立網站，設定主題。無論是新手或有經驗的開發者，這個網站都可以作為一個參考，並且讓你實際建立出自己的網站。
```

將「教學用」改為粗體後儲存

```markdown
**教學用**
```

接著運行網站，在終端機輸入，網站啟動

```shell
hugo server
```

在左邊選單點擊「關於」，看結果

![HugoSampleSite About](hugo-sample-site-about-intro.png)

### 新增主題段落

這裡會注意到網站有設定顯示目錄，但是目錄沒有出現，這是因為這個主題的設定，**要有兩個以上的二級標題才會顯示目錄**，所以我目在新增一個主題的段落。

```markdown
## 網站主題

網站主題套用 Hugo-Theme-Stack
```

#### 加入超連結

這邊把「Hugo-Theme-Stack」改成超連結，並貼上超連結網址「https://themes.gohugo.io/themes/hugo-theme-stack/」

```markdown
[Hugo-Theme-Stack](https://themes.gohugo.io/themes/hugo-theme-stack/)
```

Hugo 是即時渲染，只要在網站執行的情況下，儲存後網站就會是最新的狀態，不需要重新執行，所以直接切回瀏覽器就可以看到最新結果，可以看到因為有 2 個段落了，所以目錄出現了，接著可以點擊超連結確認是否會連到主題的網站。

![HugoSampleSite Hyperlink](hugo-sample-site-about-theme.png)

#### 改變超連結字型樣式

這一步可以不用做，但如果覺得超連結不夠醒目，可以再變成粗體或斜，範例示範粗斜體

```markdown
**_[Hugo-Theme-Stack](https://themes.gohugo.io/themes/hugo-theme-stack/)_**
```

![HugoSampleSite Hyperlink applied font bold](hugo-sample-site-hyperlink-fontbold.png)

#### 加入主題圖片

接下來加入主題的圖片，網址「https://user-images.githubusercontent.com/5889006/190859441-141b5f81-8483-40d2-bd96-ebf85616a46d.png」，用 **\[圖片載入失敗顯示的文字\]\(圖片的連結\)**\
要特別注意 **Markdown 語法兩個 Enter 會渲染成一個換行，所以要跟上面的連結空一行**

```markdown
<!-- 假設這裡是超連結 -->

![Hugo-Theme-Stack](https://user-images.githubusercontent.com/5889006/190859441-141b5f81-8483-40d2-bd96-ebf85616a46d.png)
```

儲存後切回瀏覽器會看到主題圖片出現了

![HugoSampleSite image](hugo-sample-site-theme-image.png)

### 新增參考網站段落

最後新增「參考網站」的區塊，練習清單做法，下面加入 4 個參考網站，可以選擇使用編號或不編號的清單，這裡示範編號清單

1. Hugo 的官方網站：https://gohugo.io/
2. Theme 的官方網站：https://stack.jimmycai.com/
3. Theme 的 GitHub：https://github.com/CaiJimmy/hugo-theme-stack/tree/master
4. 本網站的 GitHub：https://github.com/maydayXi/Hugo-sample-site

```markdown
## 參考網站

1. [Hugo 官方網站](https://gohugo.io/)
2. [Hugo-Theme-Stack 官方網站](https://stack.jimmycai.com/)
3. [Hugo-Theme-Stack GitHub](https://github.com/CaiJimmy/hugo-theme-stack/tree/master)
4. [HugoSampleSite](https://github.com/maydayXi/Hugo-sample-site)
```

切回瀏覽器後就會看到清單項目編號出現，Markdown 語法綀習就到這邊。

![HugoSampleSite list](hugo-sample-site-list.png)

### 關於的 FrontMatter

Markdown 寫完後，基本上你已經可以寫出一頁網頁了，不過你會發現只有閱讀時間的資訊，在 **_[Markdown 實踐](#markdown-實踐)_** 有提到 Front Matter，這些頁面上的資訊，其實就是 Front Matter 做出來的，這邊先說明其中常用的 3 個，接著套用的關於這一頁上

1. **title：這一頁的標題，不是內文的標題**
2. **date：這一頁建立的時間**
3. **description：這一頁的副標題**

參考下面進行相關設定，**要特別注意 Front Matter 區塊一定要在最上面，然後用 3 個「-」號包起來**

```markdown
---
title: 關於本站
date: 2025-03-06
description: 網站的簡介
# 上一篇建立的選單連結
menu:
  main:
    name: 關於
    weight: 2
    params:
      icon: user
---

<!-- 其它標題及內容 -->
```

設定完後儲存，切回瀏覽器，就會看到出現了相關的頁面資訊，關於就完成了，可以結束執行並上傳程式版本到 GitHub，並觀察佈署的情況，確認佈署完後驗證網頁是否有上傳成功。

![HugoSampleSite final](hugo-sample-site-final.png)

## Front Matter

接下來要詳細的說明 Front Matter，第一篇建立網站的時候，有一個網頁設定檔 `hugo.yaml`，**所以可以把 Front Matter 想成是單一網頁的設定檔，而它也有特定的格式，預設是 toml（用 +++ 區隔），但也可以改成 yaml（用- - - 區隔），由於是寫在 Markdown 檔案中，所以要區隔開來**，因為我比較的慣用 yaml，所以之後都會用 yaml 的格式寫

### Front Matter 預設值

每使用 **hugo new content** 新增一篇 PO 文，都會看到 1 檔案中有預設的設定，這是可以改的，在網站的 `archetypes/default.md` 檔，開啟後就會看到預設的設定

```markdown
+++
date = '{{ .Date }}'
draft = true
title = '{{ replace .File.ContentBaseName "-" " " | title }}'
+++
```

先把它改成下面的樣子，讓它變成 `yaml` 的格式

```markdown
---
date: "{{ .Date }}"
draft: true
title: '{{ replace .File.ContentBaseName "-" " " | title }}'
---
```

這裡先說明一個屬性 **draft** 就是草稿，**hugo 不會產生出草稿狀態的網頁**，所以如果想要發佈某個文章的話，需要將它改成 **false**，下面設定讓它一開始就不是草稿狀態

```markdown
date: "{{ .Date }}"
draft: false
title: '{{ replace .File.ContentBaseName "-" " " | title }}'
```

### Front Matter 其他設定

再來看看其它設定，下面說明我常用到的 3 個

1. **categories： PO 文的分類**
2. **tags: PO 文的標籤**
3. **image: PO 文的縮圖**

可以發現上面這三個設定，在第一篇網站設定裡也有相關的設定，**其中第 3 個 image 對應到的就是在 `hugo.yaml` 中的 featuredImageField**

接下來將上面的 3 個設定寫入 `default.md` 裡，預設都先不給值，因為在寫 PO 文的時候才會知道文章要分在哪一類，給哪一個標籤。

```markdown
---
date: "{{ .Date }}"
draft: false
title: '{{ replace .File.ContentBaseName "-" " " | title }}'
categories:
tags:
image:
---
```

好了就可以儲存了，接著就要開始寫第一篇 PO 文了。這裡我再一次上版。
