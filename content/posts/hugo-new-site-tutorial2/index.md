---
date: "2025-03-04T10:16:11+08:00"
draft: false
title: "Hugo 架站教學 - 主題設定"
description: Hugo 主題設置 - 個人化自己的網站
categories:
  - WebSite
tags:
  - Hugo
  - Hugo-Theme
  - GitHub Pages
  - git
image: https://user-images.githubusercontent.com/5889006/190859441-141b5f81-8483-40d2-bd96-ebf85616a46d.png
links:
  - title: Hugo-sample-site
    description: GitHub 專案源始碼
    website: https://github.com/maydayXi/Hugo-sample-site
    image: https://github.githubassets.com/images/modules/logos_page/GitHub-Mark.png
---

# 前情提要

上一篇 **_[Hugo 架站教學 - 建立網站](/posts/hugo-new-site-tutorial)_** 說明如何新增網站，在本機編譯及上傳版本控制。主題設定也是我碰到最多問題的地方，所以本篇會著重在**系統主題設定，並且進行網站佈署**。

步驟上沒有先後，可以先佈署在做主題設定，也可以先設定主題在佈署，我採用後者。

# 重點

1. 主題設定
2. 佈署網站

# Visual Studio Code

在配置 Hugo 主題前，需要一個編輯器來調整網站的相關設定。我推薦用 **_[Visual Studio Code（VS Code）](https://code.visualstudio.com/)_** 編輯檔案。它輕量、免費，支援 Markdown 和 TOML，適合 Hugo 專案。加裝 **_[hugo 相關套件](https://gohugo.io/tools/editors/)_** 擴充方便。但想使用其他編輯器也行。

# 開啟專案

1. 開啟 Visual Studio Code，開啟終端機「**control + `(數字 1 的左邊)**」

![Visual Studio Code terminal](vscode-terninal.png)

2. 切換 HugoSampleSite 目錄

```shell
cd Documents/HugoSampleCode
```

3. 開啟 HugoSampleCode 專案

**-r：reuse this window，使用目前的視窗**

**.：目前的目錄**

如果沒有 -r，vscode 會在新的視窗開啟專案，整行指令的意思是說，用目前 vscode 視窗開啟目前所在目錄檔案

```shell
code -r .
```

![Visual Studio Code open HugoSampleSite](vscode-open-hugo-sample-site.png)

開啟後可以看到左邊 **檔案總管** 有完整的專案目錄

# 主題設定

主題設定參考下列兩個網站

1. **_[Hugo 官網](https://gohugo.io/)_**
2. **_[Stack 主題官網](https://stack.jimmycai.com/config/)_**

開啟目錄下的 `hugo.toml`，這個檔案就是網站的設定檔，開啟後會看到之前設定所套用的主題名稱。

```toml
theme = 'hugo-theme-stack'
```

![hugo.toml](hugo.toml.png)

到 **_[Stack 主題官網設定說明](https://stack.jimmycai.com/config/)_**，可以看到設定檔接受 3 種格式，可以選擇自己喜歡或習慣的格式來調整。

1. `TOML`：`.toml`
2. `YAML`：`.yaml`
3. `JSON`：`.json`

由於我比較習慣 yaml，所以我會先把 toml 轉成 toml，網路上有很多轉換工具，如 **_[https://transform.tools/toml-to-yaml](https://transform.tools/toml-to-yaml)_** 將設定檔內容做轉換，並將 `hugo.toml` 存成 `hugo.yaml`

![hugo.toml to hugo.yaml](toml-to-yaml.png)

## 預設的設定

建立網站時的初始設定

### baseUrl

網站的網址，本機開發不需要去改它，如果之後買自己的網址才需要去更改。

參考 **_[hugo baseUrl configuration](https://gohugo.io/getting-started/configuration/#baseurl)_**

### languageCode

網站的語系編碼，這個下面的設定會調整。參考 **_[語系設定](#語系設定)_**

### title

網站的標題，也就是**標籤上顯示的標題**，這邊改成 HugoSampleSite。

`hugo.yaml`

```yaml
title: HugoSampleSite
```

並在終端機將本機網站執行

```shell
hugo server
```

![HugoSampleSite Title](hugo-sample-site-title.png)

### theme

套用的主題名稱，前一篇已經知道了

## 本機執行

在改完設定後，可以直接儲存，不需要重新啟動本機伺服器，hugo 會即時渲染最新結果

如果想要結束本機執行，在終端機按下「**control + c**」

## 語系設定

下面有兩種設定方式可以擇一設定，如果設定了可以刪除初始的語系設定。

### Stack i18n

Stack 主題使用 **_[i18n](https://stack.jimmycai.com/config/i18n)_** 來設定網站的語系，在 `hugo.yaml` 加入繁體中文設定，儲存。
如果還在執行模式就可以切換到瀏覽器查看最新結果

```yaml
# 網站語系設定
# 參考 https://stack.jimmycai.com/config/i18n
DefaultContentLanguage: zh-tw
```

![HugoSampleSite 中文](hugo-sample-site-zh-tw.png)

### Hugo 內建語系設定

在 **_[Hugo 內建的語系設定](https://gohugo.io/content-management/multilingual/)_** 會發現 languageCode 的設定結構不太一樣，如果有需要更細的設定或其他客製需求，可以把 **_[Configure languages](https://gohugo.io/content-management/multilingual/#configure-languages)_** 的設定值複製下來，改成中文設定。

相關設定如下

- disabled：不翻譯網站內容，預設是啟用翻譯功能
- languageCode：網頁內容語系編碼 zh-tw（臺灣繁體中文），你會發現上面初始設定有一個一樣的設定，可以刪除初始設定的語系。
- languageName：語言名稱
- title：可以針對這個語系設定標題
- weight：語系的排序
- languageDirection：文字閱讀方向，左到右或右到左，不需要可以刪除，採網頁預設的方向。

另外要注意的是，如果是用 Hugo 內建的語系設定方法，需要提供 **defaultContentLanguage** 這個設定，因為 Hugo 預設是英文，要將它設定成 **zh-tw**

```yaml
defaultContentLanguage: zh-tw
languages:
  zh-tw:
    disabled: false
    languageCode: "zh-tw"
    languageName: "中文"
    title: "示範網站"
    weight: 1
# 如果需要其他語系設定，可以再複製 zh-tw 再修改成其他語系
# en:
#   disabled: false
#   languageCode: "en"
#   languageName: "English"
#   title: "HugoSampleSite"
#   weight: 2
```

### 總結

最後設定完，設定檔 `hugo.yaml` 會長這樣

```yaml
baseURL: https://example.org/
title: HugoSampleSite
theme: hugo-theme-stack
# 網站語系設定
# 參考 https://stack.jimmycai.com/config/i18n
DefaultContentLanguage: zh-tw
# 參考 https://gohugo.io/content-management/multilingual/#configure-languages
# 因為 Hugo 預設的系統語言是英文，所以需要將預設的語言改成中文
# defaultContentLanguage: zh-tw
# languages:
#   zh-tw:
#     disabled: false
#     weight: 1
#     languageCode: zh-tw
#     languageName: 中文
#     title: "示範網站"
```
