#spider note


::: tip chrome driver
- chrome driver and chrome version 匹配 [Link](https://chromedriver.chromium.org/downloads)
- headless无界面浏览器模式💦: Headless Chrome 更加方便测试 web 应用，获得网站的截图，做爬虫抓取信息等;
-  使用headless无界面浏览器模式
     ```py
    chrome_options = webdriver.ChromeOptions()
    chrome_options.add_argument('--headless') //增加无界面选项
    chrome_options.add_argument('--disable-gpu') //如果不加这个选项，有时定位会出现问题
    driver = webdriver.Chrome(chrome_options=chrome_options)
    ```
- 结合selenium获取cookie并把cookie交给requests使用来进行登录是比较有帮助的
   ```py
    current_cookies = driver.driver.get_cookies()
    s = requests.Session()
    [s.cookies.set(c['name'], c['value']) for c in current_cookies]
    # 根据浏览器request headers模拟headers dict 数据
    s.get(url, headers=headers)
   ```
   
     
:::

::: tip Driver常用方法
- driver.find_element( By.CSS_SELECTOR/ID, value)
- el.click()
- el.send_keys(value)
- el.text
- el.get_attribute(tagAttr)
:::

::: tip 数据写入CSV
```py
header = ['index', 'date', 'diagnostic', 'dc_name', 'pt_num', 'pt_name', 'project', 'order_cost', 'note']
with open("result.csv", "w", encoding="utf-8-sig", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(header)
    writer.writerows(data) 
```
- 中文乱码使用：==encoding="utf-8-sig"==
:::



::: warning issues
- stale element reference: element is not attached to the page document
    - 刷新页面后，之前定位元素不再可引用
- 嵌套在iframe中的元素定位
    - ==switch_to.frame/alert/default_content==
        - alert ——返回浏览器的Alert对象，可对浏览器alert、confirm、prompt框操作;
        - frame(frame_element) ——切到某个frame
        - parent_frame() ——切到父frame
        - default_content() ——切到主文档
- css 选取包含某元素的标签🐑: =='.con2 table[background]'==
- ElementNotInteractableException (headless 模式下有时出现)
    - 解决方式，调用ActionChains()，模拟人类操作过程，先定位的元素
    - 将鼠标放在该元素上，重新定位元素，并点击元素
    ```py
    from selenium.webdriver.common.action_chains import ActionChains
    my_actionChain = ActionChains(driver)
    my_actionChain.move_to_element(more_arrow).perform()
    driver.click("css", "xxx")
    ```
:::
