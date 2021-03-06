---
title: 半自动国税发票查验
author: Situ
layout: post
categories: [work]
tags: [工作实用,爬虫]
---
<font face="仿宋" >“半自动”指验证码还需手动填写o(╥﹏╥)o</font>

<script src="https://cdn.jsdelivr.net/gh/google/code-prettify@master/loader/run_prettify.js"></script>

## 1.0版本
### 代码步骤
#### 1. 手动填写发票信息
发票信息如发票代码、发票号码、发票时间、税前金额需先行填写在excel中，如[invoice_sample.xlsx](https://github.com/Snowing-ST/invoice_checking/blob/master/invoice_sample.xlsx)所示


|业务编号|	发票代码|	发票号码|	发票日期|	税前金额|
|  ----  | ----  | ----  | ----  | ----  |
|ABC2019FP00004|	3400183130|	02049280|	20181225	|471495.24 |
|ABC2019FP00004|	3400183130|	02049281	|20181225|	937220.75 |

<pre class="prettyprint"><code class="language-py">
import os
import pandas as pd
# 建立截图、与存放截图的文档文件夹
if not os.path.exists(screenshot_save_path):
    os.mkdir(screenshot_save_path) 
if not os.path.exists(doc_save_path):
    os.mkdir(doc_save_path) 
#直接默认读取到这个Excel的第一个表单
invoice_info = pd.read_excel(invoice_file_path,dtype="str")
invoice_info.head()
</code></pre>

#### 2. 自动填写表单
运行[invoice_checking.py](https://github.com/Snowing-ST/invoice_checking/blob/master/invoice_checking.py),提示填写excel表格的路径，填写完毕回车后，selenium库的webdriver将打开谷歌浏览器，自动登陆[国家税务总局全国增值税发票查验平台](https://inv-veri.chinatax.gov.cn/index.html)，自动填写发票信息，**验证码需手动输入程序端，不能输在网页上**，如果输错等待浏览器自动跳转，**不要点击浏览器任何按钮**，再次在程序端输入验证码。

- <font size=2 > 注：使用selenium库的webdriver打开谷歌浏览器需到[此处](http://chromedriver.storage.googleapis.com/index.html)下载chromedriver.exe，注意下载版本一定要和电脑中谷歌浏览器版本一致！并将该文件放入excel文件的目录下（可在代码中修改存放路径）</font>

<pre class="prettyprint"><code class="language-py">
from selenium import webdriver
from selenium.webdriver.common.action_chains import ActionChains
from selenium.common.exceptions import StaleElementReferenceException

## 启动web driver
driver = webdriver.Chrome(os.path.join(path,"chromedriver.exe"))
driver.maximize_window()

# 手动输入验证码
def yzm(driver,invoice_num):
    print("正在查验发票%s..." % invoice_num)
    code = input("请输入验证码:")
    driver.find_element_by_id('yzm').send_keys(code)
    driver.find_element_by_id('checkfp').click()
    time.sleep(3)
</code></pre>

#### 3. 自动截图
跳转至发票验证页面后，使用selenium库控制鼠标在页面滚动至最上方，自动截图，以发票号码命名，保存在excel文件的目录下的“截图”文件夹中。

- <font size=2 >注：由于网速等问题网页跳转慢，可能出现截错页面的情况</font>

<pre class="prettyprint"><code class="language-py">
def screen_shot(driver,screenshot_save_path,invoice_num):
    
    time.sleep(3)
    
#    ele = driver.find_element_by_class_name('logo')
    # 将鼠标移动到定位的元素上面
#    ActionChains(driver).move_to_element(ele).perform()
#   以下操作是滚动鼠标将网页移动至最上端，使截图完整
    driver.execute_script("""
    (function () {
        var y = 0;
        var step = 100;
        window.scroll(0, 0);

        function f() {
            if (y < document.body.scrollHeight) {
                y += step;
                window.scroll(0, y);
                setTimeout(f, 100);
            } else {
                window.scroll(0, 0);
                document.title += "scroll-done";
            }
        }

        setTimeout(f, 1000);
    })();
""")  
    driver.save_screenshot(os.path.join(screenshot_save_path,'%s.png' % invoice_num))
    print("发票%s截图成功" % invoice_num)
    time.sleep(2)
</code></pre>

#### 4. 循环查验发票数据
<pre class="prettyprint"><code class="language-py">
for i in range(len(invoice_info)):
    try:
        invoice_code,invoice_num,date,value = [str(ele) for ele in invoice_info.ix[i,1:]]
        invoice_check(driver,invoice_code,invoice_num,date,value,screenshot_save_path)
        time.sleep(1)
    # 查验发票的过程中可能出现如下错误，刷新一下就好了
    except StaleElementReferenceException as msg:
        print("查找元素异常%s，请等待..."%msg)
        print("重新获取元素...")
        driver.refresh()
        i = i+1
driver.close()
print("截图已完成!请打开%s查看，正在将截图插入word文档......" % screenshot_save_path)
</code></pre>
#### 5. 自动生成word文档
所有发票验证截图后，将同一个业务项下的发票插入word文档，以业务编码命名，保存在excel文件的目录下的“文档”文件夹中，方便后续打印操作。

<pre class="prettyprint"><code class="language-py">
from docx import Document
from docx.shared import Inches
#将同一笔业务的截图放入一个word文档中，以业务编号命名
for business_no in invoice_info["业务编号"].value_counts().index:
    invoice_no = invoice_info["发票号码"][invoice_info["业务编号"]==business_no]
    doc = Document()    # doc对象
    doc.add_heading(business_no ,0)

    for invoice_no_i in invoice_no:
        #string = '文字内容'
        images = os.path.join(screenshot_save_path,'%s.png' % invoice_no_i)    # 保存在本地的图片
        
        #doc.add_paragraph(string)   # 添加文字
        doc.add_picture(images,width=Inches(6))     # 添加图, 设置宽度
    doc.save(os.path.join(doc_save_path,'%s.docx' % business_no))     # 保存路径   

print("已将截图插入word文档，请打开%s查看!" % doc_save_path)
</code></pre>

### 打包exe文件
为了让不懂程序的人使用该程序，提高工作效率，本代码还可用pyinstaller打包成exe文件，总文件大概700M。在cmd或者anaconda prompt中输入以下代码：
```
pyinstaller -p C:/.../Anaconda2/envs/py3/Lib/site-packages -D invoice_checking.py
```
    -p 指定python第三方包的路径，将代码依赖的包都打包进去
    -D 产生一个目录（包含多个文件）作为可执行程序
    可执行文件路径：../dist/invoice_checking/invoice_checking.exe

当需要一次性查询大量发票时，该程序优势就显示出来啦~(*^▽^*)~

## 2.0版新特点
- 可直接在python的console中输出验证码图片、提示性文字，以及输入验证码，提高效率
- 识别“查验太频繁，请1分钟后再试”的问题，程序将等待一分钟后自动跳转
- 截图时简化模拟鼠标滚动操作，鼠标仅模拟滑动至最上方

## 3.0版新特点
- 用tkinter包构建图形界面，打包成exe程序后方便不懂代码的人使用，如导入文件时可直接打开文件管理器选择文件，简单直观。
- gui编程的设计思路为事件触发型，因此代码逻辑相比于2.0版本有重大修改，主要是：
    - 1 把获取验证码和提交验证码分为两步； 
    - 2 原2.0逐张验证发票使用for循环，3.0无明显循环的关键字，改为截图后重复一次查验过程，直到已查验发票数i与导入的excel文件中样本数相等； 
    - 3 出错时不再需要while循环，遂删除
    - 4 整个代码写成一个类的形式，大大方便了参数调用

<img src="{{ 'assets/images/post_images/gui.png'| relative_url }}" /> 

获取以上所有代码请点击[此处](https://github.com/Snowing-ST/Invoice-Checking)~(*^▽^*)~。