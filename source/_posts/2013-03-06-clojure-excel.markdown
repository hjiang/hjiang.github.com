---
layout: post
title: "用 Clojure 处理 Excel 数据"
date: 2013-03-06 01:25
comments: true
categories: 
---

最近遇到一个小问题。我们每个月会用一个 Excel 表格记录当月的员工工资。假设有30个员工，那么一年下来就会有12个表格，每个表格有30行。由于中国规定年薪超过12万的人都要申报缴税情况，我们需要提供给每个员工他全年在公司的收入明细。也就是说我们需要生成30个表格，每个表格有12行。组织数据的维度变了。这个转化可以请负责 HR 的同事手工完成，但是一来耗时太久，二来这也是不符合我司风格的。这里介绍一个简单办法，也留着给自己备忘。

<!-- more -->

以前曾经用 Ruby 处理过 Excel 表格，很方便，但是这次用 Clojure 代码更精简了很多。

唯一用到的第三方库是 Incanter

    (ns salaries.core
      (:use clojure.tools.trace
            [clojure.java.io :only [file]]
            [incanter.excel :only [read-xls save-xls]]
            [incanter.core :only [to-dataset]]))

主函数读取当前目录下的所有 Excel 文件，并用 `reduce` 逐一处理每个文件，得到一个从姓名到薪酬明细的 `map`, 然后把每个人的薪酬数据保存到 Excel 文件里：

    (defn -main []
      (let [xlsx-files (map (memfn getName)
                            (filter (fn [f] (.endsWith (.getName f) ".xlsx"))
                                    (file-seq (file "."))))]
        (doseq [[name rows] (reduce proc-file {} xlsx-files)]
          (save-xls (to-dataset rows) (str name "_generated.xlsx")))))
      
`proc-file` 是处理每个文件的函数，它再用一次 `reduce` 处理每一行：

    (defn proc-file [people file]
      (let [data (read-xls file)
            rows (take-while #(string? (get % "姓名")) (drop 1 (:rows data)))]
        (reduce proc-row people rows)))

其中的 `(drop 1 (:rows data))` 是为了去掉表头。

对每一行的处理就很简单了：

    (defn proc-row [people row]
      (let [name (get row "姓名")
            person (or (get people name) [])
            data (select-keys row ["姓名","税","本月实发",
                                   "应税金额","工资","小计","当月增/减",
                                   "病事假","税后扣款"])]
        (assoc people name (conj person data))))