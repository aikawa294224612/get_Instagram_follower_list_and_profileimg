# get Instagram Follower List & Profile Image

Instagram爬蟲已經禁止直接抓取(會擋需要登入才可以)，需要透過selenium fake 人類操作行為，
透過selenium去爬出個人的follower或following list

出現問題:
網路上目前提供的tutorial主要抓取英文語系頁面，且instagram登入後還會跳到其他頁面(儲存你的登入資料？)，所以就tutorial無法適用

### Install package


```python
!pip install explicit
```

    Collecting explicit
      Downloading https://files.pythonhosted.org/packages/d1/47/b586118021544b0e716fd14fe81444c4398212a9ed0f42819e1c78955c8d/explicit-0.1.3-py2.py3-none-any.whl
    Collecting selenium (from explicit)
      Downloading https://files.pythonhosted.org/packages/80/d6/4294f0b4bce4de0abf13e17190289f9d0613b0a44e5dd6a7f5ca98459853/selenium-3.141.0-py2.py3-none-any.whl (904kB)
    Requirement already satisfied: six in c:\users\aikawa\anaconda3\lib\site-packages (from explicit) (1.12.0)
    Collecting pbr>=2.0 (from explicit)
      Downloading https://files.pythonhosted.org/packages/fb/48/69046506f6ac61c1eaa9a0d42d22d54673b69e176d30ca98e3f61513e980/pbr-5.5.1-py2.py3-none-any.whl (106kB)
    Requirement already satisfied: urllib3 in c:\users\aikawa\anaconda3\lib\site-packages (from selenium->explicit) (1.25.9)
    Installing collected packages: selenium, pbr, explicit
    Successfully installed explicit-0.1.3 pbr-5.5.1 selenium-3.141.0
    


```python
!pip install selenium
```

    Requirement already satisfied: selenium in c:\users\aikawa\anaconda3\lib\site-packages (3.141.0)
    Requirement already satisfied: urllib3 in c:\users\aikawa\anaconda3\lib\site-packages (from selenium) (1.25.9)
    


```python
!pip install openpyxl
```

    Requirement already satisfied: openpyxl in c:\users\aikawa\anaconda3\lib\site-packages (3.0.0)
    Requirement already satisfied: jdcal in c:\users\aikawa\anaconda3\lib\site-packages (from openpyxl) (1.4.1)
    Requirement already satisfied: et-xmlfile in c:\users\aikawa\anaconda3\lib\site-packages (from openpyxl) (1.0.1)
    

### Import


```python
import requests
from bs4 import BeautifulSoup
import os.path
import itertools

from explicit import waiter, XPATH
from selenium import webdriver
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.common.by import By
from time import sleep

from openpyxl import Workbook, load_workbook
import unittest
```


```python
# https://stackoverflow.com/questions/37233803/how-to-web-scrape-followers-from-instagram-web-browser
def login(driver):
    username = ""  # <username here>
    password = ""  # <password here>

    # Load page
    driver.get("https://www.instagram.com/accounts/login/")
    sleep(3)
    
    # Login
    driver.find_element_by_name("username").send_keys(username)
    driver.find_element_by_name("password").send_keys(password)
    submit = driver.find_element_by_tag_name('form')
    submit.submit()

    # Wait for the user dashboard page to load
    # WebDriverWait(driver, 15).until(
    #     EC.presence_of_element_located((By.LINK_TEXT, "See All")))
    
    sleep(3)

    driver.get("https://www.instagram.com/"+username+"/")
    

def scrape_followers(driver, account):
    # Load account page
    driver.get("https://www.instagram.com/{0}/".format(account))

    # Click the '追蹤中'/ '追蹤者' link
    sleep(2)
    driver.find_element_by_partial_link_text("追蹤者").click()  # 抓follower(追蹤者) or following(追蹤中)

    # Wait for the followers modal to load
    waiter.find_element(driver, "//div[@role='dialog']", by=XPATH)
    allfoll = int(driver.find_element_by_xpath("//li[2]/a/span").text)

    follower_css = "ul div li:nth-child({}) a.notranslate"  # Taking advange of CSS's nth-child functionality
    for group in itertools.count(start=1, step=12):
        for follower_index in range(group, group + 12):
            if follower_index > allfoll:
                raise StopIteration
            yield waiter.find_element(driver, follower_css.format(follower_index)).text
 
        last_follower = waiter.find_element(driver, follower_css.format(group+11))
        driver.execute_script("arguments[0].scrollIntoView();", last_follower)
    
        
if __name__ == "__main__":
    account = ""  # <account to check>
    driver = webdriver.Chrome()
    
    wb = Workbook()
    ws = wb.active
        
    try:
        login(driver)
        print('Followers of the "{}" account'.format(account))
        for count, follower in enumerate(scrape_followers(driver, account=account), 1):
            print("\t{}".format(follower))
            ws.append([follower])
    finally:
        driver.quit()   
        wb.save('instagran_id_'+account+'.xlsx')
```

    Followers of the "" account
    	a22177861
    	mina____photo
    	iverson09123
    	...
    ---------------------------------------------------------------------------

    StopIteration                             Traceback (most recent call last)

    <ipython-input-10-7227b6e4a2b0> in scrape_followers(driver, account)
         40             if follower_index > allfoll:
    ---> 41                 raise StopIteration
         42             yield waiter.find_element(driver, follower_css.format(follower_index)).text
    

    StopIteration: 

    
    The above exception was the direct cause of the following exception:
    

    RuntimeError                              Traceback (most recent call last)

    <ipython-input-10-7227b6e4a2b0> in <module>
         56         login(driver)
         57         print('Followers of the "{}" account'.format(account))
    ---> 58         for count, follower in enumerate(scrape_followers(driver, account=account), 1):
         59             print("\t{}".format(follower))
         60             ws.append([follower])
    

    RuntimeError: generator raised StopIteration



```python
def scrape_image(driver, image_id, account):
    isfamous = 0
    # Load account page
    driver.get("https://www.instagram.com/{0}/".format(account))

    sleep(2)
    
    # https://stackoverflow.com/questions/30002313/selenium-finding-elements-by-class-name-in-python
    #List<WebElement> deleteLinks = driver.find_elements(By.CLASS_NAME, "be6sR")
    
    # 所帳號的class不同
    if len(driver.find_elements(By.CLASS_NAME, "be6sR")) == 0:
        img_src = driver.find_element(By.CLASS_NAME, "_6q-tv").get_attribute("src")
    else:
        img_src = driver.find_element(By.CLASS_NAME, "be6sR").get_attribute("src")
        
    followertext = driver.find_elements(By.CLASS_NAME, "g47SY")[1].text
    
    if followertext.isnumeric() and int(followertext) == 1:
        famous = 1
    elif not followertext.isnumeric() and followertext == 'NaN':
        famous = 0
    else:
        famous = int(driver.find_elements(By.CLASS_NAME, "g47SY")[1].get_attribute("title").replace(',',''))
            
    
    print(famous)
    
    if famous > 1000:
        isfamous = 1
    
    return img_src + str(isfamous)
```


```python
if __name__ == "__main__":
    driver = webdriver.Chrome()
    wr = load_workbook('instagran_id.xlsx')
    sheet = wr.active 
    
    wb = Workbook()
    ws = wb.active
        
    try:
        login(driver) 
        
        # https://blog.techbridge.cc/2018/10/05/how-to-use-python-manipulate-excel-spreadsheet/
        image_id = 1
        for column in sheet.columns:
            for cell in column:
                res = scrape_image(driver, image_id, account=cell.value)
                
                print(res[:-1])
                
                r = requests.get(res[:-1])
                file = open("img/" + str(image_id) +".png", "wb")
                file.write(r.content)
                file.close()
                
                ws.append([image_id, res[-1]])
                
                image_id += 1
    finally:
        driver.quit()  
        wb.save('data.xlsx')
```

    96
    https://instagram.ftpe8-2.fna.XXblablabla...
    ... 

```python

```
