# DemoProjects 
聽說這年頭找工作談外包之類的要有git專案之類的咚咚才會有人理你！

問題是我的程式要嘛放在客戶他們自己的server，要嘛就是幫他們用gitea架自己的server

所以我這邊一片漆黑......因此我花點時間做了些demo，架構大概是這樣的
跨越前端、後端與全端的技術展示專案集，涵蓋主流框架與語言，每個專案皆具備 CI/CD 自動化流程與測試覆蓋，雖說是demo但是是認真的demo。
---

## 技能樹總攬

| 領域 | 技術 |
|------|------|
| 後端 | Java (Spring Boot)、C# (.NET)、PHP (Laravel)、Python (FastAPI)、Node.js (NestJS) |
| 前端 | Angular、React、Vue.js、Next.js |
| 行動端 | Flutter、Kotlin (Android)、Swift (iOS) |
| 資料庫 | Oracle、SQL Server、MySQL、PostgreSQL |
| DevOps | GitHub Actions CI/CD、Docker、AWS Lightsail、Nginx |

---

## 專案架構

本 Repo 以 Git Submodule 組織，依前端 / 後端 / 全端分類，方便獨立維護與展示。

```
DemoProjects/                    # 主 repo
├── 文件/
│   └── SA之類的文件
└── 原始碼/
    ├── APP端/
    │   ├── 各種APP端專案        # submodule
    ├── 前端/
    │   ├── 各種前端專案        # submodule
    ├── 後端/
    │   ├── 各種後端專案        # submodule    
    └── 全端/
        └── 各種全端專案        # submodule


```
每個專案均遵循以下工程標準：

- **CI/CD 自動化**：GitHub Actions 建置、測試與部署流程，全線綠燈 ✅
- **測試覆蓋**：單元測試與整合測試
- **程式碼品質**：Linter 檢查與一致的程式碼風格
- **文件完整**：API 文件（Swagger/OpenAPI）與專案說明


目前進度

全端/

├── django-demo    ![CI](https://github.com/kamesan634/django-demo/actions/workflows/ci.yml/badge.svg)

├── dotnet-demo    ![CI](https://github.com/kamesan634/dotnet-demo/actions/workflows/ci.yml/badge.svg)

├── laravel-demo    ![CI](https://github.com/kamesan634/laravel-demo/actions/workflows/ci.yml/badge.svg)

├── nextjs-demo    ![CI](https://github.com/kamesan634/nextjs-demo/actions/workflows/ci.yml/badge.svg)

└── springboot-demo    ![CI](https://github.com/kamesan634/springboot-demo/actions/workflows/ci.yml/badge.svg)


後端/

├── dotnetapi-demo    ![CI](https://github.com/kamesan634/dotnetapi-demo/actions/workflows/ci.yml/badge.svg)

├── fastpi-demo    ![CI](https://github.com/kamesan634/fastapi-demo/actions/workflows/ci.yml/badge.svg)

├── laravelapi-demo    ![CI](https://github.com/kamesan634/laravelapi-demo/actions/workflows/ci.yml/badge.svg)

├── go-demo    ![CI](https://github.com/kamesan634/go-demo/actions/workflows/ci.yml/badge.svg)

├── nestjs-demo    ![CI](https://github.com/kamesan634/nestjs-demo/actions/workflows/ci.yml/badge.svg)

└── springbootapi-demo    ![CI](https://github.com/kamesan634/springbootapi-demo/actions/workflows/ci.yml/badge.svg)


前端/

├── angular-demo    ![CI](https://github.com/kamesan634/angular-demo/actions/workflows/ci.yml/badge.svg)

├── react-demo    ![CI](https://github.com/kamesan634/react-demo/actions/workflows/ci.yml/badge.svg)

└── vue-demo    ![CI](https://github.com/kamesan634/vue-demo/actions/workflows/ci.yml/badge.svg)


Java 與 .NET 是我一開始的專長，然後到處打工才開始點新的技能樹咩

一開始出來賺，才發現其實外面頗多要用 PHP 的，所以就順勢點亮，不過後來我比較少接到PHP的案子說

然後Python橫空出世，但他的寫法會讓我有點小困擾，本想保持距離無奈遇到恩客拿新台幣脅迫我去學，沒想到後來甚至去職訓課程中擔任過一期講師

有一天睡醒突然發現世界變了！出現了前端這個詞，還自帶御三家，當我還在摸索另一個語言時，有位恩客拿錢砸我說：別鬧了，就選Angular

御三家最不想碰的就是Vue，因為聽說是個競爭激烈超級紅海？沒想到這個紅海這麼大，近年還滿常遇到它的！

人生就是那麼奇妙，御三家第一個想學的卻最後才遇到恩客(笑

羨慕硬體有乖乖！綠燈怎麼架雞掰！(跌坐

有一天合作過的PM臨時抓我去頂某專案的後端！當我施展springboot召喚術到一半時....PM：咦？我沒說是Nestjs嗎？

2023年我接了一個最荒謬也算虧本的專案，唯一沒那麼遺憾的就是學了Next.js

2024打開求職網，突然發現了好多人要go，沒錯！我是來蹭的，沒參與過專案的我自覺得學得沒很好，有好心的老爺賞口go飯吃嗎？讓我補足短版(笑

技能樹上的其他分枝會陸續更新上來的！
