
严格说来，Selenium是一套完整的Web应用程序测试系统，它包含了测试的录制（Selenium IDE）、编写及运行（Selenium Remote Control）和测试的并行处理（Selenium Grid）。Selenium的核心Selenium Core基于JsUnit，完全由JavaScript编写，因此可运行于任何支持JavaScript的浏览器上。与WatiN相同，Selenium也是一款同样使用Apache License 2.0协议发布的开源框架

打印当前url:


	import time
	from selenium import webdriver

	driver = webdriver.Firefox()
	url = "http://www.baidu.com"
	print ("now access %s" %(url))
	driver.get(url)
	driver.find_element_by_xpath(".//*[@id='kw']").send_keys("zhaoshanshan")
	driver.find_element_by_xpath(".//*[@id='su']").submit()
	print (driver.current_url)




浏览器前进、后退

	# -*- coding:utf-8 -*-

	import time
	from selenium import webdriver

	browser = webdriver.Firefox()
	first_url = "http://www.baidu.com"
	print ("first url is %s"%(first_url))
	browser.get(first_url)
	time.sleep(2)

	second_url = "http://news.baidu.com"
	print ("second url is %s" %(second_url))
	browser.get(second_url)
	time.sleep(2)

	#浏览器后退
	browser.back()
	time.sleep(2)

	#浏览器前进
	browser.forward()
	time.sleep(2)s



还有一个问题，有时候我们并不想勾选页面的所有的复选框（checkbox），可以通过下面办法把最后一个被勾选的框去掉。如下：

其中pop方法介绍：
python list.pop() 方法: 从列表中移除并返回最后一个对象或者obj， 返回从列表中删除的对象 

	# -*- coding:utf-8 -*-

	import time
	from selenium import webdriver
	import os

	driver = webdriver.Firefox()
	driver.get('file:///'+os.path.abspath("checkbox.html"))
	checkboxes = driver.find_elements_by_css_selector('input[type=checkbox]')
	for checkbox in checkboxes:
    	checkbox.click()
    	time.sleep(2)

	driver.find_elements_by_css_selector("input[type=checkbox]").pop().click()
	time.sleep(2)
	driver.quit()
